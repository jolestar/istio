# Istio官方文档中文版

[Istio](https://istio.io) 是一个开放平台，提供统一的方式来集成微服务，管理跨微服务的流量，执行策略和汇总遥测数据。Istio的控制面板在底层集群管理平台（如Kubernetes，Mesos等）上提供了一个抽象层。

这是 Istio 官方文档的中文翻译版。

当前已经完成内容：

* [官方文档](docs/index.md)
  * [概念](docs/concepts/index.md)

    * [Istio是什么?](docs/concepts/what-is-istio/index.md)
    * [概述](docs/concepts/what-is-istio/overview.md)
    * [设计目标](docs/concepts/what-is-istio/goals.md)
  * [流量管理](docs/concepts/traffic-management/index.md)
    * [概述](docs/concepts/traffic-management/overview.md)
    * [Pilot](docs/concepts/traffic-management/pilot.md)
    * [请求路由](docs/concepts/traffic-management/request-routing.md)
    * [发现和负载均衡](docs/concepts/traffic-management/load-balancing.md)
    * [处理故障](docs/concepts/traffic-management/handling-failures.md)
    * [故障注入](docs/concepts/traffic-management/fault-injection.md)
    * [规则配置](docs/concepts/traffic-management/rules-configuration.md)
  * [网络和认证](docs/concepts/network-and-auth/index.md)
      * [认证](docs/concepts/network-and-auth/auth.md)
    * [策略与控制](docs/concepts/policy-and-control/index.md)
      * [属性](docs/concepts/policy-and-control/attributes.md)
      * [Mixer](docs/concepts/policy-and-control/mixer.md)
      * [Mixer配置](docs/concepts/policy-and-control/mixer-config.md)
      * [Mixer Aspect配置](docs/concepts/policy-and-control/mixer-aspect-config.md)

后续计划中的内容：

- TBD

文档内容发布于 gitbook，请点击下面的链接阅读:

- [在线阅读](https://istio.doczh.cn)
- [gitbook书籍首页](https://www.gitbook.com/book/doczhcn/istio/)：可选择下载 pdf/epub/mobi 格式