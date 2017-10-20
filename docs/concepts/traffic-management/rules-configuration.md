# 规则配置

Istio提供了一种简单的领域特定语言（DSL）来控制应用程序部署中跨各种服务的API调用和第4层流量。DSL允许运维人员配置服务级别的属性，如熔断器，超时，重试，以及设置常见的连续部署任务，如金丝雀推出，A/B测试，基于百分比流量拆分的分阶段推出等。有关详细信息，请参阅 [路由规则参考](../../reference/config/traffic-rules/index.md)

例如，使用规则DSL来描述，将“reviews”服务的100％的传入流量发送到版本“v1”的简单规则可以使用规则DSL来如下描述：

```yaml
destination: reviews.default.svc.cluster.local
route:
- tags:
    version: v1
  weight: 100
```

destination是要路由流量的服务的名称。在Istio的Kubernetes部署中，路由*tag* "version：v1"对应于Kubernetes *label* "version：v1"。该规则确保只有包含标签"version：v1"Kubernetes pod将会收到流量。可以使用 [istioctl CLI](../../reference/commands/istioctl.md) 配置规则。有关示例，请参阅 [配置请求路由任务](../../tasks/request-routing.md)。

在Istio中有两种类型的规则，**Routes/路由 和 Destination
Policies/目的地策略**（这些与Mixer策略不同）。两种类型的规则控制请求如何路由到目标服务。

## 路由

Routes 控制请求如何路由到不同版本的服务。请求可以基于源和目标，HTTP header字段以及与个别服务版本相关联的权重进行路由。编写路由规则时，必须牢记以下重要方面：

### 通过destination限定规则

每个规则对应于规则中的 *destination* 字段标识的目的地服务。例如，适用于"reviews"服务调用的所有规则将包括下面字段。

```yaml
destination: reviews.default.svc.cluster.local
```

*destination* 的值应该是一个完全限定域名（Fully Qualified Domain Name,FQDN）。Pilot用它来给服务匹配规则。例如，在Kubernetes中，服务的完全限定域名可以使用以下格式构建：*serviceName.namespace.dnsSuffix*。

### 通过source/headers限定规则

规则可以选择仅限于仅适用于符合以下特定条件的请求：

1. *限制为特定的调用者*

	例如，以下规则仅适用于来自"reviews"服务的调用。

    ```yaml
    destination: ratings.default.svc.cluster.local
    match:
      source: reviews.default.svc.cluster.local
    ```

	*source* 的值，和 *destination* 一样，必须是一个服务的FQDN。

2. *限制为调用者的特定版本*

	例如，以下规则将细化上一个示例，仅适用于"reviews"服务的"v2"版本的调用。

    ```yaml
    destination: ratings.default.svc.cluster.local
    match:
      source: reviews.default.svc.cluster.local
      sourceTags:
        version: v2
    ```

3. *选择基于HTTP header的规则*

	例如，以下规则仅适用于传入请求，如果它包含"cookie" header, 并且内容包含"user=jason"。

    ```yaml
    destination: reviews.default.svc.cluster.local
    match:
      httpHeaders:
        cookie:
          regex: "^(.*?;)?(user=jason)(;.*)?$"
    ```

如果提供了多个属性值对，则所有相应的 header 必须与要应用的规则相匹配。

可以同时设置多个标准。在这种情况下，AND语义适用。例如，以下规则仅适用于请求的source为"reviews:v2"，并且存在包含"user=jason"的"cookie" header。

```yaml
destination: ratings.default.svc.cluster.local
match:
  source: reviews.default.svc.cluster.local
  sourceTags:
    version: v2
  httpHeaders:
    cookie:
      regex: "^(.*?;)?(user=jason)(;.*)?$"
```

### 在服务版本之间拆分流量

当规则被激活时，每个 *路由规则* 标识一个或多个要调用的加权后端。每个后端对应于目标服务的特定版本，其中版本可以使用 _tags_ 表示。

如果有多个具有指定tag的注册实例，则将根据为该服务配置的负载均衡策略来路由，或默认轮循。

例如，以下规则会将"reviews"服务的25％的流量路由到具有"v2"标签的实例，其余流量（即75％）转发到"v1"。

```yaml
destination: reviews.default.svc.cluster.local
route:
- tags:
    version: v2
  weight: 25
- tags:
    version: v1
  weight: 75
```

### 超时和重试

缺省情况下，http请求的超时时间为15秒，但可以在路由规则中覆盖，如下所示：

