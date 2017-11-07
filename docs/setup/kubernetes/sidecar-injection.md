# 安装 Istio sidecar

## Pod Spec 需满足的条件

为了成为 Service Mesh 中的一部分，kubernetes 集群中的每个 Pod 都必须满足如下条件：

1. _**Service 注解**：_每个 pod 都必须只属于某**一个** [Kubernetes Service](https://kubernetes.io/docs/concepts/services-networking/service/) （当前不支持一个 pod 同时属于多个 service）。
2. _**命名的端口**：_ Service 的端口必须命名。端口的名字必须遵循如下格式 `<protocol>[-<suffix>]`，可以是_http_、_http2_、 _grpc_、 _mongo_、 或者 _redis_ 作为 `<protocol>` ，这样才能使用 Istio 的路由功能。例如`name: http2-foo` 和 `name: http` 都是有效的端口名称，而 `name: http2foo` 不是。如果端口的名称是不可识别的前缀或者未命名，那么该端口上的流量就会作为普通的 TCP 流量（除非使用 `Protocol: UDP` 明确声明使用 UDP 端口）。
3. _**带有 app label 的 Deployment**：_ 我们建议 kubernetes 的`Deploymenet` 资源的配置文件中为 Pod 明确指定 `app` label。每个Deployment 的配置中都需要有个不同的有意义的 `app` 标签。`app` label 用于在分布式坠重中添加上下文信息。
4. _**Mesh 中的每个 pod 里都有一个 Sidecar**：_ 最后，Mesh 中的每个 pod 都必须运行与 Istio 兼容的sidecar。遗爱部分介绍了将 sidecar 注入到 pod 中的两种方法：使用`istioctl` 命令行工具手动注入，或者使用 istio initializer 自动注入。注意 sidecar 不涉及到容器间的流量，因为他们都在同一个 pod 中。

## 手动注入 sidecar

The `istioctl` CLI has a convenience utility called [kube-inject]({{home}}/docs/reference/commands/istioctl.html#istioctl-kube-inject) that can be used to add the Istio sidecar specification into kubernetes workload specifications. Unlike the Initializers, `kube-inject` merely transforms the YAML specification to include the Istio sidecar. You are responsible for deploying the modified YAMLs using standard tools like `kubectl`. For example, the following command adds the sidecars into pods specified in sleep.yaml and submits the modified specification to Kubernetes:

```bash
kubectl apply -f <(istioctl kube-inject -f samples/sleep/sleep.yaml)
```

### 示例

Let us try to inject the Istio sidecar into a simple sleep service.

```bash
kubectl apply -f <(istioctl kube-inject -f samples/sleep/sleep.yaml)
```

Kube-inject subcommand adds the Istio sidecar and the init container to the deployment specification as shown in the transformed output below:

```yaml
... trimmed ...
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
        ... trimmed ...
      initContainers:
      - name: istio-init
        image: docker.io/istio/proxy_init:0.2.6
        imagePullPolicy: IfNotPresent
        args:
        ... trimmed ...
---
```

The crux of sidecar injection lies in the `initContainers` and the istio-proxy container. The output above has been trimmed for brevity.

Verify that sleep's deployment contains the sidecar. The injected version corresponds to the image TAG of the injected sidecar image. It may be different in your setup.

```bash
echo $(kubectl get deployment sleep -o jsonpath='{.metadata.annotations.sidecar\.istio\.io\/status}')
```

```bash
injected-version-9c7c291eab0a522f8033decd0f5b031f5ed0e126
```

You can view the full deployment with injected containers and volumes.

```bash
kubectl get deployment sleep -o yaml
```

## 自动注入 sidecar

Istio sidecars can be automatically injected into a Pod before deployment using an alpha feature in Kubernetes called [Initializers](https://kubernetes.io/docs/admin/extensible-admission-controllers/#what-are-initializers).

> Note: Kubernetes InitializerConfiguration is not namespaced and applies to workloads across the entire cluster. Do _not_ enable this feature in shared testing environments.

### 前置条件

Initializers need to be explicitly enabled during cluster setup as outlined [here](https://kubernetes.io/docs/admin/extensible-admission-controllers/#enable-initializers-alpha-feature). 
Assuming RBAC is enabled in the cluster, you can enable the initializers in different environments as follows:

* _GKE_

  ```bash
  gcloud container clusters create NAME \
      --enable-kubernetes-alpha \
      --machine-type=n1-standard-2 \
      --num-nodes=4 \
      --no-enable-legacy-authorization \
      --zone=ZONE
  ```

* _IBM Bluemix_ kubernetes clusters with v1.7.4 or newer versions have initializers enabled by default.

* _Minikube_

  Minikube version v0.22.1 or later is required for proper certificate configuration for the GenericAdmissionWebhook feature. Get the latest version from https://github.com/kubernetes/minikube/releases.

  ```bash
  minikube start \
      --extra-config=apiserver.Admission.PluginNames="Initializers,NamespaceLifecycle,LimitRanger,ServiceAccount,DefaultStorageClass,GenericAdmissionWebhook,ResourceQuota" \
      --kubernetes-version=v1.7.5
  ```

### 安装

You can now setup the Istio Initializer from the Istio install root directory.

```bash
kubectl apply -f install/kubernetes/istio-initializer.yaml
```

This creates the following resources:

1. The `istio-sidecar` InitializerConfiguration resource that  specifies resources where Istio sidecar should be injected. By default the Istio sidecar will be injected into `deployments`, `statefulsets`, `jobs`, and `daemonsets`.

2. The `istio-inject` ConfigMap with the default injection policy for the initializer, a set of namespaces to initialize, and template parameters to use during the injection itself. These options are explained in more detail under [configuration options](#configuration-options).

3. The `istio-initializer` Deployment that runs the initializer controller.

4. The `istio-initializer-service-account` ServiceAccount that is used by the `istio-initializer` deployment. The `ClusterRole` and `ClusterRoleBinding` are defined in `install/kubernetes/istio.yaml`. Note that `initialize` and `patch` are required on _all_ resource types. It is for this reason that the initializer is run as its own deployment and not embedded in another controller, e.g. istio-pilot.

### 验证

In order to test whether sidecar injection is working, let us take the sleep service described above. Create the deployments and services.

```bash
kubectl apply -f samples/sleep/sleep.yaml
```

You can verify that sleep's deployment contains the sidecar. The injected version corresponds to the image TAG of the injected sidecar image. It may be different in your setup.

```bash
$ echo $(kubectl get deployment sleep -o jsonpath='{.metadata.annotations.sidecar\.istio\.io\/status}')
```

```bash
injected-version-9c7c291eab0a522f8033decd0f5b031f5ed0e126
```

You can view the full deployment with injected containers and volumes.

```bash
kubectl get deployment sleep -o yaml
```

### Understanding what happened

Here's what happened after the workload was submitted to Kubernetes:

1) kubernetes adds `sidecar.initializer.istio.io` to the list of pending initializers in the workload.

2) istio-initializer controller observes a new uninitialized workload was created.  It finds its configured name `sidecar.initializer.istio.io` as the first in the list of pending initializers.

3) istio-initializer checks to see if it was responsible for initializing workloads in the namespace of the workload. No further work is done and the initializer ignores the workload if the initializer is not configured for the namespace. By default the initializer is responsible for all namespaces (see [configuration options](#configuration-options)).

4) istio-initializer removes itself from the list of pending initializers. Kubernetes will not finish creating workloads if the list of pending initializers is non-empty. A misconfigured initializer means a broken cluster.

5) istio-initializer checks the default injection policy for the mesh _and_ any possible per-workload overrides to determine whether the sidecar should be injected.

6) istio-initializer injects the sidecar template into the workload and submits it back to kubernetes via PATCH.

7) kubernetes finishes creating the workload as normal and the workload includes the injected sidecar.

