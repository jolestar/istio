# Mixer Aspect 配置

说明如何配置 Mixer **切面** 及其依赖项。

## 概述

Mixer配置通过指定三个关键信息来表达系统行为：采取**什么**行动，**如何**采取行动以及**何时**采取行动。

* **采取什么行动:** [_Aspect_](./mixer-config.md#切面) 配置定义要采取**什么**行动。这些行动包括日志，metrics收集，名单检查，配额执行等。[描述符](./mixer-config.md#描述符) 是切面配置的命名和可重用的部分。例如，`metrics` 切面定义MetricDescriptor，并按名称引用MetricDescriptor实例。

* **如何采取行动:** 适配器配置定义**如何**采取行动。metrics适配器配置包括基础设施后端的详细信息。

* **何时采取行动:** 选择器和`subjects`定义**何时**采取行动。选择器是基于属性的表达式，如`response.code == 200`，Subject是分层资源名称，如`myservice.namespace.svc.cluster.local`。

## 配置步骤

请考虑以下启用限速的切面配置。

```yaml
- aspects:
  - kind: quotas
    params:
      quotas:
      - descriptorName: RequestCount
        maxAmount: 5
        expiration: 1s
        labels:
          label1: target.service
```

它使用 `RequestCount` 来描述配额。以下是 `RequestCount` 描述符的例子。

```yaml
name: RequestCount
rate_limit: true
labels:
   label1: 1 # STRING
```

在此示例中，`rate_limit` 为 `true`，因此该 `aspect` 必须指定 `expiration`。 类似地，该 `aspect` 必须提供一个`string`类型的标签。

Mixer 将使用限速的工作代理给实现 `quotas` 类型的 `adapter`。 [adapters.yml](https://github.com/istio/mixer/blob/master/testdata/configroot/scopes/global/adapters.yml) 定义这个配置。

```yaml
- name: default
  kind: quotas
  impl: memQuota
  params:
    minDeduplicationDuration: 2s
```

上述示例中的 `memQuota` 适配器需要一个参数。运维人员可以通过指定备用 `quotas` 适配器从 `memQuota` 切换到 `redisQuota`。

```yaml
- name: default
  kind: quotas
  impl: redisQuota
  params:
    redisServerUrl: redisHost:6379
    minDeduplicationDuration: 2s
```

以下示例显示了如何使用 [选择器](./mixer-config.md#选择器) 选择性地应用限速。

```yaml
- selector: source.labels["app"]=="reviews" && source.labels["version"] == "v3"
  aspects:
  - kind: quotas
    params:
      quotas:
      - descriptorName: RequestCount
        maxAmount: 5
        expiration: 1s
        labels:
          label1: target.service
```

## 切面组合

上一节中概述的步骤适用于Mixer的所有切面。每个切面都需要特定的`描述符`和`适配器`。 下表列举了`切面`，`描述符`和`适配器`的有效组合。

|切面   |描述符               |适配器
|-----------------------------------------------
|[Quota enforcement]({{book.aspectConfig}}/quotas.md) | [QuotaDescriptor]({{book.mixerConfig}}#istio.mixer.v1.config.descriptor.QuotaDescriptor) |  [memQuota]({{book.adapterConfig}}/memQuota.md), [redisQuota]({{book.adapterConfig}}/redisquota.md)
|[Metrics collection]({{book.aspectConfig}}/metrics.md)| [MetricDescriptor]({{book.mixerConfig}}#metricdescriptor) |[prometheus]({{book.adapterConfig}}/prometheus.md),[statsd]({{book.adapterConfig}}/statsd.md)
|[Whitelist/Blacklist]({{book.aspectConfig}}/lists.md)| None |[genericListChecker]({{book.adapterConfig}}/genericListChecker.md),[ipListChecker]({{book.adapterConfig}}/ipListChecker.md)
|[Access logs]({{book.aspectConfig}}/accessLogs.md)|[LogEntryDescriptor]({{book.mixerConfig}}#logentrydescriptor)  |[stdioLogger]({{book.adapterConfig}}/stdioLogger.md)
|[Application logs]({{book.aspectConfig}}/applicationLogs.md)|[LogEntryDescriptor]({{book.mixerConfig}}#logentrydescriptor)  |[stdioLogger]({{book.adapterConfig}}/stdioLogger.md)
|[Deny Request]({{book.aspectConfig}}/denials.md)| None |[denyChecker]({{book.adapterConfig}}/denyChecker.md)

Istio使用 [`protobufs`](https://developers.google.com/protocol-buffers/) 来定义配置模式。 [编写配置](../../reference/writing-config.md) 文档解释了如何将  `proto`  定义表达为 `yaml`。

## 配置的组织

切面配置适用到 `subject`。 `subject` 是层次结构中的资源。 通常 `subject` 是服务，命名空间或集群的完全限定名称。切面配置可以应用于 `subject` 资源及其子资源。

## 推送配置

`istioctl` 将配置更改推送到API服务器。从Alpha版本开始，API服务器支持仅推送切面规则。

临时解决方法允许您按如下所示推送 `adapters.yml` 和 `descriptors.yml`。

1. 找到 Mixer pod

   ```bash
   kubectl get pods -l istio=mixer
   ```

   输出类似于：

   ```bash
   NAME                           READY     STATUS    RESTARTS   AGE
   istio-mixer-2657627433-3r0nn   1/1       Running   0          2d
   ```

2. 从 Mixer 中获取adapters.yml

   ``` bash
   kubectl cp istio-mixer-2657627433-3r0nn:/etc/opt/mixer/configroot/scopes/global/adapters.yml  adapters.yml
   ```

3. 编辑文件并推送回去

   ```bash
   kubectl cp adapters.yml istio-mixer-2657627433-3r0nn:/etc/opt/mixer/configroot/scopes/global/adapters.yml
   ```

4. 同样更新 `/etc/opt/mixer/configroot/scopes/global/descriptors.yml`

5. 查看Mixer日志以检查验证错误，因为上述操作绕过了API服务器。

## 默认配置

Mixer 为常用的 [描述符](https://github.com/istio/mixer/blob/master/testdata/configroot/scopes/global/descriptors.yml) 和 [适配器](https://github.com/istio/mixer/blob/master/testdata/configroot/scopes/global/adapters.yml) 提供默认定义。

## 下一步

* 了解有关 [Mixer](./mixer.md) 和 [Mixer 配置](./mixer-config.md) 的更多信息。

* 发现完整的 [属性词汇](../../reference/config/mixer/attribute-vocabulary.md)。
