# 设置请求超时

这一任务用于展示如何使用 Istio 在 Envoy 中设置请求超时。

## 开始之前

- 依据 [安装指南](../../../setup/) 的指导安装 Istio。
- 部署 [BookInfo](../../..//guides/bookinfo.html) 示例程序。
- 执行如下命令，初始化基于应用版本的路由。

`istioctl create -f samples/bookinfo/kube/route-rule-all-v1.yaml`

> 注意：本任务假设读者在 Kubernetes 上部署这一应用。所有涉及的命令都使用的是 Kubernetes 版本
> 的 yaml 文件（例如`samples/bookinfo/kube/route-rule-all-v1.yaml`）。如果要在不同环境
>运行这一任务，则需要把路径中的`kube`替换成对应的环境（例如
>`samples/bookinfo/consul/route-rule-all-v1.yaml`）。

## 请求超时

http 的请求超时可以在路由规则中的`httpReqTimeout`字段来设置。缺省的超时时间是 15 秒，在下面我们会把`reviews`服务的超时时间设置为一秒钟。为了展示效果，我们还需要在调用`ratings`服务的时候加入两秒钟的延迟。

1.把`reviews`服务的请求路由到 v2，v2 版本的`reviews`服务会调用`ratings`服务

~~~bash
cat <<EOF | istioctl replace -f -
apiVersion: config.istio.io/v1alpha2
kind: RouteRule
metadata:
  name: reviews-default
spec:
  destination:
    name: reviews
  route:
  - labels:
      version: v2
EOF
~~~

2.给`ratings`服务加入两秒钟的延迟

~~~bash
cat <<EOF | istioctl replace -f -
apiVersion: config.istio.io/v1alpha2
kind: RouteRule
metadata:
  name: ratings-default
spec:
  destination:
    name: ratings
  route:
  - labels:
      version: v1
  httpFault:
    delay:
      percent: 100
      fixedDelay: 2s
EOF
~~~

3.在浏览器中打开 BookInfo 的网址：`http://$GATEWAY_URL/productpage`。

你会看到 BookInfo 应用在正常运行（并且显示了评级的星星），但是你也会注意到，每次刷新页面的时候会有两秒钟的延迟。

4.接下来我们给对`reviews`的调用加入一秒钟的超时限制

~~~bash
cat <<EOF | istioctl replace -f -
apiVersion: config.istio.io/v1alpha2
kind: RouteRule
metadata:
  name: reviews-default
spec:
  destination:
    name: reviews
  route:
  - labels:
      version: v2
  httpReqTimeout:
    simpleTimeout:
      timeout: 1s
EOF
~~~

5.刷新 BookInfo 页面

现在会看到，只有一秒钟就刷新完成了（而不是之前的两秒钟），但是 Reviews 部分不再可用。

## 幕后故事

这一任务中使用 Istio 为`reviews`服务设置了一秒钟的请求超时（而不是缺省的 15 秒）。由于`reviews`服务在处理请求的时候会调用下游的`ratings`服务，而`ratings`服务已经被我们利用 Istio 注入了两秒钟的延迟，这一变动会让`reviews`服务的处理时间超过一秒，故而引发了超时。

你会看到 BookInfo productpage（需要调用`reviews`服务来生成页面）不会显示 review 信息，而是显示为：“Sorry, product reviews are currently unavailable for this book. ”，这是`reviews`服务的超时引发的错误造成的结果。

如果检查一下[错误注入任务](../../../tasks/traffic-management/fault-injection.html)，会发现`productpage`服务在调用`reviews`的时候，也有自己的应用级别的超时（三秒钟）。我们前面的测试中使用了一秒钟的超时，如果我们把下游服务的超时时间设置为大于三秒钟的值（例如四秒），由于三秒钟的限制更为严格，会被优先处理，因此过大的超时时间就不会生效了。参看[错误处理](../../../concepts/traffic-management/handling-failures.html#faq)一节会有更详细的信息。

还需要注意的就是，除了在路由规则中设置超时之外，还可以在请求中加入“x-envoy-upstream-rq-timeout-ms”头来设置超时，在这一设置中的时间单位是毫秒而不是秒。

## 清理

- 删除路由规则。

·`istioctl delete -f samples/bookinfo/kube/route-rule-all-v1.yaml`

- 如果没有计划进一步运行下面的任务，可以参照[BookInfo 清理](../../../samples/bookinfo.html#cleanup) 中的介绍来关闭这一应用。

## 延伸阅读

- [错误处理](../../../concepts/traffic-management/handling-failures.html)
- [路由规则](../../../concepts/traffic-management/handling-failures.html)
