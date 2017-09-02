# 认证

## 概述

Istio Auth的目标是提高微服务及其通信的安全性，而不需要修改服务代码。它负责：

* 为每个服务提供强大的身份，代表其角色，以实现跨集群和云的互通性

* 加密服务到服务的通讯

* 提供密钥管理系统来自动执行密钥和证书的生成，分发，轮换和撤销

在将来的版本中，它还将提供：

* 加密终端用户到服务的通信

* 细粒度的授权和审核,来控制和监控访问您服务，api或资源的人员

* 多重授权机制：[ABAC](https://en.wikipedia.org/wiki/Attribute-Based_Access_Control), [RBAC](https://en.wikipedia.org/wiki/Role-based_access_control), 授权钩子

## 架构

下图展示 Istio Auth 架构，其中包括三个组件：身份，密钥管理和通信安全。它描述了Istio Auth如何用于加密服务A(作为服务帐户“foo”运行)和服务B(作为服务帐户“bar”运行)之间的服务到服务通信。

![](./img/auth/auth.svg)

## 组件

### 身分

在 Kubernetes 上运行时，由于以下原因，Istio Auth使用 [Kubernetes服务帐户](../../tasks/configure-pod-container/configure-service-account/) 来识别运行该服务的人员：

* 服务帐户是**工作负载运行的身份（或角色）**，表示该工作负载的权限。对于需要强大安全性的系统，工作负载的特权量不应由随机字符串（如服务名称，标签等）或部署的二进制文件来标识。

	* 例如，假设我们有一个从多租户数据库中提取数据的工作负载。如果Alice运行这个工作负载，她将能够提取一组和Bob运行这个工作负载不同的数据。

* 服务帐户通过提供灵活性来识别机器，用户，工作负载或一组工作负载（不同的工作负载可以以同一服务帐户运行）来实现强大的安全策略。

* 工作负载运行的服务帐户将不会在工作负载的生命周期内更改。

* 可以通过域名约束确保服务帐户的唯一性

### 通讯安全

服务到服务通信通过客户端Envoy和服务器端Envoy进行隧道传送。端到端通信通过一下方式加密：

* 服务与Envoy之间的本地TCP连接

* 代理之间的相互TLS连接

* 安全命名：在握手过程中，客户端Envoy检查服务器端证书提供的服务帐号是否允许运行目标服务

### 密钥管理

Istio Auth 提供每群集CA（证书颁发机构）来自动化密钥和证书管理。它执行四个关键操作：

* 为每个服务帐户生成一个 [SPIFFE][] 密钥和证书对

* 根据服务帐户将密钥和证书对分发给每个pod

* 定期轮换密钥和证书

* 必要时撤销特定的密钥和证书对

## 工作流

Istio Auth工作流由两个阶段组成，部署和运行时。这部分涵盖他们两个。

### 部署阶段

1. Istio CA 观察 Kubernetes API Server，为每个现有和新的服务帐户创建一个 [SPIFFE][] 密钥和证书对，并将其发送到API服务器。

2. 当创建pod时，API Server会根据服务帐户使用 [Kubernetes secrets][] 来挂载密钥和证书对。

3. [Pilot](../traffic-management/pilot.md) 使用适当的密钥和证书以及安全命名信息生成配置，该信息定义了什么服务帐户可以运行某个服务，并将其传递给Envoy。

### 运行时阶段

1. 来自客户端服务的出站流量被重新路由到它本地的Envoy。

2. 客户端Envoy与服务器端Envoy开始相互TLS握手。在握手期间，它还进行安全的命名检查，以验证服务器证书中显示的服务帐户是否可以运行服务器服务。

3. mTLS连接建立后，流量将转发到服务器端Envoy，然后通过本地TCP连接转发到服务器服务。

## 最佳实践

在本节中，我们提供了一些部署指南，然后讨论了一个现实世界的场景。

### 部署指南

* 如果有多个服务运维人员（也称为[SRE]）在集群中部署不同的服务（通常在中型或大型集群中），我们建议为每个SRE团队创建一个单独的[namespace]，以隔离其访问。例如，您可以为team1创建一个"team1-ns"命名空间，为team2创建"team2-ns"命名空间，这样两个团队就无法访问对方的服务。

* 如果Istio CA受到威胁，则可能会暴露集群中被它管理的所有密钥和证书。我们强烈建议在专门的命名空间（例如istio-ca-ns）上运行Istio CA，只有集群管理员才能访问它。

### 示例

我们考虑一个三层应用程序，其中有三个服务：照片前端，照片后端和数据存储。照片前端和照片后端服务由照片SRE团队管理，而数据存储服务由数据存储SRE团队管理。照片前端可以访问照片后端，照片后端可以访问数据存储。但是，照片前端无法访问数据存储。

在这种情况下，集群管理员创建3个命名空间：istio-ca-ns，photo-ns和datastore-ns。管理员可以访问所有命名空间，每个团队只能访问自己的命名空间。照片SRE团队创建了2个服务帐户，以分别在命名空间photo-ns中运行照片前端和照片后端。数据存储SRE团队创建1个服务帐户以在命名空间datastore-ns中运行数据存储服务。此外，我们需要在 [Istio Mixer](../policy-and-control/mixer.md) 中强制执行服务访问控制，以使照片前端无法访问数据存储。

在此设置中，Istio CA能够为所有命名空间提供密钥和证书管理，并隔离彼此的微服务部署。

## 未来的工作

* 细粒度授权和审核

* 安全Istio组件（Mixer, Pilot等）

* 集群间服务到服务认证

* 使用 JWT/OAuth2/OpenID_Connect 终端到服务的认证

* 支持GCP服务帐户和AWS服务帐户

* 非http流量（MySql，Redis等）支持

* Unix域套接字，用于服务和Envoy之间的本地通信

* 中间代理支持

* 可插拔密钥管理组件

[SPIFFE]:https://spiffe.github.io/docs/svid
[Kubernetes secrets]:https://kubernetes.io/docs/concepts/configuration/secret/
[SRE]:https://en.wikipedia.org/wiki/Site_reliability_engineering
[namespace]:https://kubernetes.io/docs/tasks/administer-cluster/namespaces-walkthrough/