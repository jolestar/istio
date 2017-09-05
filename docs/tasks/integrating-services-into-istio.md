# 将服务集成到网格中

这个任务展示如何将Kubernetes上的应用程序与Istio进行集成。您将学习如何使用 [istioctl kube-inject](../reference/commands/istioctl.md#istioctl-kube-inject) 将 Envoy sidecar 插入到部署中.

## 开始之前

这个任务假设你已经在Kubernetes上部署了Istio。如果还没有这样做，请先完成 [安装步骤](./installing-istio.md)。

## 将Envoy sidecar注入到部署中

示例部署和用来演示此任务的服务。保存为 `apps.yaml`。

```yaml
apiVersion: v1
kind: Service
metadata:
  name: service-one
  labels:
    app: service-one
spec:
  ports:
  - port: 80
    targetPort: 8080
    name: http
  selector:
    app: service-one
---
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: service-one
spec:
  replicas: 1
  template:
    metadata:
      labels:
        app: service-one
    spec:
      containers:
      - name: app
        image: gcr.io/google_containers/echoserver:1.4
        ports:
        - containerPort: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: service-two
  labels:
    app: service-two
spec:
  ports:
  - port: 80
    targetPort: 8080
    name: http-status
  selector:
    app: service-two
---
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: service-two
spec:
  replicas: 1
  template:
    metadata:
      labels:
        app: service-two
    spec:
      containers:
      - name: app
        image: gcr.io/google_containers/echoserver:1.4
        ports:
        - containerPort: 8080
```

[Kubernetes Services](https://kubernetes.io/docs/concepts/services-networking/service/) 是正常使用Istio服务所必需的。服务端口必须命名，这些名称必须以 _http_ 或 _grpc_ 前缀开头，以利用Istio的L7路由功能，例如 `name: http-foo` 或者 `name: http` 很好。带有未命名端口或没有  _http_ 或 _grpc_ 前缀的服务将作为L4流量路由。

提交一个YAML资源到API服务器,带有被注入的 Envoy sidecar。以下任何一种方法都可以正常工作。

```bash
kubectl apply -f <(istioctl kube-inject -f apps.yaml)
```

发起一个从客户端（服务一）到服务器（service-two）的请求。

```bash
CLIENT=$(kubectl get pod -l app=service-one -o jsonpath='{.items[0].metadata.name}')
SERVER=$(kubectl get pod -l app=service-two -o jsonpath='{.items[0].metadata.name}')

kubectl exec -it ${CLIENT} -c app -- curl service-two:80 | grep x-request-id
```
```bash
x-request-id=a641eff7-eb82-4a4f-b67b-53cd3a03c399
```

验证流量被 Envoy sidecar 拦截。用sidecar访问日志比较HTTP响应中的 `x-request-id` 。`x-request-id` 是随机的. 出站请求日志中的IP是 service-two pod 的IP。

客户端pod的代理上的出站请求。

```bash
kubectl logs ${CLIENT} proxy | grep a641eff7-eb82-4a4f-b67b-53cd3a03c399
```
```bash
[2017-05-01T22:08:39.310Z] "GET / HTTP/1.1" 200 - 0 398 3 3 "-" "curl/7.47.0" "a641eff7-eb82-4a4f-b67b-53cd3a03c399" "service-two" "10.4.180.7:8080"
```

服务器pod代理上的入站请求。

```bash
kubectl logs ${SERVER} proxy | grep a641eff7-eb82-4a4f-b67b-53cd3a03c399
```
```bash
[2017-05-01T22:08:39.310Z] "GET / HTTP/1.1" 200 - 0 398 2 0 "-" "curl/7.47.0" "a641eff7-eb82-4a4f-b67b-53cd3a03c399" "service-two" "127.0.0.1:8080"
```

Envoy sidecar 不拦截相同 pod 内的容器到容器的流量,当流量是通过 localhost 通讯时。这是设计决定的。

```bash
kubectl exec -it ${SERVER} -c app -- curl localhost:8080 | grep x-request-id
```

## 理解发生了什么

`istioctl kube-inject` 在向Kubernetes API服务器提交_之前_，将额外的容器注入客户端的YAML资源。这个将最终被服务器端注入取代。使用

```bash
kubectl get deployment service-one -o yaml
```

来检查已修改的部署，并查找以下内容：

* A proxy container which includes the Envoy proxy and agent to manage
  local proxy configuration.

* An [init-container](https://kubernetes.io/docs/concepts/workloads/pods/init-containers/)
  to program [iptables](https://en.wikipedia.org/wiki/Iptables).

The proxy container runs with a specific UID so that the iptables can
differentiate outbound traffic from the proxy itself from the
applications which are redirected to proxy.

```yaml
- args:
    - proxy
    - sidecar
    - "-v"
    - "2"
  env:
    -
      name: POD_NAME
      valueFrom:
        fieldRef:
          apiVersion: v1
          fieldPath: metadata.name
    -
      name: POD_NAMESPACE
      valueFrom:
        fieldRef:
          apiVersion: v1
          fieldPath: metadata.namespace
    -
      name: POD_IP
      valueFrom:
        fieldRef:
          apiVersion: v1
          fieldPath: status.podIP
  image: "docker.io/istio/proxy:<...tag... >"
  imagePullPolicy: Always
  name: proxy
  securityContext:
    runAsUser: 1337

```

iptables is used to transparently redirect all inbound and outbound
traffic to the proxy. An init-container is used for two reasons:

1. iptables requires
[NET_CAP_ADMIN](http://man7.org/linux/man-pages/man7/capabilities.7.html).

2. The sidecar iptable rules are fixed and don't need to be updated
after pod creation. The proxy container is responsible for dynamically
routing traffic.

   ```json
   {
     "name":"init",
     "image":"docker.io/istio/init:<..tag...>",
     "args":[ "-p", "15001", "-u", "1337" ],
     "imagePullPolicy":"Always",
     "securityContext":{
       "capabilities":{
         "add":[
           "NET_ADMIN"
         ]
       }
     }
   },
   ```

## Cleanup

Delete the example services and deployment.

```bash
kubectl delete -f apps.yaml
```

## What's next

* Review full documentation for [istioctl kube-inject]({{home}}/docs/reference/commands/istioctl.html#istioctl-kube-inject)

* See the [BookInfo]({{home}}/docs/samples/bookinfo.html) sample for a more complete example of applications integrated on Kubernetes with Istio.
