# 深入遥测

这个例子演示了如何使用Istio Mixer和Istio Sidecar，从多个服务中获取一致的指标、日志、跟踪信息。

## 概述

把微服务应用部署到Istio Service Mesh集群上，就可以在外部控制服务的监控、跟踪、（版本相关的）请求路由、弹性测试、安全和策略>增强等，并且可以跨越服务界限，从整个应用的层面进行管理。

本文将会使用[Bookinfo示例应用](bookinfo.html)，展示在无需开发改动代码的情况下，运维人员如何从运行中的多语言平台应用获取一致的指标和跟踪信息。

## 开始之前

- 依据[安装指南](../setup/)安装Istio控制平面。
- 按照[应用部署指南](bookinfo.html#deploying-the-application)运行BookInfo示例应用。

## 任务

1. [指标收集](../tasks/telemetry/metrics-logs.html)：该任务会对Mixer进行配置，用统一方式从Bookinfo应用的所有服务中收集一套指标。
2. [指标查询](../tasks/telemetry/querying-metrics.html)：这里安装并配置了Prometheus，用于Istio指标的收集、查询和展示。
3. [分布式跟踪](../tasks/telemetry/distributed-tracing.html)：现在我们要使用Istio来跟踪请求在一个应用的不同服务中的流向。分布式跟踪让开发者能够快速的理解最终用户所感受的延时在各个不同服务之间的分布，这一功能对分布式应用的检测和排错工作也是很有价值的。
4. [使用Istio仪表盘](../tasks/telemetry/using-istio-dashboard.html)：这个任务安装了Grafana扩展，其中带有一个用于监控Mesh流量的预定义仪表盘。

## 清理

Bookinfo试验结束之后，可以根据[Bookinfo清理指南](bookinfo.html#cleanup)的介绍来清理测试环境。
