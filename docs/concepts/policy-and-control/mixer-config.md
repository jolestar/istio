# Mixer 配置

本节介绍 Mixer 的配置模型。

## 背景

Istio是一个具有数百个独立功能的复杂系统。Istio部署可能涉及数十个服务的蔓延事件，这些服务有一群Envoy代理和Mixer实例来支持它们。在大型部署中，许多不同的运维人员（每个运维人员都有不同的范围和责任范围）可能涉及管理整体部署。

Mixer的配置模式可以利用其所有功能和灵活性，同时保持使用的相对简单。该模型的范围特征使大型支持组织能够轻松地集中管理复杂的部署。该模型的一些主要功能包括：

- **专为运维人员而设计**。服务运维人员通过操纵配置记录来控制Mixer部署中的所有操作和策略切面。

- **范围**。配置被分层描述，可以实现粗略的全局控制以及细粒度的本地控制。

- **灵活**。配置模型围绕Istio的 [属性](./attributes.md) 构建，使运维人员能够对部署中使用的策略和生成的遥测进行前所未有的控制。

- **健壮**。配置模型旨在提供最大的静态正确性保证，以帮助减少导致服务中断的错误配置更改的可能性。

- **扩展**。该模型旨在支持Istio的整体可扩展性思路。可以将新的或自定义的 [适配器](./mixer.md#适配器) 添加到Istio中，并可以使用与现有适配器相同的通用机制进行完全操作。

## 概念

Mixer是一种属性处理机器。请求到达Mixer时带有一组 [属性](./attributes.md) ，并且基于这些属性，Mixer会生成对各种基础设施后端的调用。该属性集确定Mixer为给定的请求调用哪个后端以及每个给出哪些参数。为了隐藏各个后端的细节，Mixer使用称为[适配器](./mixer.md#适配器)的模块。

<img src="./img/mixer-config/machine.svg" alt="Attribute Machine" title="Attribute Machine" />

Mixer的配置有两个中心职责：

- 描述哪些适配器正在使用以及它们的运行方式。
- 描述如何将请求属性映射到适配器参数中。

配置使用YAML格式来表示,围绕五个核心抽象构建：

|概念                     |描述|
|----------------------------|-----------|
|[适配器](#适配器)       | 适用于各个Mixer适配器的低级别的关注于操作的配置。|
|[切面](#切面)         | 适用于各个Mixer适配器的高级别的关注于意图的配置。|
|[描述符](#描述符) | 用于各个切面的参数描述。|
|[范围](#范围)           | 根据请求的属性选择要使用哪些切面和描述符的机制。|
|[清单](#清单)     | Istio部署的诸多静态特性的描述。|

以下部分将详细介绍这些概念。

### 适配器

[适配器](#适配器)是基础工作单位,Istio Mix围绕它而构建。适配器封装了将Mixer与特定的外部基础设施后端（如[Prometheus](https://prometheus.io)，[New Relic](https://newrelic.com)或[Stackdriver](https://cloud.google.com/logging)）接入所必需的逻辑。单个适配器通常需要提供一些基本的操作参数才能完成他们的工作。例如，日志适配器可能需要知道应该将日志数据吐到哪个IP地址和端口。

Mixer可以使用一套适配器，每个都需要单独的配置参数。以下是一个示例，说明如何配置适配器：

```yaml
adapters:
  - name: myListChecker     # 这个配置块的用户定义的名称
    kind: lists             # 这个适配器可以使用的切面类型
    impl: ipListChecker     # 要使用的特定适配器组件的名称
    params:
      publisherUrl: https://mylistserver:912
      refreshInterval: 60s
```

该 `name` 字段为适配器配置块提供了一个名称，因此可以从别处引用它。该 `kind` 字段指示此配置适用的[切面类型](#切面)。该 `impl` 字段给出正在配置的适配器的名称。最后，该  `params` 部分是指定实际的适配器特定配置参数的位置。在这个案例中，这里配置的是适配器在其查询中应使用的URL，并定义刷新其本地缓存的时间间隔。

对于每个可用的适配器实现，您可以定义任意数量的独立配置块。这允许在单个部署中多次使用相同的适配器。根据具体情况，例如涉及哪个服务，将使用某一个配置块而不是其他。例如，这里有两个可以与前一个共存的配置块：

```yaml
adapters:
  - name: mySecondaryListChecker
    kind: lists
    impl: ipListChecker
    params:
      publisherUrl: https://mysecondlistserver:912
      refreshInterval: 3600s
  - name: myTernaryListChecker
    kind: lists
    impl: genericListChecker
    params:
      listEntries:
        "400"
        "401"
        "402"
```

还有一个：

```yaml
adapters:
  - name: myMetricsCollector
    kind: metrics
    impl: prometheus
```

这将配置适配器,将数据报告给Prometheus系统。此适配器不需要任何自定义参数，因此没有 `params` 节。

每个适配器定义其自己的特定格式的配置数据。适配器的详尽集及其特定的配置格式可以在[这里](../../reference/config/mixer/adapters/) 找到。

### 切面

切面定义高级别配置（有时称为基于意图的配置），独立于特定适配器类型的特定实现细节。而适配器专注于**如何(how)**做某些事情，切面侧重于要做**什么(what)**。

我们来看一个切面的定义：

```yaml
aspects:
- kind: lists               # 切面的类型
  adapter: myListChecker    # 实现这个切面的适配器
  params:
    blacklist: true
    checkExpression: source.ip
```

该 `kind` 字段区分定义的切面的行为。支持的切面如下表所示。

|类型             |描述|
|-----------------|-----------|
|quotas           |执行配额和限速。|
|metrics          |生成metric|
|lists            |执行基于白名单或黑名单的访问控制。|
|access-logs      |为每个请求生成固定格式的访问日志。|
|application-logs |为每个请求生成灵活的应用程序日志。|
|attributes       |为每个请求生成补充属性。|
|denials          |按步就班地产生可预测的错误代码。|

在上面的示例中，切面声明指定了 `lists` 类型,表示我们正在配置一个切面，其目的是使用白名单或黑名单作为访问控制的一种简单形式。

该 `adapter` 字段指示与此切面关联的适配器配置块。切面总是以这种方式与特定适配器相关联，因为适配器负责实际执行由切面配置表示的工作。在这种具体案例中，所选择的特定适配器确定要使用的名单，以执行方面的名单检查功能。

通过将切面配置与适配器配置分开，可以轻松地更改用于实现特定切面行为的适配器，而无需更改切面本身。另外，许多切面都可以引用相同的适配器配置。

`params` 节是您输入特定种类的配置参数的地方。在这个 `lists` 类型的案例中，配置参数指定名单是否是黑名单（名单中的条目导致拒绝），而不是白名单（不在列表中的条目导致拒绝）。`checkExpression` 字段指示在请求时使用的属性, 用来获取符号以相关适配器的名单进行检查.

这是另一个切面，这次是一个 `metrics` 切面：

```yaml
aspects:
- kind: metrics
  adapter: myMetricsCollector
  params:
    metrics:
    - descriptorName: request_count
      value: "1"
      labels:
        source: source.name
        target: target.name
        service: api.name
        responseCode: response.code
```

这定义了一个切面，它产生了metrics, metrics被发送到前面定义的myMetricsCollector适配器。该 `metrics` 节定义了在请求处理期间为这个切面生成的metrics集合。该 `descriptorName` 字段指定描述符的名称，该描述符是单独的配置块，[如下所述](#descriptors)，声明了这种metrics的类型。该 `value` 字段和四个标签字段描述哪些属性在请求时使用以产生metrics。

每个切面类型都定义了自己的配置数据格式。[这里](../../reference/config/mixer/aspects/) 可以找到详尽的切面配置格式。

#### 属性表达式

Mixer 具有多个独立的 [请求处理阶段](./mixer.md#请求阶段) 。**属性处理阶段**负责摄取一组属性，并产生调用各个适配器时需要的适配器参数。该阶段通过评估一系列**属性表达式**来运行。

在前面的例子中，我们已经看到了一些简单的属性表达式。特别是：

```yaml
  source: source.name
  target: target.name
  service: api.name
  responseCode: response.code
```

冒号右侧的序列是属性表达式的最简单形式。它们只包括属性名称。在上面，`source` 标签将被赋予 `source.name` 属性的值。以下是条件表达式的示例：

```yaml
  service: api.name | target.name
```

使用上述方法，服务标签将被赋予 `api.name` 属性的值，或者如果没有定义该属性，它将被赋予 `target.name` 属性的值。

可以在属性表达式中使用的属性必须在部署的 [**属性清单**](#清单) 中定义。在清单中，每个属性都有类型,表示该属性所携带的数据类型。同样，属性表达式也有类型，它们的类型是从表达式中的属性和应用于这些属性的运算符派生出来的。

属性表达式的类型用于确保在什么情况下使用哪些属性的一致性。例如，如果metric描述符指定的特定标签类型为 INT64，则只能使用产生64位整数的属性表达式来填充该标签。上述 `responseCode` 标签就是这种情况。

有关详细信息，请参阅 [属性表达式引用](../../reference/config/mixer/expression-language.md)。

#### 选择器

选择器是应用于某切面的注释，以确定该切面是否适用于任何给定的请求。选择器使用产生布尔值的属性表达式。如果表达式返回 `true` 则相关切面将被应用。否则会被忽略，没有任何效果。

让我们添加一个选择器到上一个切面例子：

```yaml
aspects:
- selector: target.service == "MyService"
  kind: metrics
  adapter: myMetricsCollector
  params:
    metrics:
    - descriptorName: request_count
      value: "1"
      labels:
        source: source.name
        target: target.name
        service: api.name
        responseCode: response.code
```

 `selector` 上面的字段定义了一个表达式，如果 `target.service` 属性等于"MyService"就返回`true` 。如果表达式返回 `true`，则切面定义对给定的请求生效，否则就像切面未定义一样。

### 描述符

描述符用于准备Mixer，其适配器及其基础设施后端以接收特定类型的数据。例如，声明一组metrics描述符告诉Mixer不同metrics将携带的数据类型，以及用于标识这些度量的不同实例的标签集。

有不同类型的描述符，每种都与特定切面类型相关联：

|描述符类型     |切面类型     |描述
|--------------------|----------------|-----------
|Metric Descriptor   |metrics         |描述单个metric是什么样的。
|Log Entry Descriptor|application-logs|描述单个日志条目是什么样的。
|Quota Descriptor    |quotas          |描述单个配额是什么样的。

这是一个示例metric描述符：

```yaml
metrics:
  - name: request_count
    kind: COUNTER
    value: INT64
    displayName: "Request Count"
    description: Request count by source, target, service, and code
    labels:
      source: STRING
      target: STRING
      service: STRING
      responseCode: INT64
```

以上是声明系统可以产生名为 `request_count` 的metrics。这样的metrics将持有64位整数值并作为绝对计数器进行管理。报告的每个metric将有四个标签，两个指定源和目标名称，一个是服务名称，另一个是请求的响应代码。对于此描述符，Mixer可以确保生成的度量标准总是正确组成的，可以安排这些metrics的高效存储，并且可以确保基础设施后端可以接受这些metrics。这些 `displayName` 和 `description` 字段是可选的，并且传达给基础设施后端，后端可以使用这些文本来增强其metric可视化界面。

明确定义描述符并使用它们来创建适配器参数类似于传统编程语言中的类型和对象。这样做可以实现以下重要场景：

- 明确定义描述符集后，Istio可以对基础设置后端进行编程，以接受Mixer生成的流量。例如，metric描述符提供所需的所有信息, 来编程基础设置后端以接受符合描述符形状（它的值类型及其标签集）。

- 描述符可以被多个切面引用和重用。

- 它使Istio能够提供强类型的脚本环境，如 [这里](../../reference/config/mixer/mixer-config.md) 所述

不同的描述符类型的细节在 [这里](../../reference/config/mixer/mixer-config.md)。

### 范围

Istio部署可以负责管理大量服务。组织通常有数十种或数百种交互式服务，而Istio的使命是使其易于管理所有服务。Mixer的配置模式旨在支持不同运维人员管理Istio部署的不同部分，而不会踩在彼此的脚趾上，同时允许他们对其区域进行控制，但不允许其他人操作。

这一切是如何工作：

- 上一节（适配器，切面和描述符）中描述的各种配置块始终在层次结构的上下文中定义。

- 层次结构由DNS风格的点号分割的名称表示。像DNS一样，层次结构从点号分割的名称中最右边的元素开始。

- 每个配置块与**范围**和**主题**相关联，这两个对象都是点号分割的名称,表示层次结构中的位置：

	- 范围代表创建配置块的权限。层次结构中的高级别的权力比较低级别的权力更强大。

	- 主题表示层次结构中状态块的位置。在层次结构内主题必须始终处于或低于范围的级别。

- 如果配置的多个块具有相同的主题，则与层次结构中最高范围相关联的块始终优先。

构成层次结构的各个元素取决于Istio部署的具体细节。Kubernetes部署可能使用Kubernetes命名空间作为部署Istio配置状态的层次结构。例如，一个有效的范围可能是 `svc.cluster.local` 而一个主题可能是 `myservice.ns.svc.cluster.local`

范围模型旨在与访问控制模型进行配对，以限制允许哪个人容许为特定范围创建配置块。具有在层次结构更高的范围内创建块的权限的操作符可能会影响与较低范围相关联的所有配置。尽管这是设计意图，但是Mixer配置还不支持配置的访问控制，因此对于哪个操作员可以操作哪个范围没有实际的约束。

#### 决议

当请求到达时，Mixer会经过多个 [请求处理阶段](./mixer.md#请求阶段)。决议阶段涉及确定要用于处理传入请求的确切配置块。例如，到达 Mixer 的 service A 的请求可能与 service B 的请求有一些配置差异。决议决定哪个配置用于请求。

决议取决于众所周知的属性来指导其选择，即所谓的身份属性。该属性的值是一个虚线名称，它决定了Mixer开始在层次结构中查找用于请求的配置块。

Resolution depends on a well-known attribute to guide its choice, a so-called *identity attribute*.
The value of this attribute is a dotted name which determines where Mixer begins to look in the
hierarchy for configuration blocks to use for the request.

Here's how it all works:

1. A request arrives and Mixer extracts the value of the identity attribute to produce the current
lookup value.

2. Mixer looks for all configuration blocks whose subject matches the lookup value.

3. If Mixer finds multiple blocks that match, it keeps only the block that has the highest scope.

4. Mixer truncates the lowest element from the lookup value's dotted name. If the lookup value is
not empty, then Mixer goes back to step 2 above.

All the blocks found in this process are combined together to form the final effective configuration that is used to
evaluate the current request.

### 清单

Manifests capture invariants about the components involved in a particular Istio deployment. The only
kind of manifest supported at the moment are *attribute manifests* which are used to define the exact
set of attributes produced by individual components. Manifests are supplied by component producers
and inserted into a deployment's configuration.

Here's part of the manifest for the Istio proxy:

```yaml
manifests:
  - name: istio-proxy
    revision: "1"
    attributes:
      source.name:
        valueType: STRING
        description: The name of the source.
      target.name:
        valueType: STRING
        description: The name of the target
      source.ip:
        valueType: IP_ADDRESS
        description: Did you know that descriptions are optional?
      origin.user:
        valueType: STRING
      request.time:
        valueType: TIMESTAMP
      request.method:
        valueType: STRING
      response.code:
        valueType: INT64
```

## Examples

You can find fully formed examples of Mixer configuration by visiting the [Samples]({{home}}/docs/samples). As
a specific example, here is the [Default configuration](https://github.com/istio/mixer/blob/master/testdata/configroot/scopes/global/subjects/global/rules.yml).
