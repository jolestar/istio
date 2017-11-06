# 快速开始

## 前置条件

下面的操作说明需要您可以访问 kubernetes **1.7.3 后更高版本** 的集群，并且启用了 [RBAC (基于角色的访问控制)](https://kubernetes.io/docs/admin/authorization/rbac/)。您需要安装了 **1.7.3 或更高版本** 的 `kubectl` 命令。如果您希望启用 [自动注入 sidecar](sidecar-injection.md#自动注入-sidecar)，您需要启用 kubernetes 集群的 alpha 功能。

> 注意：如果您安装了 Istio 0.1.x，在安装新版本前请先 [卸载](#卸载) 它们（包括已启用 Istio 应用程序 Pod 中的 sidecar）。

* 取决于您的 kubernetes 提供商：

  * 本地安装 Istio，安装最新版本的 [Minikube](https://kubernetes.io/docs/getting-started-guides/minikube/) (version 0.22.1 或者更高)。

  * [Google Container Engine](https://cloud.google.com/container-engine)

    * 使用 kubectl 获取证书 （使用您自己的集群的名字替换 `<cluster-name>` ，使用集群实际所在的位置替换 `<zone>` ）：
  ```bash
  gcloud container clusters get-credentials <cluster-name> --zone <zone> --project <project-name>
  ```

    * 将集群管理员权限授予当前用户（需要管理员权限才能为Istio创建必要的RBAC规则）：
  ```bash
  kubectl create clusterrolebinding cluster-admin-binding --clusterrole=cluster-admin --user=$(gcloud config get-value core/account)
  ```

  * [IBM Bluemix Container Service](https://www.ibm.com/cloud-computing/bluemix/containers)

    * 使用 kubectl 获取证书 （使用您自己的集群的名字替换 `<cluster-name>` ）：
  ```bash
  $(bx cs cluster-config <cluster-name>|grep "export KUBECONFIG")
  ```

  * [Openshift Origin](https://www.openshift.org) 3.7 或者以上版本：

    * 默认情况下，Openshift 不允许以 UID 0运行容器。为 Istio 的入口（ingress）和出口（egress）service account 启用使用UID 0运行的容器：
  ```bash
  oc adm policy add-scc-to-user anyuid -z istio-ingress-service-account -n istio-system
  oc adm policy add-scc-to-user anyuid -z istio-egress-service-account -n istio-system
  oc adm policy add-scc-to-user anyuid -z default -n istio-system
  ```

    * 运行应用程序 Pod 的 service account 需要特权安全性上下文限制，以此作为 sidecar 注入的一部分:
  ```bash
  oc adm policy add-scc-to-user privileged -z default -n <target-namespace>
  ```

* 安装或升级 Kubernetes 命令行工具 [kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/) 以匹配您的集群版本（1.7或以上版本）。

## 安装步骤

不论对于哪个 Istio 发行版，都安装到 `istio-system` namespace 下，即可以管理所有其它 namespace 下的微服务。

1. 到 [Istio release](https://github.com/istio/istio/releases) 页面上，根据您的操作系统下载对应的发行版。如果您使用的是 MacOS 或者 Linux 系统，可以使用下面的额命令自动下载和解压最新的发行版：
  ```bash
  curl -L https://git.io/getLatestIstio | sh -
  ```

2. 解压安装文件，切换到文件所在目录。安装文件目录下包含：
    * `install/` 目录下是 kubernetes 使用的 `.yaml` 安装文件
    * `samples/` 目录下是示例程序
    * `istioctl` 客户端二进制文件在 `bin` 目录下。`istioctl` 文件用户手动注入 Envoy sidecar 代理、创建路由和策略等。
    * `istio.VERSION` 配置文件

3. 切换到 istio 包的解压目录。例如 istio-0.2.7：
  ```bash
  cd istio-0.2.7
  ```

4. 将 `istioctl` 客户端二进制文件加到 PATH 中。

   例如，在 MacOS 或 Linux 系统上执行下面的命令：
   ```bash
   export PATH=$PWD/bin:$PATH
   ```

5. 安装 Istio 的核心部分。选择面两个 **互斥** 选项中的之一：

  a) 安装 Istio 的时候不启用 sidecar 之间的 [TLS 交互认证](../../concepts/security/mutual-tls.md)：

  为具有现在应用程序的集群选择该选项，使用 Istio sidecar 的服务需要能够与非 Istio Kubernetes 服务以及使用 [liveliness 和 readiness 探针](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-probes/)、headless service 和 StatefulSet 的应用程序通信。

  ```bash
  kubectl apply -f install/kubernetes/istio.yaml
  ```

  _**OR**_

  b) 安装 Istio 的时候启用 sidecar 之间的 [TLS 交互认证](../../concepts/security/mutual-tls.md)：
  ```bash
  kubectl apply -f install/kubernetes/istio-auth.yaml
  ```

  这两个选项都会创建 `istio-system` 命名空间以及所需的 RBAC 权限，并部署 Istio-Pilot、Istio-Mixer、Istio-Ingress、Istio-Egress 和 Istio-CA（证书颁发机构）。

6. *可选的*：如果您的 kubernetes 集群开启了 alpha 功能，并想要启用 [自动注入 sidecar](../../..//docs/setup/kubernetes/sidecar-injection.md#automatic-sidecar-injection)，需要安装 Istio-Initializer：
  ```bash
  kubectl apply -f install/kubernetes/istio-initializer.yaml
  ```

## 验证安装

1. 确认系列 kubernetes 服务已经部署了： `istio-pilot`、 `istio-mixer`、`istio-ingress`、 `istio-egress`：
  ```bash
  kubectl get svc -n istio-system
  ```
   ```bash
  NAME            CLUSTER-IP      EXTERNAL-IP       PORT(S)                       AGE
  istio-egress    10.83.247.89    <none>            80/TCP                        5h
  istio-ingress   10.83.245.171   35.184.245.62     80:32730/TCP,443:30574/TCP    5h
  istio-pilot     10.83.251.173   <none>            8080/TCP,8081/TCP             5h
  istio-mixer     10.83.244.253   <none>            9091/TCP,9094/TCP,42422/TCP   5h
   ```

  注意：如果您运行的集群不支持外部负载均衡器（如 minikube）， `istio-ingress`  服务的 `EXTERNAL-IP` 显示  `<pending>`。你必须改为使用 NodePort service 或者 端口转发方式来访问应用程序。

2. 确人对应的 Kubernetes pod 已部署并且所有的容器都启动并运行：
   `istio-pilot-*`、 `istio-mixer-*`、 `istio-ingress-*`、 `istio-egress-*`、`istio-ca-*`， `istio-initializer-*` 是可以选的。
   ```bash
     kubectl get pods -n istio-system
   ```
   ```bash 
     istio-ca-3657790228-j21b9           1/1       Running   0          5h
     istio-egress-1684034556-fhw89       1/1       Running   0          5h
     istio-ingress-1842462111-j3vcs      1/1       Running   0          5h
     istio-initializer-184129454-zdgf5   1/1       Running   0          5h
     istio-pilot-2275554717-93c43        1/1       Running   0          5h
     istio-mixer-2104784889-20rm8        2/2       Running   0          5h
   ```

## 部署应用

您可以部署自己的应用或者示例应用程序如 [BookInfo](../../../docs/guides/bookinfo.md)。
注意：应用程序必须使用 HTTP/1.1 或 HTTP/2.0 协议来传递 HTTP 流量，因为 HTTP/1.0 已经不再支持。

如果您启动了 [Istio-Initializer](../../../docs/setup/kubernetes/sidecar-injection.md)，如上所示，您可以使用 `kubectl create` 直接部署应用。Istio-Initializer 会向应用程序的 pod 中自动注入 Envoy 容器：

   ```bash
kubectl create -f <your-app-spec>.yaml
   ```

如果您没有安装 Istio-initializer 的话，您必须使用 [istioctl kube-inject](../../../docs/reference/commands/istioctl.md#istioctl-kube-inject) 命令在部署应用之前向应用程序的 pod 中手动注入 Envoy 容器：
   ```bash
kubectl create -f <(istioctl kube-inject -f <your-app-spec>.yaml)
   ```

## 卸载

* 卸载 Istio initializer:

  如果您安装 Isto 的时候启用了 initializer，请卸载它：
   ```bash
  kubectl delete -f install/kubernetes/istio-initializer.yaml
   ```

* 卸载 Istio 核心组件。对于某一 Istio 版本，删除 RBAC 权限，`istio-system` namespace，和该命名空间的下的各层级资源。

   不必理会在层级删除过程中的各种报错，因为这些资源可能已经被删除的。

    a) 如果您在安装 Istio 的时候关闭了 TLS 交互认证：

   ```bash
   kubectl delete -f install/kubernetes/istio.yaml
   ```

   **或者**

   b) 如果您在安装 Istio 的时候启用到了 TLS 交互认证：

   ```bash
   kubectl delete -f install/kubernetes/istio-auth.yaml
   ```

## 下一步

* 查看 [BookInfo](../../../docs/guides/bookinfo.md) 应用程序示例

* 查看如何 [验证Istio交互TLS认证](../../../docs/tasks/security/mutual-tls.md)
