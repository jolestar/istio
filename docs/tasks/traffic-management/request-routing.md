---
title: Configuring Request Routing
overview: This task shows you how to configure dynamic request routing based on weights and HTTP headers.

order: 10

layout: docs
type: markdown
---
此任务将为您展示如何根据权重和HTTP header配置动态请求路由。

## 前提条件

* 参照该文档中的步骤安装Istio [Installation guide](../docs/setup/)。

* 部署 [BookInfo](../docs/guides/bookinfo.html) 示例应用程序。

> 请注意：本文档假设示例应用程序通过kubernetes进行部署。所有的示例命令行都采用规则yaml文件指定的kubernetes版本（例如`samples/bookinfo/kube/route-rule-all-v1.yaml`)。如果您在不同的环境下运行本任务，请将`kube`修改为您运行环境中相应的目录（例如，对基于Consul的运行环境，目录就是`samples/bookinfo/consul/route-rule-all-v1.yaml`）。

## 基于内容的路由

BookInfo示例部署了三个版本的reviews服务，因此我们需要设置一个缺省路由。否则当你多次访问该应用程序时，你会发现有时输出会包含带星级的评价内容，有时又没有。出现该现象的原因是当没有为应用显示指定缺省路由时，Istio会将请求随机路由到所有版本的可用服务上。

> 请注意：本文档假设您还没有设置任何路由规则。如果您已经为示例应用程序创建了存在冲突的路由规则，您将需要在下面的命令中使用 `replace` 关键字代替 `create`。

1. 将所有微服务的缺省版本设置为v1。

   ```bash
   istioctl create -f samples/bookinfo/kube/route-rule-all-v1.yaml
   ```

   > 请注意：在kubernetes中部署Istio时，您可以在上面及其它所有命令行中用 `kubectl` 代替 `istioctl`。但是目前 `kubectl` 不提供对命令输入参数的验证。

   您可以通过下面的命令来显示所有以创建的路由规则。

   ```bash
   istioctl get routerules -o yaml
   ```
   ```yaml
   apiVersion: config.istio.io/v1alpha2
   kind: RouteRule
   metadata:
     name: details-default
     namespace: default
     ...
   spec:
     destination:
       name: details
     precedence: 1
     route:
     - labels:
         version: v1
   ---
   apiVersion: config.istio.io/v1alpha2
   kind: RouteRule
   metadata:
     name: productpage-default
     namespace: default
     ...
   spec:
     destination:
       name: productpage
     precedence: 1
     route:
     - labels:
         version: v1
   ---
   apiVersion: config.istio.io/v1alpha2
   kind: RouteRule
   metadata:
     name: ratings-default
     namespace: default
     ...
   spec:
     destination:
       name: ratings
     precedence: 1
     route:
     - labels:
         version: v1
   ---
   apiVersion: config.istio.io/v1alpha2
   kind: RouteRule
   metadata:
     name: reviews-default
     namespace: default
     ...
   spec:
     destination:
       name: reviews
     precedence: 1
     route:
     - labels:
         version: v1
   ---
   ```

   由于路由规则是通过异步方式进行分发到代理的，过几秒钟后规则才会同步到所有pod上。因此您需要等几秒钟再尝试访问应用。


1. 在浏览器中打开BookInfo应用程序的 URL (http://$GATEWAY_URL/productpage)。

   您应该可以看到BookInfo应用程序的productpage页面.
   请注意`productpage`页面显示的内容中不包含带星的评级信息，这是因为`reviews:v1`服务不会访问`ratings`服务。

1. 将来自某个特定用户的请求路由到`reviews:v2`。

   把来自测试用户"jason"的请求路由到`reviews:v2`，以启动评级服务。

   ```bash
   istioctl create -f samples/bookinfo/kube/route-rule-reviews-test-v2.yaml
   ```

   查看刚才创建的路由规则：

   ```bash
   istioctl get routerule reviews-test-v2 -o yaml
   ```
   ```yaml
   apiVersion: config.istio.io/v1alpha2
   kind: RouteRule
   metadata:
     name: reviews-test-v2
     namespace: default
     ...
   spec:
     destination:
       name: reviews
     match:
       request:
         headers:
           cookie:
             regex: ^(.*?;)?(user=jason)(;.*)?$
     precedence: 2
     route:
     - labels:
         version: v2
   ```

1. 以"jason"用户登录`productpage`页面。

   此时您应该可以在每条评价后面看到星级信息。请注意如果您以别的用户登录，您还是只能看到`reviews:v1`版本服务呈现出的内容，即不包含星级信息的内容。

## 理解规则背后的内容

在这个任务中，您首先使用Istio将100%的请求流量都路由到了v1版本的BookInfo服务。
然后再设置了一条路由规则，该路由规则基于请求的header（例如一个用户cookie）选择性地将特定的流量路由到了v2版本的reviews服务。

一旦v2版本的reviews服务经过测试后满足要求，我们就可以使用Istio将来自所有用户的流量一次性或者渐进地路由到v2版本。我们将在另一个单独的任务中对此进行尝试。

## 清理

* 删除路由规则。

  ```bash
  istioctl delete -f samples/bookinfo/kube/route-rule-all-v1.yaml
  istioctl delete -f samples/bookinfo/kube/route-rule-reviews-test-v2.yaml
  ```

* 如果您不打算尝试后面的任务，请参照
  [BookInfo cleanup](../docs/guides/bookinfo.html#cleanup) 中的步骤关闭应用程序。
 
## 下一步

* 更多的内容请参见 [request routing](../docs/concepts/traffic-management/rules-configuration.html).   