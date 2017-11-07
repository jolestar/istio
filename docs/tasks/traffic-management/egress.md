# 控制Egress流量

缺省情况下，启用了Istio的服务是无法访问外部URL的，这是因为Pod中的iptables把所有外发传输都转向到了Sidecar代理，而这一代理只处理集群内的访问目标。

本节内容会描述如何把外部服务提供给启用了Istio的客户端服务使用，你会学到如何使用Egress规则访问外部服务，或者如何简单的让特定IP范围穿透Istio代理。

## 开始之前

- 遵循[安装指南](../../setup)设置Istio
- 启动[sleep](https://github.com/istio/istio/tree/master/samples/sleep)示例，用于测试外部访问。

    ```bash
    kubectl apply -f <(istioctl kube-inject -f samples/sleep/sleep.yaml)
    ```

	注意所有可以在其中执行`exec`和`curl`的pod都可以这样做。

## 使用Istio的Egress规则

使用Egress规则，就可以从Istio集群内访问任何公开服务。本任务中我们会使用[httpbin.org](http://httpbin.org/)以及[www.google.com](www.google.com)作为范例。

### 配置外部服务

1. 创建一个Egress规则，来允许访问外部HTTP服务：

    ```bash
    cat <<EOF | istioctl create -f -
    apiVersion: config.istio.io/v1alpha2
    kind: EgressRule
    metadata:
      name: httpbin-egress-rule
    spec:
      destination:
        service: httpbin.org
      ports:
        - port: 80
          protocol: http
    EOF
    ```

1. 创建一个Egress规则，容许访问外部HTTPS服务：

    ```bash
    cat <<EOF | istioctl create -f -
    apiVersion: config.istio.io/v1alpha2
    kind: EgressRule
    metadata:
     name: google-egress-rule
    spec:
     destination:
       service: www.google.com
     ports:
       - port: 443
         protocol: https
    EOF
    ```

### 发出对外请求

1. 使用`exec`进入作为测试源使用的pod。例如，如果你正在使用sleep服务，运行下列命令：

    ```bash
    export SOURCE_POD=$(kubectl get pod -l app=sleep -o jsonpath={.items..metadata.name})
    kubectl exec -it $SOURCE_POD -c sleep bash
    ```

2. 发出请求到外部HTTP服务：

    ```bash
    curl http://httpbin.org/headers
    ```

3. 发送请求到外部HTTPS服务

	外部的HTTPS服务必须可以通过HTTP访问，在请求中指定端口：

    ```bash
    curl http://www.google.com:443
    ```

### 为外部服务设置路由规则

和集群内请求类似，Istio的[路由规则](../../concepts/traffic-management/rules-configuration.md)也可以用于设置使用了Egress规则的外部服务。为了进行演示，我们为httpbin.org服务设置一个超时。

1. 在作为测试源使用的Pod内，调用外部服务httpbin.org的`/delay`端点：

    ```bash
    kubectl exec -it $SOURCE_POD -c sleep bash
    time curl -o /dev/null -s -w "%{http_code}\n" http://httpbin.org/delay/5
    ```

    ```bash
    200

    real    0m5.024s
    user    0m0.003s
    sys     0m0.003s
    ```

	请求应该在大约五秒钟之后返回200(OK)。

2. 退出源Pod，再使用`istioctl`来给外部服务httpbin.org设置3秒钟的调用超时：

    ```bash
    cat <<EOF | istioctl create -f -
    apiVersion: config.istio.io/v1alpha2
    kind: RouteRule
    metadata:
     name: httpbin-timeout-rule
    spec:
     destination:
       service: httpbin.org
     http_req_timeout:
       simple_timeout:
         timeout: 3s
    EOF
    ```

3. 等待几秒钟之后，再次发起_curl_调用：

    ```bash
    kubectl exec -it $SOURCE_POD -c sleep bash
    time curl -o /dev/null -s -w "%{http_code}\n" http://httpbin.org/delay/5
    ```

    ```bash
    504

    real    0m3.149s
    user    0m0.004s
    sys     0m0.004s
    ```

	这次在大概3秒钟之后返回504（网关超时）。虽然httpbin.org在等待5秒钟，Istio在3秒钟时就切断了请求。

### 直接调用外部服务

目前Istio的Egress规则只提供对HTTP/HTTPS请求的支持。如果想要访问其他协议的外部服务（例如mongodb://host/database），或者让指定IP范围直接穿透Istio，就需对源服务的Envoy Sidecar进行配置，阻止其对外部请求的[拦截](../../concepts/traffic-management/request-routing.md#communication-between-services)。可以在使用[istio kube-inject](../../reference/commands/istioctl.md#istioctl-kube-inject)时启用`--includeIPRanges`参数来满足这一要求。

最简单的`--includeIPRanges`用法就是提交所有的内部服务的IP范围，然后让所有外部服务地址都绕过Sidecar代理。内部IP范围跟你所运行的集群有关，例如Minikube中的范围是`10.0.0.1/24`，所以应该这样启动sleep服务：

```bash
kubectl apply -f <(istioctl kube-inject -f samples/sleep/sleep.yaml --includeIPRanges=10.0.0.1/24)
```

在IBM Bluemix中使用：

```bash
kubectl apply -f <(istioctl kube-inject -f samples/sleep/sleep.yaml --includeIPRanges=172.30.0.0/16,172.20.0.0/16,10.10.10.0/24)
```

在Google容器引擎（GKE）上的IP范围不是固定的，所以需要运行命令`gcloud container clusters describe`来获取要使用的IP范围。例如：

```bash
gcloud container clusters describe XXXXXXX --zone=XXXXXX | grep -e clusterIpv4Cidr -e servicesIpv4Cidr
```
```
clusterIpv4Cidr: 10.4.0.0/14
servicesIpv4Cidr: 10.7.240.0/20
```
```bash
kubectl apply -f <(istioctl kube-inject -f samples/sleep/sleep.yaml --includeIPRanges=10.4.0.0/14,10.7.240.0/20)
```

而在Azure容器服务（ACS）上就应该执行：

```bash
kubectl apply -f <(istioctl kube-inject -f samples/sleep/sleep.yaml --includeIPRanges=10.244.0.0/16,10.240.0.0/16)
```

以这种方式启动服务之后，Istio Sidecar就只会处理和管理集群内的请求，所有的外部请求就会简单的绕开Sidecar直接访问目标了：

```bash
export SOURCE_POD=$(kubectl get pod -l app=sleep -o jsonpath={.items..metadata.name})
kubectl exec -it $SOURCE_POD -c sleep curl http://httpbin.org/headers
```

## 幕后故事

这个任务中我们在Istio集群中使用两种方式来访问外部服务：

1. 使用Egress规则（推荐）。
2. 配置Istio Sidecar，在他的iptables中排除对外部IP的控制。

第一种方式目前只支持HTTP(S)请求，但是可以为外部服务提供和内部服务一样的路由支持。我们使用了一个路由规则来展示了这一能力。

第二种方式绕过了Sidecar代理，让服务可以直接访问任何外部资源，只是这样的设置需要一点对集群配置的了解。

## 清理

1. 删除规则：

    ```bash
    istioctl delete egressrule httpbin-egress-rule google-egress-rule
    istioctl delete routerule httpbin-timeout-rule
    ```

2. 关闭[sleep](https://github.com/istio/istio/tree/master/samples/sleep)服务：

    ```bash
    kubectl delete -f samples/sleep/sleep.yaml
    ```

## 延伸阅读

- 关于[Egress规则](../../concepts/traffic-management/rules-configuration.md#egress-rules)的更多内容。
- 如何为Egress通信设置[超时](../../reference/config/traffic-rules/routing-rules.md#httptimeout)、[重试](../../reference/config/traffic-rules/routing-rules.md#httpretry)以及[断路器](../../reference/config/traffic-rules/destination-policies.md#circuitbreaker)。
