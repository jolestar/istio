# 概述

本文档介绍了Istio：一个连接，管理和保护微服务的开放平台。Istio提供了一种轻松的方式来创建一个已部署服务的具有负载平衡，服务到服务认证，监控等等的网络，而不需要任何服务代码的改动。您可以通过在整个环境中部署一个特殊的边车(sidecar)代理来为服务增加对Istio的支持，代理拦截微服务之间的所有网络通信，使用Istio控制面板功能配置和管理代理。

Istio目前仅支持在Kubernetes上的服务部署，但未来版本中将支持其他环境。

有关Istio组件的详细概念信息，请参阅我们的其他 [概念](../index.md)指南。

## 为什么要使用Istio？

Istio解决了开发人员和运维在单体应用程序向分布式微服务架构的转型中面临的许多挑战。术语**服务网格**通常用于描述构成这些应用程序的微服务网络以及它们之间的交互。随着服务网格的规模和复杂性的增长，它变得更难以理解和管理。它的需求包括服务发现，负载均衡，故障恢复，指标和监控，以及通常更复杂的运维需求，例如A/B测试，金丝雀发布，限流，访问控制和端到端认证。

Istio提供了一个完整的解决方案，通过为整个服务网格提供行为洞察和操作控制来满足微服务应用程序的多样化需求。它在服务网络中统一提供了许多关键功能：

- **通讯管理**。控制服务之间的通讯和API调用的流量，使得调用更可靠，并使网络在恶劣情况下更加健壮。

- **可观察性**。了解服务之间的依赖关系，以及它们之间通讯的性质和流量，从而提供快速识别问题的能力。

- **策略执行**。将组织策略应用于服务之间的互动，确保访问策略得到实施，资源在消费者之间分配。通过配置网格而不是修改应用程序代码来进行策略更改。

- **服务身份和安全**。在网格中的服务提供具有可验证身份，并提供保护在不同程度的可信度网络上流转的服务通讯的能力。

除了这些行为，Istio还设计了可扩展性以满足不同的部署需求：

- **平台支持**。Istio旨在在各种环境中运行，包括span Cloud, on-premise，Kubernetes，Mesos等各种环境。我们最初专注于Kubernetes，但正在努力很快支持其他环境。

- **集成和定制**。可以扩展和定制策略执行组件，以便与现有的ACL，日志，监控，配额，审核等解决方案集成。

这些功能大大减少了应用程序代码，底层平台和策略之间的耦合。这种减少的耦合不仅使服务更容易实现，而且还使运维人员更容易地在环境之间移动应用程序部署或新的策略方案。因此，结果就是应用程序从本质上变得更容易移动。

## 架构

Istio服务网格逻辑上分为**数据面板**和**控制面板**。

- **数据面板**是由一组的作为sidecar部署，调停和控制微服务之间的所有网络通信的智能代理（Envoy）组成的。

- **控制面板** 负责管理和配置代理来路由通讯，以及在运行时执行策略。

下图显示了构成每个面板的不同组件：

![](./img/architecture/arch.svg)

### Envoy

Istio使用[Envoy](https://lyft.github.io/envoy/)代理的扩展版本，Envoy是以C ++开发的高性能代理，用于调解服务网格中所有服务的所有入站和出站流量。Istio利用了Envoy的许多内置功能，例如动态服务发现，负载均衡，TLS termination，HTTP/2＆gRPC代理，熔断器，健康检查，基于百分比流量拆分的分段推出(译者注：灰度)，故障注入和丰富指标。

Envoy被部署为在同一个Kubernetes pod中的对应服务的sidecar。这允许Istio将大量关于流量行为的信号作为 [属性](../policy-and-control/attributes.md) 提取出来，这些属性又可以在 [Mixer](../policy-and-control/mixer.md)中用于执行策略决策，并发送给监控系统以提供有关整个网格行为的信息。sidecar代理模型还允许您将Istio功能添加到现有部署中，无需重新构建或重写代码。您可以阅读更多来了解为什么我们在 [设计目标](goals.md) 中选择这种方法。

### Mixer

[Mixer]({{home}}/docs/concepts/policy-and-control/mixer.html) is responsible for enforcing access control and usage policies across the service mesh and collecting telemetry data from the Envoy proxy and other 
services. The proxy extracts request level [attributes]({{home}}/docs/concepts/policy-and-control/attributes.html), which are sent to Mixer for evaluation. More information on this attribute extraction and policy 
evaluation can be found in [Mixer Configuration]({{home}}/docs/concepts/policy-and-control/mixer-config.html). Mixer includes a flexible plugin model enabling it to interface with a variety of host environments and infrastructure backends, abstracting the Envoy proxy and Istio-managed services from these details.

### Pilot

[Pilot]({{home}}/docs/concepts/traffic-management/pilot.html) is responsible for collecting and validating configuration and propagating it to the various Istio components.
It abstracts environment-specific implementation details from Mixer and Envoy, providing them with an abstract representation of the user’s services 
that is independent of the underlying platform. In addition, traffic management rules (i.e. generic layer-4 rules and layer-7 HTTP/gRPC routing rules) can 
be programmed at runtime via Pilot.

### Istio-Auth

[Istio-Auth]({{home}}/docs/concepts/network-and-auth/auth.html) provides strong service-to-service and end-user authentication using mutual TLS, with built-in identity and credential management.
It can be used to upgrade unencrypted traffic in the service mesh, and provides operators the ability to enforce policy based
on service identity rather than network controls. Future releases of Istio will add fine-grained access control and auditing to control
and monitor who accesses your service, API, or resource, using a variety of access control mechanisms, including attribute and
role-based access control as well as authorization hooks.

## What's next

* Learn about Istio's [design goals](./goals.html).

* Explore and try deploying our [sample application]({{home}}/docs/samples/bookinfo.html).

* Read about Istio components in detail in our other [Concepts]({{home}}/docs/concepts/) guides.

* Learn how to deploy Istio with your own services using our [Tasks]({{home}}/docs/tasks/) guides.
