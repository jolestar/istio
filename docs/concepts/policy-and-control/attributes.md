# 属性

本节描述Istio属性，它们是什么以及如何使用它们。

## 背景

Istio使用 *属性* 来控制在服务网格中运行的服务的运行时行为。属性是描述入口和出口流量的有名称和类型的元数据片段，以及此流量发生的环境。Istio属性携带特定信息片段，例如API请求的错误代码，API请求的延迟或TCP连接的原始IP地址。例如：

	request.path: xyz/abc
	request.size: 234
	request.time: 12:34:56.789 04/17/2017
	source.ip: 192.168.0.1
	target.service: example

## 属性词汇表

给定的Istio部署有一个它可以理解的固定的属性词汇表。具体词汇表由部署中使用的属性生产者集合决定。Istio的主要属性生产者是Envoy，尽管专业的Mixer适配器和服务也可以生成属性。

[这里](../../reference/config/mixer/attribute-vocabulary.md)定义了大多数Istio部署中可用的常用基准属性集。

## 属性名

Istio属性使用类似Java的完全限定标识符作为属性名。允许的字符是 `[_.a-z0-9]` 。该字符`"."`用作命名空间分隔符。例如，`request.size`和`source.ip`。

## 属性类型

Istio属性是强类型的。支持的属性类型由 [ValueType] 定义。

[ValueType]:https://github.com/istio/api/blob/master/mixer/v1/config/descriptor/value_type.proto
