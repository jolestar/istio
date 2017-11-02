# 配置基础访问控制

下面的任务展示了如何使用 Kubernetes 标签来控制对一个服务的访问。

## 开始之前

- 在 Kubernetes 上跟随[安装指南](../../setup/install-kubernetes.md)部署 Istio。
- 部署 [BookInfo](../../guides/bookinfo.html) 示例应用。
- 设置基于版本的应用路由，用户“jason”对`reviews`服务的访问会被指向 v2 版本，其他用户则会访问
到 v3 版本。

~~~
istioctl create -f samples/bookinfo/kube/route-rule-reviews-test-v2.yaml
istioctl create -f samples/bookinfo/kube/route-rule-reviews-v3.yaml
~~~

> 注意：如果在前面的任务中存在有冲突的规则，可以用 `istiocl replace` 来替代 `istioctl create`。

> 注意：如果使用一个非`default`的命名空间，需要使用`istioctl -n namespace ...`的方式来
指定命名空间

## 使用`denials`进行访问控制

借助 Istio 能够根据 Mixer 中的任何属性来对一个服务进行访问控制。Mixer
Selector 可以进行对服务请求进行有条件的拒绝，这就构成了访问控制能力的基础。

[BookInfo](../../guides/bookinfo.html) 示例应用中的`ratings`服务会被几个不同版本的
`reviews`服务所访问。我们尝试切断`reviews`服务的`v3`版本对`ratings`服务的访问。

1.用浏览器访问 BookInfo 的`productpage`页面（http://$GATEWAY_URL/productpage）。

如果使用 "jason" 用户登录，会看到`ratings`服务展示出的星星图标是黑色的，这证明被`ratings`服
务是由 "v2" 版本的`reviews`服务调用的；如果登出或者使用其他用户登录，就会看到红色的星星，这代
表 "v3" 版本的`reviews`服务在调用`ratings`服务。

2.显式的拒绝从`v3`版本`reviews`服务到`ratings`的访问。

使用下面的命令来创建 Handler实例和拒绝规则：

`istioctl create -f samples/bookinfo/kube/mixer-rule-deny-label.yaml`

会产生类似的输出：

~~~
Created config denier/default/denyreviewsv3handler at revision 2882105
Created config checknothing/default/denyreviewsv3request at revision 2882106
Created config rule/default/denyreviewsv3 at revision 2882107
~~~

注意`denyreviewsv3`规则：

`match: destination.labels["app"] == "ratings" && source.labels["app"]=="reviews" && source.labels["version"] == "v3"`

代表从`v3`版本的`reviews`服务到`ratings`服务的请求。

这一规则使用`denier`适配器来拒绝源于`v3`版本的`reviews`服务的请求。这一适配器会使用预置的状
态码和消息来拒绝服务请求。状态码和消息的定义来自于
[denier](../../reference/config/mixer/adapters/denier.html)适配器的配置。

3.刷新浏览器

如果没有登录、或使用 "jason" 之外的用户登录，因为`ratings`服务拒绝了来自`reviews:v3`的请求。
而如果使用 "jason" 的身份登录（会使用`reviews:v2`），就会看到黑色的星星了。

## 使用`whitelists`进行访问控制

Istio 还支持基于属性的黑名单和白名单。下面的白名单配置是跟上一节中的`denier`配置等价的。这一
规则也会拒绝来自`v3`版本`reviews`服务的请求。

1.删除上一节加入的配置：

`istioctl delete -f samples/bookinfo/kube/mixer-rule-deny-label.yaml`

2.检查对`productpage`的访问(http://$GATEWAY_URL/productpage)

在登出状态下，会看到红色星星。但是在执行下面的步骤之后，只有使用 "jason" 登录才能看到星星。

3.创建`listchecker`适配器，其中包含`v1`和`v2`的列表。把下面的 YAML 文件保存为
`whitelist-handler.yaml`：

~~~yaml
apiVersion: config.istio.io/v1alpha2
kind: listchecker
metadata:
  name: whitelist
spec:
  # providerUrl: 可以使用这一字段异步获取该字段网址中定义的黑白名单内容。
  overrides: ["v1", "v2"]  # 静态列表
  blacklist: false
~~~  

然后运行命令：

`istioctl create -f whitelist-handler.yaml`

4.创建一个`listentry`模板的实例，用于解释版本标签。把下面的 YAML 代码保存为
`appversion-instance.yaml`：

~~~yaml
apiVersion: config.istio.io/v1alpha2
kind: listentry
metadata:
  name: appversion
spec:
  value: source.labels["version"]
~~~

接下来运行：

`istioctl create -f appversion-instance.yaml`

5.针对`ratings`服务，启用`whitelist`。创建`checkversion-rule.yaml`：

~~~yaml
apiVersion: config.istio.io/v1alpha2
kind: rule
metadata:
  name: checkversion
spec:
  match: destination.labels["app"] == "ratings"
  actions:
  - handler: whitelist.listchecker
    instances:
    - appversion.listentry
~~~

最后运行：

`istioctl create -f checkversion-rule.yaml`

6.访问 Bookinfo 实例（http://$GATEWAY_URL/productpage）。

如果是登出状态，则无法看到星星；用 "jason" 登录，就会看到黑色的星星。

## 清理

- 删除 Mixer 配置：

~~~
istioctl delete -f checkversion-rule.yaml
istioctl delete -f appversion-instance.yaml
istioctl delete -f whitelist-handler.yaml
~~~

- 删除应用路由规则：

~~~
stioctl delete -f samples/bookinfo/kube/route-rule-reviews-test-v2.yaml
istioctl delete -f samples/bookinfo/kube/route-rule-reviews-v3.yaml
~~~

- 如果不准备进行下面的任务，可以参考
[BookInfo 清理](../../../guides/bookinfo.html#cleanup)进行善后工作。

## 延伸阅读

- 学习如何利用 Service Account 来进行访问控制：[点击这里](secure-access-control.html)。

- 深入学习 [Mixer](../../../concepts/policy-and-control/mixer.html) 和
[Mixer 配置](../../../concepts/policy-and-control/mixer-config.html)。

- [属性词汇表](../../../reference/config/mixer/attribute-vocabulary.html)。

- [配置编写指南](../../../reference/writing-config.html)

- [Kubernetes 的网络策略和 Istio 访问控制策略的差异](https://istio.io/blog/using-network-policy-in-concert-with-istio.html)
