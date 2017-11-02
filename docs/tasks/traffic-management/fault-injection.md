本任务将演示如何注入延迟并测试应用弹性。

## 开始之前

* 参考文档[Installation guide]({{home}}/docs/setup/)中的步骤安装Istio。

* 部署[BookInfo]({{home}}/docs/guides/bookinfo.html)示例应用。

* 首先通过[request routing](./request-routing.html)任务，或通过执行下列命令，来初始化应用的版本路由信息：
  
  > 注意：这里假设尚未设置任何路由。如果已经为示例创建了存在冲突的路由规则，则需要在下列两条命令或其中之一使用`replace`代替`create`。
  
  ```bash
  istioctl create -f samples/bookinfo/kube/route-rule-all-v1.yaml
  istioctl create -f samples/bookinfo/kube/route-rule-reviews-test-v2.yaml
  ```

> 注意：本任务假设将通过Kubernetes来部署应用。所有的示例命令都采用规则yaml文件（如`samples/bookinfo/kube/route-rule-all-v1.yaml`）指定的Kubernetes版本。如果在不同的环境下运行此任务，请将`kube`修改为运行环境中（比如基于Consul运行环境就是`samples/bookinfo/consul/route-rule-all-v1.yaml`）相应的目录。

## 故障注入

为了测试BookInfo应用微服务的弹性，我们计划针对"jason"用户在reviews:v2和ratings服务之间 _注入7秒的延迟_ 。由于 _reviews:v2_ 服务针对调用ratings服务设置了10秒的超时，因此期望端到端的流程能无错持续。

1. 创建一个故障注入规则，来延迟来自用户"jason"（本次的测试用户）的流量：

   ```bash
   istioctl create -f samples/bookinfo/kube/route-rule-ratings-test-delay.yaml
   ```

   确认规则已创建：

   ```bash
   istioctl get routerule ratings-test-delay -o yaml
   ```
   ```yaml
   apiVersion: config.istio.io/v1alpha2
   kind: RouteRule
   metadata:
     name: ratings-test-delay
     namespace: default
     ...
   spec:
     destination:
       name: ratings
     httpFault:
       delay:
         fixedDelay: 7.000s
         percent: 100
     match:
       request:
         headers:
           cookie:
             regex: ^(.*?;)?(user=jason)(;.*)?$
     precedence: 2
     route:
     - labels:
         version: v1
   ```

   规则分发延迟到所有的pod需要数秒钟的时间。

1. 观察应用行为

   使用用户"jason"登录。如果应用的首页已设置正确地处理延迟，那首页将会在约7秒钟内加载完。为了解网页的响应时间，请打开IE、Chrome或Firefox（一般快捷键是_Ctrl+Shift+I_ 或 _Alt+Cmd+I_）的*Developer Tools*菜单，切换到Network标签，并reload `productpage`页面。

   将会看到网页在约6秒钟内加载完成。review部分将会显示 *Sorry, product reviews are currently unavailable for this book*。

## 理解原理

   整个review服务失败的原因是，BookInfo应用有一个bug。productpage和review服务之间的超时小于（3秒加上一次重试，总共6秒）review服务和rating服务之间的超时（10秒）。在由不同开发团队负责独立开发不同微服务的典型企业应用中，这类bug就会发生。Istio的故障注入规则有助于识别这些异常，而无需影响到最终用户。

   > 需要注意的是，这里只针对用户"jason"设置了故障影响。如果使用其他用户登录，则不会有任何延迟。

  **解决bug:** 至此，按理来说已经能解决这个问题了，通过增加productpage的超时，或者减少review服务到rating服务的超时，终止并重启相关的微服务，然后确认`productpage`不报错并正常返回结果即可。

  不过，在review服务的v3版本中已经解决了此问题，所以只要按照[traffic shifting]({{home}}/docs/tasks/traffic-management/traffic-shifting.html) 任务所述，转移所有流量到 `reviews:v3` 就能简单地修复问题。
  
  （这里为读者留一个练习——修改延迟规则，改为2.8秒的延迟，然后运行v3版本的review服务。）

## 清除

* 清除应用的路由规则：

  ```bash
  istioctl delete -f samples/bookinfo/kube/route-rule-all-v1.yaml
  istioctl delete -f samples/bookinfo/kube/route-rule-reviews-test-v2.yaml
  istioctl delete -f samples/bookinfo/kube/route-rule-ratings-test-delay.yaml
  ```

* 如果并不计划了解更多后续任务，参考[BookInfo cleanup]({{home}}/docs/guides/bookinfo.html#cleanup)的说明来关闭应用。

## 进阶阅读

* 了解更多关于[fault injection]({{home}}/docs/concepts/traffic-management/fault-injection.html)的内容。
