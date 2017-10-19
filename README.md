# Istio官方文档中文版

## 介绍

[Istio](https://istio.io)是有Google/IBM/Lyft共同开发的新一代Service Mesh开源项目。

您可以通过以下文章快速了解Service Mesh技术和Isito项目：

- [Service Mesh：下一代微服务][servicemesh]: QCON 2017 上海站的演讲，系统介绍Service Mesh技术
- [服务网格新生代--Istio][istio]: 介绍isito的文章，比较长

目前Service Mesh技术还比较新颖，国内有了解的不多，为此我们组建了Service Mesh中国技术社区，欢迎加入。

加入方式：请联系微信ID xiaoshu062 ，注明“服务网格”。

## 内容

这是Istio官方文档的中文翻译版。

文档内容发布于gitbook，请点击下面的链接阅读或者下载电子版本:

- 在线阅读
	- [国外服务器][gitbook]：gitbook提供的托管，服务器在国外，速度比较慢，偶尔被墙，HTTPS
	- [国内服务器][qcloud]：腾讯云加速，国内网速极快，非HTTPS
- [下载pdf格式][istio-pdf]
- [下载mobi格式][istio-mobi]
- [下载epub格式][istio-epub]

本文内容可以任意转载，但是需要注明来源并提供链接。

**请勿用于商业出版**。

[servicemesh]: https://mp.weixin.qq.com/s?__biz=MzA3MDg4Nzc2NQ==&mid=2652136204&idx=2&sn=a7d438896f7d376181a8ccfa74faba59&chksm=84d53336b3a2ba20394cce3f9b143df36d56585e28c274989deb1f48a4c591db94643b6a4013&mpshare=1&scene=1&srcid=1018tYegaSIkQ2ImbXp4L5mS&pass_ticket=F8CjNuTDg%2Fskt94bwJ%2B1yiPKpHJhaaRYpxDCqtNGMrMGkGsZDLF5EW1HCByba35u#rd
[istio]: https://mp.weixin.qq.com/s?__biz=MzA3MDg4Nzc2NQ==&mid=2652136078&idx=1&sn=b261631ffe4df0638c448b0c71497021&chksm=84d532b4b3a2bba2c1ed22a62f4845eb9b6f70f92ad9506036200f84220d9af2e28639a22045&mpshare=1&scene=1&srcid=0922JYb4MpqpQCauaT9B4Xrx&pass_ticket=F8CjNuTDg%2Fskt94bwJ%2B1yiPKpHJhaaRYpxDCqtNGMrMGkGsZDLF5EW1HCByba35u#rd
[gitbook]: https://doczhcn.gitbooks.io/istio/
[qcloud]: http://istio.doczh.cn/
[istio-pdf]: https://www.gitbook.com/download/pdf/book/doczhcn/istio
[istio-mobi]: https://www.gitbook.com/download/mobi/book/doczhcn/istio
[istio-epub]: https://www.gitbook.com/download/epub/book/doczhcn/istio

## 进度

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

## 翻译说明

目前的文档翻译工作由Service Mesh技术社区在主持，如果有朋友有意愿加入我们的翻译工作，可以有如下方式贡献您的力量：

1. 审核和校对：如果您在阅读时发现内容有谬误，可以通过 [github issue](https://github.com/doczhcn/istio/issues) 提交issue给我们，您也可以fork出来然后通过PR提交更改
2. 直接参与翻译工作： 您可以报名加入我们的翻译小组，请先加入Service Mesh技术社区，然后在微信群中联系我们。

