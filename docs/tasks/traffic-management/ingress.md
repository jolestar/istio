# Istio Ingress控制器

此任务将演示如何通过配置Istio将服务公开到service mesh集群外部。在Kubernetes环境中，[Kubernetes Ingress Resources](https://kubernetes.io/docs/concepts/services-networking/ingress/) 允许用户指定某个服务是否要公开到集群外部。然而，Ingress Resource规范非常精简，只允许用户设置主机，路径，以及后端服务。为了利用Istio的高级路由能力，我们建议组合使用 Ingress Resource和Istio的路由规则。

> 注意: Istio 不支持在ingress resource规范中使用`ingress.kubernetes.io` 注解（annotations）。除了`kubernetes.io/ingress.class: istio`之外的注解都会被忽略。

## 前提条件

* 参照文档[安装指南](../../setup/index.md)中的步骤安装Istio。

* 确保当前的目录是`istio`目录。

* 启动 [httpbin](https://github.com/istio/istio/tree/master/samples/httpbin) 示例, 我们会把这个服务作为目标（destination）服务公开到外部。

  如果你安装了 [Istio-Initializer](https://istio.io/docs/setup/kubernetes/sidecar-injection.html#automatic-sidecar-injection), 请执行：

  ```console
  kubectl apply -f samples/httpbin/httpbin.yaml
  ```

  如果没有 Istio-Initializer, 请执行：

  ```console
  kubectl apply -f <(istioctl kube-inject -f samples/httpbin/httpbin.yaml)
  ```

## 配置 ingress (HTTP)

1. 为 httpbin 服务创建一个基本的 Ingress Resource

   ```console
   cat <<EOF | kubectl create -f -
   apiVersion: extensions/v1beta1
   kind: Ingress
   metadata:
     name: simple-ingress
     annotations:
       kubernetes.io/ingress.class: istio
   spec:
     rules:
     - http:
         paths:
         - path: /.*
           backend:
             serviceName: httpbin
             servicePort: 8000
   EOF
   ```

   /.* 是一个 Istio 特殊表示法，以前缀 / 开始的配置，都表示前缀匹配。上面的配置允许访问 httpbin 服务中的所有 URI。然而，实际上我们只想开放一部分特殊的 URI。我们可以先定义一个默认的 *deny all* 路由规则，将所有的请求都拒绝：

   ```console
   cat <<EOF | istioctl create -f -
   ## Deny all access from istio-ingress
   apiVersion: config.istio.io/v1alpha2
   kind: RouteRule
   metadata:
     name: deny-route
   spec:
     destination:
       name: httpbin
     match:
       # Limit this rule to istio ingress pods only
       source:
         name: istio-ingress
         labels:
           istio: ingress
     precedence: 1
     route:
     - weight: 100
     httpFault:
       abort:
         percent: 100
         httpStatus: 403 #Forbidden for all URLs
   EOF
   ```

2. 然后，再定义一个高优先级的路由，允许访问  `/status/` 前缀的 URI。

   ```console
   cat <<EOF | istioctl create -f -
   ## Allow requests to /status prefix
   apiVersion: config.istio.io/v1alpha2
   kind: RouteRule
   metadata:
     name: status-route
   spec:
     destination:
       name: httpbin
     match:
       # Limit this rule to istio ingress pods only
       source:
         name: istio-ingress
         labels:
           istio: ingress
       request:
         headers:
           uri:
             prefix: /status
     precedence: 2 #must be higher precedence than the deny-route
     route:
     - weight: 100
   EOF
   ```

   这里也可以用其他的路由功能，比如重定向，重写，正则表达式匹配 HTTP headers，websocket upgrades，超时，重试等。详情请参看  [routing rules](https://istio.io/docs/reference/config/traffic-rules/routing-rules.html) 。

## 验证 ingress

1. 确定 ingress URL：

   * 如果你的集群运行的环境支持外部的负载均衡器，请使用 ingress 的外部地址：

     ```console
     kubectl get ingress simple-ingress -o wide
     ```

     ```console
     NAME             HOSTS     ADDRESS                 PORTS     AGE
     simple-ingress   *         130.211.10.121          80        1d
     ```

     ```console
     export INGRESS_HOST=130.211.10.121
     ```

   * 如果你的环境不支持负载均衡器，使用 ingress 控制器的 pod 的主机 IP：

     ```console
     kubectl get po -l istio=ingress -o jsonpath='{.items[0].status.hostIP}'
     ```

     ```console
     169.47.243.100
     ```

     同时也找到 istio-ingress 服务的 80 端口的 nodePort 映射端口:

     ```console
     kubectl get svc istio-ingress
     ```

     ```console
     NAME            CLUSTER-IP     EXTERNAL-IP   PORT(S)                      AGE
     istio-ingress   10.10.10.155   <pending>     80:31486/TCP,443:32254/TCP   32m
     ```

     ```console
     export INGRESS_HOST=169.47.243.100:31486
     ```

2. 使用 *curl* 访问 httpbin 服务：

   ```console
   curl -I http://$INGRESS_HOST/status/200
   ```

   ```console
   HTTP/1.1 200 OK
   Server: meinheld/0.6.1
   Date: Thu, 05 Oct 2017 21:23:17 GMT
   Content-Type: text/html; charset=utf-8
   Access-Control-Allow-Origin: *
   Access-Control-Allow-Credentials: true
   X-Powered-By: Flask
   X-Processed-Time: 0.00105214118958
   Content-Length: 0
   Via: 1.1 vegur
   Connection: Keep-Alive
   ```

3. 如果访问其他的没有明确公开的 URL，应该收到 HTTP 403 错误

   ```console
   curl -I http://$INGRESS_HOST/headers
   ```

   ```console
   HTTP/1.1 403 FORBIDDEN
   Server: meinheld/0.6.1
   Date: Thu, 05 Oct 2017 21:24:47 GMT
   Content-Type: text/html; charset=utf-8
   Access-Control-Allow-Origin: *
   Access-Control-Allow-Credentials: true
   X-Powered-By: Flask
   X-Processed-Time: 0.000759840011597
   Content-Length: 0
   Via: 1.1 vegur
   Connection: Keep-Alive
   ```

## 配置安全ingress(HTTPS)

1. 生成必要的[secret](https://kubernetes.io/docs/concepts/configuration/secret/)
   用[OpenSSL](https://www.openssl.org/)创建测试私钥和证书

   ```console
   openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout /tmp/tls.key -out /tmp/tls.crt -subj "/CN=foo.bar.com"
   ```

2. 用`kubectl`更新secret

   ```console
   kubectl create -n istio-system secret tls istio-ingress-certs --key /tmp/tls.key --cert /tmp/tls.crt
   ```

3.为 httpbin服务创建Ingress Resource

   ```console
   cat <<EOF | kubectl create -f -
   apiVersion: extensions/v1beta1
   kind: Ingress
   metadata:
     name: secure-ingress
     annotations:
       kubernetes.io/ingress.class: istio
   spec:
     tls:
       - secretName: istio-ingress-certs # currently ignored
     rules:
     - http:
         paths:
         - path: /.*
           backend:
             serviceName: httpbin
             servicePort: 8000
   EOF
   ```

   创建 `deny rule` ，同时为 `/status` 前缀创建前文相同的规则。同时，像前面一样，设置 INGRESS_HOST 指向 ingress 服务的 IP 和端口。

   > 注意: Envoy 当前只允许一个 TLS ingress 密钥，因为 [SNI](https://en.wikipedia.org/wiki/Server_Name_Indication) 尚未支持。也就是说看，ingress 中的 secretName 字段并没有用，secret 必须叫做 `istio-ingress-certs` 并且在 `istio-system` 命名空间（namespace）。

4. 用`curl`访问安全httpbin服务

   ```console
   curl -I -k https://$INGRESS_HOST/status/200
   ```

## 为gRPC配置ingress

Ingress控制器的`path`字段当前并不支持`.` 字符。这会导致使用命名空间的gRPC服务出问题（译注:因为gRPC用 `.` 分割命名空间和服务名）。为了解决这个问题，可以先把流量导向到一个虚拟（dummy）服务，然后设置路由规则去拦截流量并重定向到期望的服务。

1. 创建一个虚拟ingress服务：

   ```console
   cat <<EOF | kubectl create -f -
   apiVersion: v1
   kind: Service
   metadata:
     name: ingress-dummy-service
   spec:
     ports:
     - name: grpc
       port: 1337
   EOF
   ```

2. 创建一个拦截所有流量的ingress指向虚拟服务：

   ```console
   cat <<EOF | kubectl create -f -
   apiVersion: extensions/v1beta1
   kind: Ingress
   metadata:
     name: all-istio-ingress
     annotations:
       kubernetes.io/ingress.class: istio
   spec:
     rules:
     - http:
         paths:
         - backend:
             serviceName: ingress-dummy-service
             servicePort: grpc
   EOF
   ```

3. 给每个服务创建一个RouteRule ，将流量从虚拟服务重定向到正确的gRPC服务：

   ```console
   cat <<EOF | istioctl create -f -
   apiVersion: config.istio.io/v1alpha2
   kind: RouteRule
   metadata:
     name: foo-service-route
   spec:
     destination:
       name: ingress-dummy-service
     match:
       request:
         headers:
           uri:
             prefix: "/foo.FooService"
     precedence: 1
     route:
     - weight: 100
       destination:
         name: foo-service
   ---
   apiVersion: config.istio.io/v1alpha2
   kind: RouteRule
   metadata:
     name: bar-service-route
   spec:
     destination:
       name: ingress-dummy-service
     match:
       request:
         headers:
           uri:
             prefix: "/bar.BarService"
     precedence: 1
     route:
     - weight: 100
       destination:
         name: bar-service
   EOF
   ```

## 理解ingress原理

Ingress为外部流量进入Istio service mesh提供一个网关，并使Istio的流量管理和策略功能可用于边缘服务。

在前面的步骤中，我们在Istio service mesh中创建了一个服务，并展示了如何将服务的HTTP和HTTPS公开给外部流量，同时还展示了如何使用Istio路由规则来控制入口流量。

## 清理

1. 删除密钥，Ingress Resource定义以及Istio规则。

   ```console
   istioctl delete routerule deny-route status-route
   kubectl delete ingress simple-ingress secure-ingress
   kubectl delete -n istio-system secret istio-ingress-certs
   ```

2. 删除[httpbin](https://github.com/istio/istio/tree/master/samples/httpbin)服务。

   ```console
   kubectl delete -f samples/httpbin/httpbin.yaml
   ```

## 进阶阅读

* 进一步了解和学习[Ingress Resources](https://kubernetes.io/docs/concepts/services-networking/ingress/).
* 进一步了解和学习[routing rules](../../concepts/traffic-management/rules-configuration.md).