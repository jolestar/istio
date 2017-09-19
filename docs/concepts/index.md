# 概念

本章帮助您了解Istio系统的不同部分及其使用的抽象。

- [Istio是什么?](what-is-istio.md)

  [概述](what-is-istio/overview.md)：提供Istio的概念介绍，包括其解决的问题和宏观架构。

  [设计目标](what-is-istio/goals.md)：描述了Istio设计时坚持的核心原则。

- [流量管理](traffic-management/index.md)

  [概述](traffic-management/overview.md)：概述Istio中的流量管理及其功能。

  [Pilot](traffic-management/pilot.md)：引入Pilot，负责在服务网格中管理Envoy代理的分布式部署的组件。

  [请求路由](traffic-management/request-routing.md)：描述在Istio服务网格中服务之间如何路由请求。

  [发现和负载均衡](traffic-management/load-balancing.md)：描述在网格中的服务实例之间的流量如何负载均衡。

  [处理故障](traffic-management/handling-failures.md)：Envoy中的故障恢复功能概述，可以被未经修改的应用程序来利用，以提高鲁棒性并防止级联故障。

  [故障注入](traffic-management/fault-injection.md)：介绍系统故障注入的概念，可用于发现跨服务的冲突故障恢复策略。

  [规则配置](traffic-management/rules-configuration.md)：提供Istio在服务网格中配置流量管理规则所使用的领域特定语言的高级概述。

- [网络和认证](network-and-auth/index.md)

  [认证](network-and-auth/auth.md)：认证设计的深层架构，为Istio提供了安全的通信通道和强有力的身份。

- [策略与控制](policy-and-control/index.md)

  [属性](policy-and-control/attributes.md)：解释属性的重要概念，即策略和控制是如何应用于网格中的服务之上的中心机制。

  [Mixer](policy-and-control/mixer.md)：Mixer设计的深层架构，提供服务网格内的策略和控制机制。

  [Mixer配置](policy-and-control/mixer-config.md)：用于配置Mixer的关键概念的概述。

  [Mixer Aspect配置](policy-and-control/mixer-aspect-config.md)：说明如何配置Mixer Aspect及其依赖项。