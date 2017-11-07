# 安装

在不同的环境下（如 Kubernetes、Consul 等）安装 Istio 控制平面，以及在应用程序部署中安装 sidecar。

- [Kubernetes](kubernetes/index.md)
  - [快速开始](kubernetes/quick-start.md)：在 kubernetes 集群中快速安装 Istio service mesh 的说明。
  - [安装 Istio sidecar](kubernetes/sidecar-injection.md)：使用Istio初始化工具或者使用 istioctl 命令行工具在应用程序的 pod 中安装 Istio sidecar 的说明。
  - [Istio Mesh 扩张](kubernetes/mesh-expansion.md)：将虚拟机或裸机集成到安装在 kubernetes 集群上的 Istio mesh 中的说明。
  - [FAQ](kubernetes/faq.md)：常见问题，目前关于该主题的限制和故障排查。
- [Nomad 和 Consul](consul/index.md)
  - [使用 Docker 快速开始](consul/quick-start.md)：使用  Docker Compose 快速安装 Istio service mesh 指南。
  - [安装](consul/installation.md)：在基于 Consul 的环境中安装 Istio 控制平面，不论是否使用 Nomad。
  - [FAQ](consul/faq.md)：常见问题，当前的限制与问题排查。
- [Eureka](eureka/index.md)
  - [使用 Docker 快速开始](eureka/quick-start.md)：使用  Docker Compose 快速安装 Istio service mesh 指南。
  - [安装](eureka/install.md)：如何在基于 Eureka 的环境中安装 Isto 控制平面的说明。
  - [FAQ](eureka/faq.md)：常见问题，当前限制和故障排查。
- [Cloud Foundry](cloudfoundry/index.md)
  - [安装](cloudfoundry/install.md)
- [Mesos](mesos/index.md)
  - [安装](mesos/install.md)