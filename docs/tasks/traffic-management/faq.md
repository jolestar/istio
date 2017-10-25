* _如何查看当前已经通过Istio配置好的规则？_

  使用`istioctl get routerules -o yaml` 或 `kubectl get routerules -o yaml`可以查看规则。

* _我创建了一个权重路由规则，想把流量分配到某个服务的两个版本上，但无法看到预期行为。_
  
  对于目前的Envoy sidecar实现，100次以上的请求才可能是能被观测到的理想分布。
  
* _创建路由规则后为什么部分服务不可用了？_
 
  这是目前Envoy sidecar实现的一个已知问题。在创建规则2秒以后，服务就应当可用。

* _可以不设置路由规则直接使用标准Ingress规格吗？_

  主机、TLS和基于扩展路径匹配的简单ingress规格，都可以开箱即用，而无需路由规则。但需要注意的是，ingress资源所用的路径不能包含`.`字符。
  
  举个例子，下列ingress资源匹配example.com主机上的/helloworld请求。
  
  ```bash
  cat <<EOF | kubectl create -f -
  apiVersion: extensions/v1beta1
  kind: Ingress
  metadata:
    name: simple-ingress
    annotations:
      kubernetes.io/ingress.class: istio
  spec:
    rules:
    - host: example.com
      http:
        paths:
        - path: /helloworld
          backend:
            serviceName: myservice
            servicePort: grpc
  EOF
  ```
 
  但是下列规则并不能工作，因为在路径中使用了正则表达式，并使用了`ingress.kubernetes.io`注释。

  ```bash
  cat <<EOF | kubectl create -f -
  apiVersion: extensions/v1beta1
  kind: Ingress
  metadata:
    name: this-will-not-work
    annotations:
      kubernetes.io/ingress.class: istio
      # Ingress annotations other than ingress class will not be honored
      ingress.kubernetes.io/rewrite-target: /
  spec:
    rules:
    - host: example.com
      http:
        paths:
        - path: /hello(.*?)world/
          backend:
            serviceName: myservice
            servicePort: grpc
  EOF
  ```
 
