# 配置安全访问控制

本任务将演示如何通过使用Istio认证提供的服务账户，来安全地对服务做访问控制。

当Istio双向TLS认证打开时，服务器就会根据其证书来认证客户端，并从证书获取客户端的服务账户。服务账户在`source.user`的属性中。请参考[Istio auth identity](../../concepts/security/mutual-tls.md#identity)了解Istio中服务账户的格式。

## 开始之前

* 根据说明[quick start](../../setup/kubernetes/quick-start.md)在开启认证的Kubernetes中安装Istio。注意，应当在[installation steps](../../setup/kubernetes/quick-start.md#installation-steps)中的第四步开启认证。

* 部署[BookInfo](../../guides/bookinfo.md)示例应用。

* 执行下列命令创建`bookinfo-productpage`服务账户，并重新部署`productpage`服务及其服务账户。

  ```bash
  kubectl create -f <(istioctl kube-inject -f samples/bookinfo/kube/bookinfo-add-serviceaccount.yaml)
  ```

  > 注意：如果使用`default`之外的命名空间，需要通过`istioctl -n namespace ...`来指定命名空间。

## 使用 _denials_ 做访问控制

在示例应用[BookInfo](../../guides/bookinfo.md)中，`productpage`服务会同时访问`reviews`服务和`details`服务。现在让`details`服务拒绝来自`productpage`服务的请求：

1. 在浏览器中打开BookInfo的`productpage` (http://$GATEWAY_URL/productpage)页面。

   将会在页面左下部看到"Book Details"部分，包括了类型、页数、出版社等信息。`productpage`服务是从`details`服务获取的"Book Details"信息。

1. 明确拒绝从`productpage` 到 `details`的请求。

   执行下列命令来安装拒绝规则及对应的handler和实例。

   ```bash
   istioctl create -f samples/bookinfo/kube/mixer-rule-deny-serviceaccount.yaml
   ```

   将会看到类似下面的输出：

   ```bash
   Created config denier/default/denyproductpagehandler at revision 2877836
   Created config checknothing/default/denyproductpagerequest at revision 2877837
   Created config rule/default/denyproductpage at revision 2877838
   ```

   注意`denyproductpage` 规则中的下述内容:

   ```
   match: destination.labels["app"] == "details" && source.user == "spiffe://cluster.local/ns/default/sa/bookinfo-productpage"
   ```

   匹配到了来自details服务上的服务账户 "_spiffe://cluster.local/ns/default/sa/bookinfo-productpage_"的请求。

   > 注意: 如果使用`default`之外的命名空间，请在`source.user`的值中使用自己的命名空间替换`default`。

   该规则使用`denier`适配器来拒绝这些请求。适配器通常通过一个预先配置的状态码和消息来拒绝请求。状态码和消息是在[denier](../../reference/config/mixer/adapters/denier.md)适配器配置中指定的。

1. 在浏览器中刷新`productpage`页面。

   将会看到消息

   "_Error fetching product details! Sorry, product details are currently unavailable for this book._"

   出现在页面的左下部分。这就验证了从`productpage`到`details`的访问是被禁止的。

## 清除

* 清除mixer配置：

  ```bash
  istioctl delete -f samples/bookinfo/kube/mixer-rule-deny-serviceaccount.yaml
  ```

* 如果不打算继续接下来的更多任务，可参考[BookInfo cleanup](../../guides/bookinfo.md#cleanup)的指南来关闭应用。

## 进阶阅读

* 了解更多关于[Mixer](../../concepts/policy-and-control/mixer.md) 和 [Mixer配置](../../concepts/policy-and-control/mixer-config.md)的内容。

* 探索全部的[属性字典](../../reference/config/mixer/attribute-vocabulary.md)。

* 阅读参考指南来[编写配置](../../reference/writing-config.md)。

* 通过[博客文章](https://istio.io/blog/using-network-policy-in-concert-with-istio.html)来理解Kubernetes网络策略和Istio访问控制策略的区别。
