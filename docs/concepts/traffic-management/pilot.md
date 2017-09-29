# Pilot

Pilot负责在Istio服务网格中部署的Envoy实例的生命周期。

<img src="./img/pilot/PilotAdapters.svg" width="50%" height="50%" alt="Pilot's overall architecture." title="Pilot Architecture" />

如上图所示，Pilot维护了网格中的服务的规范表示，这个表示是独立于底层平台的。Pilot中的平台特定适配器负责适当填充此规范模型。例如，Pilot中的Kubernetes适配器实现必要的控制器来查看Kubernetes API服务器，以得到pod注册信息的更改，入口资源以及存储流量管理规则的第三方资源。该数据被翻译成规范表示。Envoy特定配置是基于规范表示生成的。

Pilot公开了用于 [服务发现](https://envoyproxy.github.io/envoy/configuration/cluster_manager/sds_api.html) 、[负载均衡池](https://envoyproxy.github.io/envoy/configuration/cluster_manager/cds.html) 和 [路由表](https://envoyproxy.github.io/envoy/configuration/http_conn_man/rds.html) 的动态更新的 API。这些API将Envoy从平台特有的细微差别中解脱出来，简化了设计并提升了跨平台的可移植性。

运维人员可以通过 [Pilot的Rules API](../../reference/config/traffic-rules/index.md) 指定高级流量管理规则。这些规则被翻译成低级配置，并通过discovery API分发到Envoy实例。

[服务发现]: https://lyft.github.io/envoy/docs/configuration/cluster_manager/sds_api.html
[负载均衡池]: https://lyft.github.io/envoy/docs/configuration/cluster_manager/cds.html
[路由表]: https://lyft.github.io/envoy/docs/configuration/http_conn_man/rds.html