### 配置选项

The istio-initializer has a global default policy for injection as well as per-workload overrides. The global policy is configured by the `istio-inject` ConfigMap (see example below). The initializer pod must be restarted to adopt new configuration changes.

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

The following are key parameters in the configuration:

1. _**policy**_

 `off` - Disable the initializer from modifying resources. The pending `sidecar.initializer.istio.io` initializer is still removed to avoid blocking creation of resources.

 `disabled` - The initializer will not inject the sidecar into resources by default for the namespace(s) being watched. Resources can enable injection using the `sidecar.istio.io/inject` annotation with value of `true`.

 `enabled` - The initializer will inject the sidecar into resources by default for the namespace(s) being watched. Resources can disable injection using the `sidecar.istio.io/inject` annotation with value of `false`.

2. _**namespaces**_

 This is a list of namespaces to watch and initialize. The special `""` namespace corresponds to `v1.NamespaceAll` and configures the initializer to initialize all namespaces. kube-system, kube-public, and  istio-system are exempt from initialization.

3. _**excludeNamespaces**_

 This is a list of namespaces to be excluded from istio initializer. It cannot be definend as `v1.NamespaceAll` or defined together with `namespaces`.

4. _**initializerName**_

 This must match the name of the initializer in the InitializerConfiguration. The initializer only processes workloads that match its configured name.

5. _**params**_

 These parameters allow you to make limited changes to the injected sidecar. Changing these values will not affect already deployed workloads.

### Overriding automatic injection

Individual workloads can override the global policy using the `sidecar.istio.io/inject` annotation. The global policy applies if the annotation is omitted.

If the value of the annotation is `true`, sidecar will be injected regardless of the global policy.

If the value of the annotation is `false`, sidecar will _not_ be injected regardless of the global policy.

The following truth table shows the combinations of global policy and per-workload overrides.

| policy   | workload annotation | injected |
| -------- | ------------------- | -------- |
| off      | N/A                 | no       |
| disabled | omitted             | no       |
| disabled | false               | no       |
| disabled | true                | yes      |
| enabled  | omitted             | yes      |
| enabled  | false               | no       |
| enabled  | true                | yes      |

For example, the following deployment will have sidecars injected, even if the global policy is `disabled`.

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

This is a good way to use auto-injection in a cluster containing a mixture of Istio and non-Istio services.


### 卸载 Initializer

运行下面的命令，删除 Istio initializer：

```bash
kubectl delete -f install/kubernetes/istio-initializer.yaml
```

注意上述命令并不会删除已注入到 Pod 中的 sidecar。要想删除这些 sidecar，需要在不使用 initializer 的情况下重新部署这些 pod。