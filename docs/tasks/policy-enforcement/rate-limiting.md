本任务将演示如何使用Istio实现动态流控。

## 开始之前

* 根据快速入门指南 [Installation guide]({{home}}/docs/setup/kubernetes/quick-start.html) 在Kubernetes集群中安装Istio。

* 部署 [BookInfo]({{home}}/docs/guides/bookinfo.html) 示例应用。

* 初始化应用版本，把直接来自测试用户“Jason”的`reviews`服务请求路由到v2版本，同时把来自其他用户的请求路由到v3版本。

  ```bash
  istioctl create -f samples/bookinfo/kube/route-rule-reviews-test-v2.yaml
  istioctl create -f samples/bookinfo/kube/route-rule-reviews-v3.yaml
  ```
  
  > 注意：如果与之前的任务中设置了重复的规则，请使用`istioctl replace`替代`istioctl create`。

## 限流 

Istio支持用户为一个服务设置流量速率限制。
 
可以把`ratings`当作一个外部付费服务，像Rotten Tomatoes®之类的，限额`1qps`免费。使用Istio我们能保证不会超过`1qps`。

1. 在浏览器中打开BookInfo的`productpage`（http://$GATEWAY_URL/productpage）。

   如果使用用户"jason"登录，应当会看见每个审核的空白评级星号，表明`ratings`服务是通过`reviews`服务的"v2"版本调用的。
   
   如果使用其他用户登录（或退出），应当会看见每个审核的红色评价星号，表明`ratings`服务是通过`reviews`服务的"v3"版本调用的。

1. 配置一个速率限制的`memquota`适配器。
   
   将下列YAML片段保存到文件`ratelimit-handler.yaml`中：

   ```yaml
   apiVersion: config.istio.io/v1alpha2
   kind: memquota
   metadata:
     name: handler
     namespace: istio-system
   spec:
     quotas:
     - name: requestcount.quota.istio-system
       # default rate limit is 5000qps
       maxAmount: 5000
       validDuration: 1s
       # The first matching override is applied.
       # A requestcount instance is checked against override dimensions.
       overrides:
       # The following override applies to traffic from 'rewiews' version v2,
       # destined for the ratings service. The destinationVersion dimension is ignored.
       - dimensions:
           destination: ratings
           source: reviews
           sourceVersion: v2
         maxAmount: 1
         validDuration: 1s
   ```

   然后执行命令：

   ```bash
   istioctl create -f ratelimit-handler.yaml
   ```
 
   这个配置指定了一个默认5000 qps的速率限制。通过review-v2调用评级服务的流量，受到 1qps的速率限制。在本示例中，用户"jason"被路由到reviews-v2，因此受到1qps的速率限制。
 
1. 配置速率限制实例和规则 

   创建一个限额实例，命名为`requestcount`，映射进入属性到限额规则，并创建一个规则用于memquota处理器。

   ```yaml
   apiVersion: config.istio.io/v1alpha2
   kind: quota
   metadata:
     name: requestcount
     namespace: istio-system
   spec:
     dimensions:
       source: source.labels["app"] | source.service | "unknown"
       sourceVersion: source.labels["version"] | "unknown"
       destination: destination.labels["app"] | destination.service | "unknown"
       destinationVersion: destination.labels["version"] | "unknown"
   ---
   apiVersion: config.istio.io/v1alpha2
   kind: rule
   metadata:
     name: quota
     namespace: istio-system
   spec:
     actions:
     - handler: handler.memquota
       instances:
       - requestcount.quota
   ```

   保存此配置到`ratelimit-rule.yaml`并执行下述命令：

   ```bash
   istioctl create -f ratelimit-rule.yaml
   ```

1. 使用下列命令为`productpage`页生成负载：

   ```bash
   while true; do curl -s -o /dev/null http://$GATEWAY_URL/productpage; done
   ```

1. 在浏览器中刷新`productpage`页。

   如果在负载生成器运行时（即生成大于1 req/s）使用用户"jason"登录，浏览器所生成的流量将被限流在1 qps。
   review-v2服务不能访问评级服务，将不再看到星号。对所有其他用户来说默认生效的是5000qps的限流额度，将会持续看到红色星号。

## 带条件的速率限制

前述例子中应用了一个速率限制策略到`ratings`服务上，但没有考虑非规格属性。通过在限额规则中使用匹配条件，就可以有条件地应用速率限制到任意属性上。

举个例子，考虑如下配置：

   ```yaml
   apiVersion: config.istio.io/v1alpha2
   kind: rule
   metadata:
     name: quota
     namespace: istio-system
   spec:
     match: source.namespace != destination.namespace
     actions:
     - handler: handler.memquota
       instances:
       - requestcount.quota

   ```

这个配置将应用限额规则到那些源命名空间和目的命名空间不同的请求上。

## 理解限流

之前的例子中我们看到了Mixer如何应用速率限制到满足特定条件的请求上。

每个指定的限额实例，比如`requestcount`，代表着一个计数器集合。该集合由所有限额规则的笛卡尔积来定义。如果最后一个`expiration`周期中的请求数超过了`maxAmount`，Mixer就返回一个`RESOURCE_EXHAUSTED`消息给代理，代理就依次返回`HTTP 429`状态给调用方。

`memquota`适配器使用一个亚秒级分辨率的滑动窗口来强行限流。

适配器配置中的`maxAmount`为一个限额实例的所有关联计数器，设置了默认的限制。如果没有任何限额重载匹配请求，则这个默认限制会生效。Memquota选择匹配请求的第一个重载。一个重载无需指定所有限额规则。在ratelimit-handler.yaml示例中，`1qps`重载被只匹配四分之三的限额规则所选中。

如果想在指定命名空间强制执行以上策略，而在不是整个Istio网格，那么可以使用指定的命名空间来替代istio系统的所有事件。

## 取消流控

* 取消限流配置：

  ```bash
  istioctl delete -f ratelimit-handler.yaml
  istioctl delete -f ratelimit-rule.yaml
  ```

* 取消应用路由规则：

  ```
  istioctl delete -f samples/bookinfo/kube/route-rule-reviews-test-v2.yaml
  istioctl delete -f samples/bookinfo/kube/route-rule-reviews-v3.yaml
  ```

* 如果并不计划了解更多后续任务，参考[BookInfo cleanup]({{home}}/docs/guides/bookinfo.html#cleanup)的说明来关闭应用。

## 进阶阅读

* 了解更多关于[Mixer]({{home}}/docs/concepts/policy-and-control/mixer.html)和[Mixer Config]({{home}}/docs/concepts/policy-and-control/mixer-config.html)的内容。

* 探索完整的[Attribute Vocabulary]({{home}}/docs/reference/config/mixer/attribute-vocabulary.html)。

* 阅读[Writing Config]({{home}}/docs/reference/writing-config.html)的参考指南。
