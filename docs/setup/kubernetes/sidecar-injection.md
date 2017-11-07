# 安装 Istio sidecar

## Pod Spec 中需满足的条件

为了成为 Service Mesh 中的一部分，kubernetes 集群中的每个 Pod 都必须满足如下条件：

1. **Service 注解**：每个 pod 都必须只属于某**一个** [Kubernetes Service](https://kubernetes.io/docs/concepts/services-networking/service/) （当前不支持一个 pod 同时属于多个 service）。
2. **命名的端口**：Service 的端口必须命名。端口的名字必须遵循如下格式 `<protocol>[-<suffix>]`，可以是_http_、_http2_、 _grpc_、 _mongo_、 或者 _redis_ 作为 `<protocol>` ，这样才能使用 Istio 的路由功能。例如`name: http2-foo` 和 `name: http` 都是有效的端口名称，而 `name: http2foo` 不是。如果端口的名称是不可识别的前缀或者未命名，那么该端口上的流量就会作为普通的 TCP 流量（除非使用 `Protocol: UDP` 明确声明使用 UDP 端口）。
3. **带有 app label 的 Deployment**：我们建议 kubernetes 的`Deploymenet` 资源的配置文件中为 Pod 明确指定 `app` label。每个Deployment 的配置中都需要有个不同的有意义的 `app` 标签。`app` label 用于在分布式坠重中添加上下文信息。
4. **Mesh 中的每个 pod 里都有一个 Sidecar**：最后，Mesh 中的每个 pod 都必须运行与 Istio 兼容的sidecar。遗爱部分介绍了将 sidecar 注入到 pod 中的两种方法：使用`istioctl` 命令行工具手动注入，或者使用 istio initializer 自动注入。注意 sidecar 不涉及到容器间的流量，因为他们都在同一个 pod 中。

## 手动注入 sidecar

`istioctl` 命令行中有一个称为 [kube-inject](../../../docs/reference/commands/istioctl.md#istioctl-kube-inject) 的便利工具，使用它可以将 Istio 的 sidecar 规范添加到 kubernetes 工作负载的规范配置中。与 Initializer 程序不同，`kube-inject` 只是将 YAML 规范转换成包含 Istio sidecar 的规范。您需要使用标准的工具如 `kubectl` 来部署修改后的 YAML。例如，以下命令将 sidecar 添加到 sleep.yaml 文件中指定的 pod 中，并将修改后的规范提交给 kubernetes：

```bash
kubectl apply -f <(istioctl kube-inject -f samples/sleep/sleep.yaml)
```

### 示例

我们来试一试将 Istio sidecar  注入到 sleep 服务中去。

```bash
kubectl apply -f <(istioctl kube-inject -f samples/sleep/sleep.yaml)
```

Kube-inject 子命令将 Istio sidecar 和 init 容器注入到 deployment 配置中，转换后的输出如下所示：

```yaml
... 略过 ...
---
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  annotations:
    sidecar.istio.io/status: injected-version-root@69916ebba0fc-0.2.6-081ffece00c82cb9de33cd5617682999aee5298d
  name: sleep
spec:
  replicas: 1
  template:
    metadata:
      annotations:
        sidecar.istio.io/status: injected-version-root@69916ebba0fc-0.2.6-081ffece00c82cb9de33cd5617682999aee5298d
      labels:
        app: sleep
    spec:
      containers:
      - name: sleep
        image: tutum/curl
        command: ["/bin/sleep","infinity"]
        imagePullPolicy: IfNotPresent
      - name: istio-proxy
        image: docker.io/istio/proxy_debug:0.2.6
        args:
        ... 略过 ...
      initContainers:
      - name: istio-init
        image: docker.io/istio/proxy_init:0.2.6
        imagePullPolicy: IfNotPresent
        args:
        ... 略过 ...
---
```

注入 sidecar 的关键在于 `initContainers` 和 istio-proxy 容器。为了简洁起见，上述输出有所省略。

验证 sleep deployment 中包含 sidecar。injected-version 对应于注入的 sidecar 镜像的版本和镜像的 TAG。在您的设置的可能会有所不同。

```bash
echo $(kubectl get deployment sleep -o jsonpath='{.metadata.annotations.sidecar\.istio\.io\/status}')
```

```bash
injected-version-9c7c291eab0a522f8033decd0f5b031f5ed0e126
```

你可以查看包含注入的容器和挂载的 volume 的完整 deployment 信息。

```bash
kubectl get deployment sleep -o yaml
```

## 自动注入 sidecar

Istio sidecar 可以在部署之前使用 Kubernetes 中一个名为 [Initializer](https://kubernetes.io/docs/admin/extensible-admission-controllers/#what-are-initializers) 的 Alpha 功能自动注入到 Pod 中。

> 注意：Kubernetes InitializerConfiguration没有命名空间，适用于整个集群的工作负载。不要在共享测试环境中启用此功能。

### 前置条件

Initializer 需要在集群设置期间显示启用，如 [此处](https://kubernetes.io/docs/admin/extensible-admission-controllers/#enable-initializers-alpha-feature) 所述。
假设集群中已启用RBAC，则可以在不同环境中启用初始化程序，如下所示：

* _GKE_

  ```bash
  gcloud container clusters create NAME \
      --enable-kubernetes-alpha \
      --machine-type=n1-standard-2 \
      --num-nodes=4 \
      --no-enable-legacy-authorization \
      --zone=ZONE
  ```

* _IBM Bluemix_ kubernetes v1.7.4 或更高版本的集群已默认启用 initializer。

* _Minikube_

  Minikube v0.22.1 或更高版本需要为 GenericAdmissionWebhook 功能配置适当的证书。获取最新版本： https://github.com/kubernetes/minikube/releases.

  ```bash
  minikube start \
      --extra-config=apiserver.Admission.PluginNames="Initializers,NamespaceLifecycle,LimitRanger,ServiceAccount,DefaultStorageClass,GenericAdmissionWebhook,ResourceQuota" \
      --kubernetes-version=v1.7.5
  ```

### 安装

您现在可以从 Istio 安装根目录设置 Istio Initializer。

```bash
kubectl apply -f install/kubernetes/istio-initializer.yaml
```

将会创建下列资源：

1. `istio-sidecar` InitializerConfiguration 资源，指定 Istio sidecar 注入的资源。默认情况下  Istio sidecar 将被注入到 `deployment`、 `statefulset`、 `job` 和 `daemonset`中。

2. `istio-inject` ConfigMap，initializer 的默认注入策略，一组初始化 namespace，以及注入时使用的模版参数。这些配置的详细说明请参考 [配置选项](#configuration-options)。

3. `istio-initializer` Deployment，运行 initializer 控制器。

4. `istio-initializer-service-account` ServiceAccount，用于 `istio-initializer` deployment。`ClusterRole` 和 `ClusterRoleBinding` 在 `install/kubernetes/istio.yaml` 中定义。注意所有的资源类型都需要有 `initialize` 和 `patch` 。正式处于这个原因，initializer 要作为 deployment 的一部分来运行而不是嵌入到其它控制器中，例如 istio-pilot。

### 验证

为了验证 sidecar 是否成功注入，为上面的 sleep 服务创建 deployment 和 service。

```bash
kubectl apply -f samples/sleep/sleep.yaml
```

验证 sleep deployment 中包含 sidecar。injected-version 对应于注入的 sidecar 镜像的版本和镜像的 TAG。在您的设置的可能会有所不同。

```bash
$ echo $(kubectl get deployment sleep -o jsonpath='{.metadata.annotations.sidecar\.istio\.io\/status}')
```

```bash
injected-version-9c7c291eab0a522f8033decd0f5b031f5ed0e126
```

你可以查看包含注入的容器和挂载的 volume 的完整 deployment 信息。

```bash
kubectl get deployment sleep -o yaml
```

### 了解发生了什么

以下是将工作负载提交给 Kubernetes 后发生的情况：

1) kubernetes 将 `sidecar.initializer.istio.io` 添加到工作负载的 pending initializer 列表中。

2) istio-initializer 控制器观察到有一个新的未初始化的工作负载被创建了。pending initializer 列表中的第一个个将作为 `sidecar.initializer.istio.io` 的名称。

3) istio-initializer 检查它是否负责初始化 namespace 中的工作负载。如果没有为该 namespace 配置 initializer，则不需要做进一步的工作，而且 initializer 会忽略工作负载。默认情况下，initializer 负责所有的 namespace（参考 [配置选项](#配置选项)）。

4) istio-initializer 将自己从  pending initializer 中移除。如果 pending initializer 列表非空，则 Kubernetes 不回结束工作负载的创建。错误配置的 initializer 意味着破损的集群。

5) istio-initializer 检查 mesh 的默认注入策略，并检查所有单个工作负载的策略负载值，以确定是否需要注入 sidecar。

6) istio-initializer 向工作负载中注入 sidecar 模板，然后通过 PATCH 向 kubernetes 提交。

