# 智能路由

这一指南演示了Istio Service Mesh的通信管理能力。

## 概述

把微服务应用部署到Istio Service Mesh集群上，就可以在外部控制服务的监控、跟踪、（版本相关的）请求路由、弹性测试、安全和策略增强等，并且可以跨越服务界限，从整个应用的层面进行管理。

本文章将会使用[Bookinfo示例应用](bookinfo.html)来进行展示，看一个运维人员如何为一个运行中的应用动态的配置请求路由以及错误注入。

### 开始之前

- 依据[安装指南](../setup/)安装Istio控制平面。
- 按照[应用部署指南](bookinfo.html#deploying-the-application)运行BookInfo示例应用。

## 任务

1. [请求路由](../tasks/traffic-management/request-routing.html)：这一任务会首先把所有进入Bookinfo的访问请求指向`review`服务的v1版本。然后会在不影响其他用户的情况下，把特定测试用户的请求发送到`review`服务的v2版本去。
2. [错误注入](../tasks/traffic-management/fault-injection.html)：我们接下来会在`reviews:v2`和`ratings`服务之间假造一个延时，以此来测试Bookinfo应用的弹性。观察测试用户的行为，我们会注意到v2版本的`reviews`服务有个bug。注意所有其他用户对这一在线系统的测试都是毫不知情的。
3. [流量转移](../tasks/traffic-management/traffic-shifting.html)：最后我们修复了v2版本`reviews`服务的问题，使用Istio优雅的把所有用户的流量转移到v3版本的`reviews`上。

## 清理

Bookinfo试验结束之后，可以根据[Bookinfo清理指南](bookinfo.html#cleanup)的介绍来清理测试环境。
