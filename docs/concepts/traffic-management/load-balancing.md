# 发现和负载均衡

本节描述Istio如何在服务网格中的服务实例之间实现流量的负载均衡。

**服务注册:** Istio假定存在服务注册表以跟踪应用程序中服务的pod/VM。它还假设服务的新实例自动注册到服务注册表，并且不健康的实例将被自动删除。诸如Kubernetes，Mesos等平台已经为基于容器的应用程序提供了这样的功能。为基于虚拟机的应用程序提供了大量的解决方案。

**服务发现:** Pilot 使用来自服务注册的信息，并提供与平台无关的服务发现接口。网格中的Envoy实例执行服务发现，并相应地动态更新其负载均衡池。

<img src="./img/pilot/LoadBalancing.svg" width="50%" height="50%" alt="Discovery and Load Balancing" title="Discovery and Load Balancing" />

如上图所示，网格中的服务使用其DNS名称彼此访问。绑定到服务的所有HTTP流量都会自动通过Envoy重新路由。Envoy在负载均衡池中的实例之间分发流量。虽然Envoy支持几种 [复杂的负载均衡算法][]，但Istio目前允许三种负载平衡模式：轮循，随机和带权重的最少请求。

除了负载均衡外，Envoy还会定期检查池中每个实例的运行状况。Envoy遵循熔断器风格模式，根据健康检查API调用的失败率将实例分类为不健康或健康。换句话说，当给定实例的健康检查失败次数超过预定阈值时，它将从负载均衡池中弹出。类似地，当通过的健康检查数超过预定阈值时，该实例将被添加回负载均衡池。您可以在 [处理故障](handling-failures.md) 中了解更多有关Envoy的故障处理功能。

服务可以通过使用HTTP 503响应健康检查来主动减轻负担。在这种情况下，服务实例将立即从调用者的负载均衡池中删除。

[复杂的负载均衡算法]: https://lyft.github.io/envoy/docs/intro/arch_overview/load_balancing.html