```yaml
destination: "ratings.default.svc.cluster.local"
route:
- tags:
    version: v1
httpReqTimeout:
  simpleTimeout:
    timeout: 10s
```

最大尝试次数，或者在默认或被覆盖的超时时间内的尽可能多，可以设置如下：

```yaml
destination: "ratings.default.svc.cluster.local"
route:
- tags:
    version: v1
httpReqRetries:
  simpleRetry:
    attempts: 3
```

请注意，请求超时和重试也可以 [根据每个请求重写](./handling-failures.md#微调)。

请参阅 [请求超时任务](../../tasks/request-timeouts.md) 以演示超时控制。

### 在请求路径中注入故障

在将http请求转发到规则的相应请求目的地时，路由规则可以指定一个或多个要注入的故障。故障可能是延迟或中断。

以下示例将在"reviews"微服务的"v1"版本的10％的请求中引入5秒的延迟。

```yaml
destination: reviews.default.svc.cluster.local
route:
- tags:
    version: v1
httpFault:
  delay:
    percent: 10
    fixedDelay: 5s
```

另一种故障，中断，可以用来提前终止请求，例如模拟故障。

以下示例将为"ratings"服务"v1"的10％请求返回HTTP 400错误代码。

```yaml
destination: "ratings.default.svc.cluster.local"
route:
- tags:
    version: v1
httpFault:
  abort:
    percent: 10
    httpStatus: 400
```

有时延迟和中止故障会一起使用。例如，以下规则将所有从"reviews"服务"v2"到"ratings"服务"v1"的请求延迟5秒钟，然后中止其中的10％：

```yaml
destination: ratings.default.svc.cluster.local
match:
  source: reviews.default.svc.cluster.local
  sourceTags:
    version: v2
route:
- tags:
    version: v1
httpFault:
  delay:
    fixedDelay: 5s
  abort:
    percent: 10
    httpStatus: 400
```

要查看故障注入的实际使用，请参阅 [故障注入任务](../../tasks/fault-injection.md)。

### 规则优先级

多个路由规则可以应用于同一目的地。当有多个规则时，可以通过设置规则的*precedence*字段来指定与给定目的地相对应的规则的评估顺序。

```yaml
destination: reviews.default.svc.cluster.local
precedence: 1
```

precedence字段是可选的整数值，默认为0。首先评估具有较高优先级值的规则。_如果有多个具有相同优先级值的规则，则评估顺序是未定义的_。

**优先级什么时候有用？** 只要特定服务的路由故障纯粹是基于权重的，可以在单个规则中指定，如前面的示例所示。另一方面，当正在使用的其他标准（例如，来自特定用户的请求）来路由流量时，将需要多于一个的规则来指定路由。这是必须设置规则*precedence*字段的时候，以确保以正确的顺序对规则进行评估。

广义的路由规范的通用模式是提供一个或多个较高优先级的规则，该规则通过source/header到特定destination来限定规则，然后提供单个基于权重的规则，连最低优先级的匹配准则都不具备，以提供所有其他情况的流量。

例如，以下2条规则一起指定包含名为"Foo"值为"bar"的header的"reviews"服务的所有请求都将发送到"v2"实例。所有剩余的请求将被发送到"v1"。

```yaml
destination: reviews.default.svc.cluster.local
precedence: 2
match:
  httpHeaders:
    Foo:
      exact: bar
route:
- tags:
    version: v2
---
destination: reviews.default.svc.cluster.local
precedence: 1
route:
- tags:
    version: v1
  weight: 100
```

请注意，基于header的规则具有较高的优先级（2对1）。如果它较低，这些规则将无法正常工作，因为基于权重的规则（没有特定匹配条件）将首先被评估，然后将简单地将所有流量路由到“v1”，即使是包括匹配“Foo” header的请求。一旦找到适用于传入请求的规则，它将被执行，并且规则评估过程将终止。这就是为什么当有不止一个规则时，仔细考虑每个规则的优先级是非常重要的。

## 目的地策略

目的地策略描述与特定服务版本相关联的各种路由相关策略，例如负载均衡算法，熔断器配置，健康检查等。与路由规则不同，目的地策略不能根据请求的属性进行限定，例如正在调用的服务或HTTP请求header。

但是，可以限制策略适用于使用特定标签路由到后端的请求。例如，以下负载均衡策略仅适用于针对"reviews"微服务器的"v1"版本的请求。

```yaml
destination: reviews.default.svc.cluster.local
policy:
- tags:
    version: v1
  loadBalancing:
    name: RANDOM
```

### 熔断器

可以根据诸如连接和请求限制的多个标准来设置简单的熔断器。

例如，以下目的地策略设置到"reviews"服务版本"v1"后端的100个连接的限制。

```yaml
destination: reviews.default.svc.cluster.local
policy:
- tags:
    version: v1
  circuitBreaker:
    simpleCb:
      maxConnections: 100
```

[这里](../../reference/config/traffic-rules/destination-policies.md#istio.proxy.v1.config.CircuitBreaker)可以找到一整套简单的熔断器字段。

### 目的地策略评估

类似于路由规则，目的地策略与特定*目的地*相关联，但是如果它们还包括*标签*，则其激活取决于路由规则评估结果。

规则评估过程的第一步将评估目的地的路由规则（如果有定义），以确定当前请求将路由到的目标服务的标签（例如，特定版本）。下一步，评估目的地策略集,如果有，以确定它们是否适用。

**注意**：算法要注意的一个微妙之处在于，仅当对应的已标记实例被明确路由时，才会应用为特定标记目标定义的策略。例如，考虑以下规则，作为"reviews"服务的唯一规则。

```yaml
destination: reviews.default.svc.cluster.local
policy:
- tags:
    version: v1
  circuitBreaker:
    simpleCb:
      maxConnections: 100
```

由于没有为"reviews"服务定义特定的路由规则，因此默认轮循路由行为将适用，这有可能偶尔调用“v1”实例，如果“v1”是唯一运行的版本甚至会一致调用。然而，上述策略将永远不会被调用，因为默认路由在较低级别完成。规则评估引擎将不知道最终目的地，因此无法将目标策略与请求相匹配。

您可以通过以下两种方式之一修复上述示例。您可以从规则中删除`tags:`，如果“v1”是唯一的实例，或者更好地，为服务定义适当的路由规则。例如，您可以为"reviews:v1"添加一个简单的路由规则。

```yaml
destination: reviews.default.svc.cluster.local
route:
- tags:
    version: v1
```

虽然默认的Istio行为可以很方便地将流量从源服务的所有版本发送到目标服务的所有版本，而不用设置任何规则，一旦需要版本区别，将需要规则。因此，从一开始就为每个服务设置默认规则通常被认为是Istio的最佳实践。

### 出站规则

出站规则用于配置需要在服务网格中被调用的外部服务。例如下面的规则可以被用来配置在*.foo.com domain域名下的外部服务。

```yaml
apiVersion: config.istio.io/v1alpha2
kind: EgressRule
metadata:
  name: foo-egress-rule
spec:
  destination:
    service: *.foo.com
  ports:
    - port: 80
      protocol: http
    - port: 443
      protocol: https
```

外部服务的地址通过*service*字段指定,该字段可以是一个完全限定域名（Fully Qualified Domain Name,FQDN），也可以是一个带通配符的域名。该域名代表了可在服务网格中访问的外部服务的一个白名单，该白名单中包括一个(字段值为完全限定域名的情况)或多个外部服务(字段值为带通配符的域名的情况)。[这里](https://istio.io/docs/reference/config/traffic-rules/egress-rules.html)可以找到*service*字段支持的域名通配符格式。

目前Istio在服务网格内只支持通过HTTP协议访问外部服务。然而边车（sidecar）和外部服务之间的通信可以是基于TLS的。如上面的例子所示，通过把*protocol*字段设置为"https",边车（sidecar）就可以通过TLS和外部服务进行通信。此时，服务网格内的应用只能通过HTTP协议对外部服务进行调用（例如，使用`http://secure-service.foo.com:443`，而不是`https://secure-service.foo.com`来访问外部服务），然而边车（sidecar）在向外部服务转发该请求时会采用TLS。

只要这些规则采用相同的目的地配置，以指向相同的外部服务，出站规则可以很好地和路由规则及目的地策略协同工作。例如下面的规则可以和前面示例中的出站规则一起作用，将这些外部服务的调用超时设置为10秒。

```yaml
apiVersion: config.istio.io/v1alpha2
kind: RouteRule
metadata:
  name: foo-timeout-rule
spec:
  destination:
    service: *.foo.com
  httpReqTimeout:
    simpleTimeout:
      timeout: 10s
```

目的地策略和路由规则的流量重定向和前转，重试，超时，故障注入策略等特性都可以很好地支持外部服务。但是由于外部服务没有多版本的概念，和服务版本关联的按权重的路由规则是不支持的。
