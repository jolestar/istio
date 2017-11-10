# 安装

在非 kubernetes 环境中使用 Istio  主要包含以下关键任务：

1. 使用 Istio API 服务搭建和设置 Istio 控制面板
2. 添加 Istio sidecar到每个服务的实例中
3. 确保请求是通过 sidecar 路由的

## 安装控制平面

Istio 控制面板由四个主要服务组成： Pilot, Mixer, CA, 以及 API 服务.

### API 服务器

Istio 的 API 服务器（基于 Kubernetes 的 API 服务器）提供了配置管理以及 RBAC（基于角色的访问控制）等关键的功能。API 服务器需要一个 etcd 集群作为持久化存储。搭建 API 服务器的详细说明在[这里](https://kubernetes.io/docs/getting-started-guides/scratch/#apiserver-controller-manager-and-scheduler)可以找到。Kubernetes API 服务器的配置选项文档在[这里](https://kubernetes.io/docs/admin/kube-apiserver/)。

#### 本地安装

为了进行_概念验证_(_proof of concept_ )，可以使用以下 Docker Compose 文件安装简单的单容器 API 服务器：

```yaml
version: '2'
services:
  etcd:
    image: quay.io/coreos/etcd:latest
    networks:
      default:
        aliases:
          - etcd
    ports:
      - "4001:4001"
      - "2380:2380"
      - "2379:2379"
    environment:
      - SERVICE_IGNORE=1
    command: [
              "/usr/local/bin/etcd",
              "-advertise-client-urls=http://0.0.0.0:2379",
              "-listen-client-urls=http://0.0.0.0:2379"
             ]

  istio-apiserver:
    image: gcr.io/google_containers/kube-apiserver-amd64:v1.7.3
    networks:
      default:
        aliases:
          - apiserver
    ports:
      - "8080:8080"
    privileged: true
    environment:
      - SERVICE_IGNORE=1
    command: [
               "kube-apiserver", "--etcd-servers", "http://etcd:2379", 
               "--service-cluster-ip-range", "10.99.0.0/16", 
               "--insecure-port", "8080", 
               "-v", "2", 
               "--insecure-bind-address", "0.0.0.0"
             ]
```


### 其它 Istio 组件

Istio 的 Pilot，Mixer 和 CA 的 Debian 系统的软件包可通过 Istio 发行版获得。或者，这些组件也可以通过 Docker 容器运行（docker.io/istio/pilot，docker.io/istio/mixer，docker.io/istio/istio-ca）。注意，这些组件是无状态的，所以可以水平伸缩。 每个组件都依赖于 Istio API 服务器，而 Istio API 服务器又依赖于 etcd 集群来实现持久化。


## 向服务实例中添加 Sidecar

应用程序中的每个服务实例必须伴随着 Istio 的 sidecar。 根据你的安装单元（Docker 容器，虚拟机，裸机节点），Istio sidecar 需要同时安装到这些组件中。 例如，如果你的基础架构使用虚拟机，则必须在每个需要成为 service mesh 一部分的虚拟机上运行 Istio sidecar 程序进程。

## 通过 Istio Sidecar 路由流量

sidecar 安装应该包括设置适当 iptable 规则，将应用程序的网络流量透明路由到 Istio sidecar。 可以在[这里](https://github.com/istio/istio/blob/master/pilot/docker/prepare_proxy.sh)可以找到设置这种转发的 iptables 脚本。

注意：这个脚本必须在启动应用程序或sidecar进程之前执行。

> 注意：这个脚本必须在启动应用程序或 sidecar 进程之前执行。