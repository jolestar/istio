通过本任务将学习如何：

* 验证Istio手动TLS认证安装

* 手动测试认证

## 开始之前

本任务假设已有一个Kubernetes集群：

* 已根据[the Istio installation task]({{home}}/docs/setup/kubernetes/quick-start.html)安装Istio并支持手动TLS认证。注意，要在"[Installation steps]({{home}}/docs/setup/kubernetes/quick-start.html#installation-steps)"中的第5步选择"enable Istio mutual TLS Authentication feature"。

## 验证Istio手动TLS认证安装

以下命令假设服务部署在默认命名空间。使用参数*-n yournamespace*来指定一个不同于默认值的命名空间。

### 验证Istio CA

验证集群级别的CA已在运行：

```bash
kubectl get deploy -l istio=istio-ca -n istio-system
```

```bash
NAME       DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
istio-ca   1         1         1            1           1m
```

Istio CA is up if the "AVAILABLE" column is 1.
如果"AVAILABLE"列为1，Istio CA会上升。

### 验证服务配置

1. 验证ConfigMap中的AuthPolicy设置

   ```bash
   kubectl get configmap istio -o yaml -n istio-system | grep authPolicy | head -1
   ```

   如果`authPolicy: MUTUAL_TLS`一行被取消注释（即没有`#`），则表示Istio的手动TLS认证是打开的。

## 测试认证安装

当打开手动TLS认证运行Istio时，可以在一个服务的envoy中使用curl来发送请求到其他服务。
举个例子，启动示例应用[BookInfo]({{home}}/docs/guides/bookinfo.html)后，可以ssh到`productpage`服务的envoy容器中，并通过curl发送请求到其他服务。

有以下几个步骤：
   
1. 获取productpage pod名称
   ```bash
   kubectl get pods -l app=productpage
   ```
   ```bash
   NAME                              READY     STATUS    RESTARTS   AGE
   productpage-v1-4184313719-5mxjc   2/2       Running   0          23h
   ```

   确认pod是"Running"状态。

1. ssh到envoy容器
   ```bash
   kubectl exec -it productpage-v1-4184313719-5mxjc -c istio-proxy /bin/bash
   ```

1. 确认 key/cert在/etc/certs/目录中
   ```bash
   ls /etc/certs/ 
   ```
   ```bash
   cert-chain.pem   key.pem   root-cert.pem
   ``` 
   
   Note that cert-chain.pem is envoy's cert that needs to present to the other side. key.pem is envoy's private key paired with cert-chain.pem. root-cert.pem is the root cert to verify the other side's cert. Currently we only have one CA, so all envoys have the same root-cert.pem.  
   注意，cert-chain.pem是envoy的证书，需要在另一侧使用。key.pem是envoy的与cert-chain.pem配对的私钥。root-cert.pem是根证书，用来验证其他方的证书。目前只有一个CA，因此所有envoy都有相同的root-cert.pem。
   
1. 发送请求到另一个服务，比如到detail服务：
   ```bash
   curl https://details:9080/details/0 -v --key /etc/certs/key.pem --cert /etc/certs/cert-chain.pem --cacert /etc/certs/root-cert.pem -k
   ```
   ```bash
   ...
   < HTTP/1.1 200 OK
   < content-type: text/html; charset=utf-8
   < content-length: 1867
   < server: envoy
   < date: Thu, 11 May 2017 18:59:42 GMT
   < x-envoy-upstream-service-time: 2
   ...
   ```
  
服务名称和端口定义参见[这里](https://github.com/istio/istio/blob/master/samples/bookinfo/kube/bookinfo.yaml)。

注意，Istio使用[Kubernetes service account](https://kubernetes.io/docs/tasks/configure-pod-container/configure-service-account) 作为服务身份，这比使用服务名称(参考 [这里]({{home}}/docs/concepts/security/mutual-tls.html#identity)查看更多)有更强的安全性。
因此Istio使用的证书没有服务名称，而curl需要使用该信息来验证服务身份。结果就是，使用curl的'-k'选项来阻止curl客户端在服务（比如 productpage）认证中验证服务身份。

请检查安全命名 [这里]({{home}}/docs/concepts/security/mutual-tls.html#workflow)，获取更多关于Istio中客户端如何验证服务器身份的信息。

## 进阶阅读 

* 参考博客[blog]({{home}}/blog/istio-auth-for-microservices.html)学习关于多个服务之间Istio的自动mTLS认证机制的设计原理。
