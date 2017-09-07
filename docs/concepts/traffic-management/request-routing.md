# 请求路由

此节描述Istio服务网格中的服务之间如何路由请求。

## 服务模型和服务版本

如 [Pilot](./pilot.md) 所述，特定网格中服务的规范表示由Pilot维护。服务的Istio模型和在底层平台（Kubernetes，Mesos，Cloud Foundry等）中的表示无关。特定平台的适配器负责使用平台中的元数据的各种字段填充内部模型表示。

Istio介绍了服务版本的概念，这是一种更细微的方法，可以通过版本（`v1`，`v2`）或环境（`staging`，`prod`）细分服务实例。这些变量不一定是API版本：它们可能是对不同环境（prod，staging，dev等）部署的相同服务的迭代更改。使用这种方式的常见场景包括A/B测试或金丝雀推出。Istio的 [流量路由规则](./rules-configuration.md) 可以参考服务版本，以提供对服务之间流量的附加控制。

## 服务之间的通讯

<img src="./img/pilot/ServiceModel_Versions.svg" width="50%" height="50%" alt="Showing how service versions are handled." title="Service Versions" />

如上图所示，服务的客户端不知道服务不同版本的差异。他们可以使用服务的主机名/IP地址继续访问服务。Envoy sidecar/代理拦截并转发客户端和服务之间的所有请求/响应。

Envoy根据运维人员使用Pilot指定的路由规则，动态地确定其服务版本的实际选择。该模型使应用程序代码能够脱离其依赖服务的演进，同时提供其他好处（参见 [Mixer](../policy-and-control/mixer.md)）。路由规则允许Envoy根据诸如header，与源/目的地相关联的标签和/或分配给每个版本的权重的标准来选择版本。

Istio还为同一服务版本的多个实例提供流量负载均衡。您可以在 [服务发现和负载均衡](load-balancing.md) 中找到更多信息。

Istio不提供DNS。应用程序可以尝试使用底层平台（kube-dns，mesos-dns等）中存在的DNS服务来解析FQDN。

## 入口和出口Envoys

Istio假定进入和离开服务网络的所有流量都会通过Envoy代理进行传输。通过将Envoy代理部署在服务之前，运维人员可以针对面向用户的服务进行A/B测试，部署金丝雀服务等。类似地，通过使用Envoy将流量路由到外部Web服务（例如，访问Maps API或视频服务API），运维人员可以添加故障恢复功能，例如熔断器，通过Mixer强加限速，并使用Istio-auth提供认证。

<img src="./img/pilot/ServiceModel_RequestFlow.svg" width="50%" height="50%" alt="Ingress and Egress Envoy." title="Request Flow" />

