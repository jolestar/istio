# 安装Istio

这个任务展示如何在Kubernetes集群中安装和配置Istio。


## 前提条件

* 以下说明假设您已经可以访问Kubernetes集群。要在本地安装Kubernetes，请尝试 [minikube](https://kubernetes.io/docs/getting-started-guides/minikube/)。

* 如果您正在使用 [Google Container Engine](https://cloud.google.com/container-engine)，请找到您的集群名称和区域，并为kubectl提取凭据：

    ```bash
    gcloud container clusters get-credentials <cluster-name> --zone <zone> --project <project-name>
    ```

* 如果您正在使用 [IBM Bluemix Container Service](https://www.ibm.com/cloud-computing/bluemix/containers)，请找到您的集群名称，并为kubectl提取凭据：

    ```bash
    $(bx cs cluster-config <cluster-name>|grep "export KUBECONFIG")
    ```

* 安装Kubernetes客户端[kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/)，或升级到集群支持的最新版本。

* 如果您以前在此集群上安装过Istio，请先按照本节末尾的 [卸载](#卸载) 步骤进行卸载。

## 安装步骤

您可以使用 [Istio Helm chart](https://github.com/kubernetes/charts/tree/master/incubator/istio#installing-the-chart) 进行安装，或按照以下步骤。

对于 {{ book.siteDataIstioVersion }} 版本，Istio必须安装在与应用程序相同的Kubernetes命名空间中。以下说明将在 default 命名空间中部署Istio。也可以修改为部署在不同的命名空间中。

1. 转到 [Istio release](https://github.com/istio/istio/releases) 页面，下载与您的操作系统对应的安装文件或运行
 `curl -L https://git.io/getIstio | sh -` 自动下载并提取最新版本（在MacOS和Ubuntu上）。

1. 解压缩安装文件，并将目录更改为文件解压缩的位置。以下说明与这个安装目录相关。

	安装目录包含：

    * Kubernetes的yaml安装文件
    * 示例应用程序
    * `istioctl` 客户端二进制文件，需要注入Envoy为sidecar代理，以及用于创建路由规则和策略。
    * istio.VERSION配置文件。

1. 如果从 [Istio release](https://github.com/istio/istio/releases) 下载安装文件，请将 `istioctl` 客户端添加到 PATH 中。例如，在Linux或MacOS系统上运行以下命令：

    ```bash
    export PATH=$PWD/bin:$PATH
    ```

1. 运行以下命令以确定您的集群是否启用了 [RBAC (Role-Based Access Control／基于角色的访问控制)](https://kubernetes.io/docs/admin/authorization/rbac/):

    ```bash
    kubectl api-versions | grep rbac
    ```

   * 如果命令显示错误,或不显示任何内容，这意味找集群不支持RBAC，您可以继续执行下面的步骤5。

   * 如果命令显示'beta'版本或'alpha'和'beta'两者都有，请使用　istio-rbac-beta.yaml　配置，如下所示：

   *(注意：如果是在 `default` 命名空间之外的其他命名空间中部署Istio，请将所有ClusterRoleBinding资源中的 `namespace: default` 行替换为实际的命名空间。)*

    ```bash
    kubectl apply -f install/kubernetes/istio-rbac-beta.yaml
    ```

   如果你收到错误:

    ```
    Error from server (Forbidden): error when creating "install/kubernetes/istio-rbac-beta.yaml": clusterroles.rbac.authorization.k8s.io "istio-pilot" is forbidden: attempt to grant extra privileges: [{[*] [istio.io] [istioconfigs] [] []} {[*] [istio.io] [istioconfigs.istio.io] [] []} {[*] [extensions] [thirdpartyresources] [] []} {[*] [extensions] [thirdpartyresources.extensions] [] []} {[*] [extensions] [ingresses] [] []} {[*] [] [configmaps] [] []} {[*] [] [endpoints] [] []} {[*] [] [pods] [] []} {[*] [] [services] [] []}] user=&{user@example.org [...]
    ```

	您需要添加以下内容：（名称用您自己的替换）

    ```
    kubectl create clusterrolebinding myname-cluster-admin-binding --clusterrole=cluster-admin --user=myname@example.org
    ```

   * 如果命令仅显示'alpha'版本，请使用　istio-rbac-alpha.yaml 配置：

   *(注意：如果是在 `default` 命名空间之外的其他命名空间中部署Istio，请将所有ClusterRoleBinding资源中的 `namespace: default` 行替换为实际的命名空间。)*

    ```bash
    kubectl apply -f install/kubernetes/istio-rbac-alpha.yaml
    ```

1. 安装Istio的核心组件。

	在这个阶段有两个相互排斥的选择：

    * 安装Istio而不启用 [Istio Auth](../concepts/network-and-auth/auth.md) 功能:

        ```bash
        kubectl apply -f install/kubernetes/istio.yaml
        ```

		此命令将安装Pilot，Mixer，Ingress-Controller，Egress-Controller核心组件。

	* 安装Istio并启用 [Istio Auth](../concepts/network-and-auth/auth.md) （在命名空间中部署CA并启用服务之间的[mTLS](https://en.wikipedia.org/wiki/Mutual_authentication)）功能:

         ```bash
         kubectl apply -f install/kubernetes/istio-auth.yaml
         ```

		该命令将安装Pilot，Mixer，Ingress-Controller和Egress-Controller，以及Istio CA（证书颁发机构）。

1. *可选:* 按照以下各节所述安装用于metrics收集和/或请求追踪的插件。

### 启用metrics收集

要收集和查看Mixer提供的metrics，请安装　[Prometheus](https://prometheus.io)，以及 [Grafana](https://grafana.com/grafana/download) 和/或ServiceGraph插件。

```bash
kubectl apply -f install/kubernetes/addons/prometheus.yaml
kubectl apply -f install/kubernetes/addons/grafana.yaml
kubectl apply -f install/kubernetes/addons/servicegraph.yaml
```

您可以在 [收集指标和日志](./metrics-logs.md) 中找到更多关于如何使用这些工具的信息。

#### 验证Grafana仪表盘

Grafana插件提供了Istio 仪表盘可视化的集群中的metrics（请求率，成功/失败率）。如果您安装了Grafana，请检查您是否可以访问仪表盘。

配置 `grafana` 服务的端口转发，具体如下：

    ```bash
    kubectl port-forward $(kubectl get pod -l app=grafana -o jsonpath='{.items[0].metadata.name}') 3000:3000 &
    ```

然后将您的Web浏览器指向 [http://localhost:3000/dashboard/db/istio-dashboard](http://localhost:3000/dashboard/db/istio-dashboard). 仪表盘看上去应该是这样的：

<img style="max-width:80%" src="./img/grafana_dashboard.png" alt="Grafana Istio Dashboard" title="Grafana Istio Dashboard" />

#### 验证ServiceGraph服务

ServiceGraph插件为集群提供服务交互图的文本（JSON）表示和图形可视化。像Grafana一样，您可以使用端口转发，service nodePort或（如果外部负载均衡器可用）外部IP来访问 servicegraph 服务。在这种情况下，服务名称是 `servicegraph` 和要访问的端口是8088：

```bash
kubectl port-forward $(kubectl get pod -l app=servicegraph -o jsonpath='{.items[0].metadata.name}') 8088:8088 &
```

ServiceGraph服务提供底层服务图的文本（JSON）表示（通过`/graph`）和图形可视化（`/dotviz`）。要查看图形可视化（假设您已按照上一个片段配置了端口转发），请打开您的浏览器，网址为：[http://localhost:8088/dotviz](http://localhost:8088/dotviz)。

运行某些服务后，例如，在安装 [BookInfo](../samples/bookinfo.md) 示例应用程序并在应用程序上生成一些负载（例如，在`while`循环中执行`curl`请求）后，生成的服务图应该看上去像这样：

<img src="./img/servicegraph.png" alt="BookInfo Service Graph" title="BookInfo Service Graph" />

### 启用与Zipkin的请求跟踪

要启用和查看分布式请求跟踪，请安装[Zipkin](http://zipkin.io)插件：

```bash
kubectl apply -f install/kubernetes/addons/zipkin.yaml
```

Zipkin可用于分析Istio应用程序的请求流程和时间，并帮助识别瓶颈。您可以在 [分布式请求跟踪](./zipkin-tracing.md) 中找到有关如何访问Zipkin仪表盘并使用Zipkin的更多信息。

## 验证安装

1. 确认以下 Kubernetes 服务已经部署: "istio-pilot", "istio-mixer", "istio-ingress", "istio-egress",
   "istio-ca" (如果启用了Istio Auth), 和, 可选的, "grafana", "prometheus', "servicegraph" and "zipkin".

    ```bash
    kubectl get svc
    ```

    ```bash
    NAME            CLUSTER-IP      EXTERNAL-IP       PORT(S)                       AGE
    grafana         10.83.252.16    <none>            3000:30432/TCP                5h
    istio-egress    10.83.247.89    <none>            80/TCP                        5h
    istio-ingress   10.83.245.171   35.184.245.62     80:32730/TCP,443:30574/TCP    5h
    istio-pilot     10.83.251.173   <none>            8080/TCP,8081/TCP             5h
    istio-mixer     10.83.244.253   <none>            9091/TCP,9094/TCP,42422/TCP   5h
    kubernetes      10.83.240.1     <none>            443/TCP                       36d
    prometheus      10.83.247.221   <none>            9090:30398/TCP                5h
    servicegraph    10.83.242.48    <none>            8088:31928/TCP                5h
    zipkin          10.83.241.77    <none>            9411:30243/TCP                5h
    ```

	请注意，如果您的集群在不支持外部负载均衡器的环境中（例如minikube）运行, `istio-ingress` 的 `EXTERNAL-IP` 则会说 `<pending>`，您将需要使用服务NodePort或端口转发来访问应用程序。

2. 检查相应的Kubernetes pod已部署，所有容器启动并运行正常:
   "istio-pilot-\*", "istio-mixer-\*", "istio-ingress-\*", "istio-egress-\*", "istio-ca-\*" (如果启用了Istio Auth),
   和, 可选的, "grafana-\*", "prometheus-\*', "servicegraph-\*" 和 "zipkin-\*".

    ```bash
    kubectl get pods
    ```

    ```bash
    grafana-3836448452-vhc1v         1/1       Running   0          5h
    istio-ca-3657790228-j21b9        1/1       Running   0          5h
    istio-egress-1684034556-fhw89    1/1       Running   0          5h
    istio-ingress-1842462111-j3vcs   1/1       Running   0          5h
    istio-pilot-2275554717-93c43     2/2       Running   0          5h
    istio-mixer-2104784889-20rm8     1/1       Running   0          5h
    prometheus-3067433533-wlmt2      1/1       Running   0          5h
    servicegraph-3127588006-pc5z3    1/1       Running   0          5h
    zipkin-4057566570-k9m42          1/1       Running   0          5h
    ```

## 部署应用程序

现在您可以部署自己的应用程序，或者安装所提供的示例应用程序之一，例如 [BookInfo](../samples/bookinfo.md)。请注意，应用程序应使用 HTTP/1.1 或 HTTP/2.0 协议来实现其所有HTTP流量; HTTP/1.0不支持。

在部署应用程序时，必须使用 [istioctl kube-inject](../reference/commands/istioctl.md#istioctl-kube-inject) 来自动将Envoy容器注入到应用程序容器中：

```bash
kubectl create -f <(istioctl kube-inject -f <your-app-spec>.yaml)
```

## 卸载

您可以使用 [Istio Helm chart](https://github.com/kubernetes/charts/tree/master/incubator/istio#uninstalling-the-chart) 进行卸载，或按照以下步骤操作。

1. 卸载Istio核心组件：

   * 如果Istio是不带Istio认证功能安装：

     ```bash
     kubectl delete -f install/kubernetes/istio.yaml
     ```

   * 如果Istio是带Istio认证功能安装：

     ```bash
     kubectl delete -f install/kubernetes/istio-auth.yaml
     ```

2. 卸载RBAC Istio角色：

	* 如果安装了beta版本：

         ```bash
         kubectl delete -f install/kubernetes/istio-rbac-beta.yaml
         ```

	* 如果安装了alpha版本：

         ```bash
         kubectl delete -f install/kubernetes/istio-rbac-alpha.yaml
         ```

3. 如果安装了Istio插件，请卸载它们：

    ```bash
    kubectl delete -f install/kubernetes/addons/
    ```

4. 删除Istio Kubernetes [TPRs](https://kubernetes.io/docs/tasks/access-kubernetes-api/extend-api-third-party-resource):

    ```bash
    kubectl delete istioconfigs --all
    kubectl delete thirdpartyresource istio-config.istio.io
    ```

## 下一步

* 请参阅 [BookInfo](../samples/bookinfo.md) 示例应用程序.

* 看看如何 [测试 Istio Auth](./istio-auth.md).