7) kubernetes 正常的完成了工作负载的创建，并且工作负载中已经包含了注入的 sidecar。

### 配置选项

istio-initializer 具有用于注入的全局默认策略以及每个工作负载覆盖配置。全局策略由 `istio-inject` ConfigMap 配置（请参见下面的示例）。Initializer pod 必须重新启动以采用新的配置更改。

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: istio-inject
  namespace: istio-system
data:
  config: |-
    policy: "enabled"
    namespaces: [""] # everything, aka v1.NamepsaceAll, aka cluster-wide
    # excludeNamespaces: ["ns1", "ns2"]
    initializerName: "sidecar.initializer.istio.io"
    params:
      initImage: docker.io/istio/proxy_init:0.2.6
      proxyImage: docker.io/istio/proxy:0.2.6
      verbosity: 2
      version: 0.2.6
      meshConfigMapName: istio
      imagePullPolicy: IfNotPresent
```

下面是配置中的关键参数：

1. **policy**

   `off` - 禁用 initializer 修改资源。pending 的 `sidecar.initializer.istio.io` initializer 将被删除以避免创建阻塞资源。

   `disable` - initializer 不会注入 sidecar 到 watch 的所有 namespace 的资源中。启用 sidecar 注入请将 `sidecar.istio.io/inject` 注解的值设置为 `true`。

   `enable` - initializer 将会注入 sidecar 到 watch 的所有 namespace 的资源中。禁用 sidecar 注入请将 `sidecar.istio.io/inject` 注解的值设置为 `false`。


2. **namespaces**

   要 watch 和初始化的 namespace 列表。特殊的 `""` namespace 对应于 `v1.NamespaceAll` 并配置初始化程序以初始化所有 namespace。`kube-system`、`kube-publice` 和 `istio-system` 被免除初始化。


3. **excludeNamespaces**

   从 Istio initializer 中排除的 namespace 列表。不可以定义为 `v1.NamespaceAll` 或者与 `namespaces` 一起定义。


4. **initializerName**

   这必须与 InitializerConfiguration 中 initializer 设定项的名称相匹配。Initializer 只处理匹配其配置名称的工作负载。


5. **params**

   这些参数允许您对注入的 sidecar 进行有限的更改。更改这些值不会影响已部署的工作负载。

### 重写自动注入

单个工作负载可以通过使用 `sidecar.istio.io/inject` 注解重写全局策略。如果注解被省略，则使用全局策略。

如果注解的值是 `true`，则不管全局策略如何，sidecar 都将被注入。

如果注解的值是 `false`，则不管全局策略如何，sidecar 都不会被注入。

下表显示全局策略和每个工作负载覆盖的组合。

| policy   | workload annotation | injected |
| -------- | ------------------- | -------- |
| off      | N/A                 | no       |
| disabled | omitted             | no       |
| disabled | false               | no       |
| disabled | true                | yes      |
| enabled  | omitted             | yes      |
| enabled  | false               | no       |
| enabled  | true                | yes      |

例如，即使全局策略是 `disable`，下面的 deployment 也会被注入sidecar。

```yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: myapp
  annotations:
    sidecar.istio.io/inject: "true"
spec:
  replicas: 1
  template:
    ...
```

这是在包含 Istio 和非 Istio 服务的混合群集中使用自动注入的好方法。


### 卸载 Initializer

运行下面的命令，删除 Istio initializer：

```bash
kubectl delete -f install/kubernetes/istio-initializer.yaml
```

注意上述命令并不会删除已注入到 Pod 中的 sidecar。要想删除这些 sidecar，需要在不使用 initializer 的情况下重新部署这些 pod。