# 翻译说明

目前istio官方文档的翻译由[Service Mesh中文网](http://servicemesh.cn)在主持，这是一个公益性的工作。

## 贡献您的力量

如果有朋友有意愿加入我们的翻译工作，可以有如下方式贡献您的力量：

1. 审核和校对：如果您在阅读时发现内容有谬误，可以通过 [github issue][]提交issue给我们，您也可以fork出来然后通过PR提交更改
2. 直接参与翻译工作： 您可以报名加入我们的翻译小组，请先加入Service Mesh技术社区，然后在微信群中联系我们。

	> 加入方式：请联系微信ID xiaoshu062 ，注明“服务网格”。

## 工作方式

以下是和我们翻译内容相关的信息：

- [Istio官方文档在线浏览地址][istio-publish]
- [Istio官方文档源文件托管地址][istio-source]
- [保存翻译后的中文内容的github地址][chinese-source]
- [中文翻译内容发布地址][chinese-publish]：目前用的是gitbook，会自动从github拉取最新内容生成html发布出来

我们目前采用github保存翻译后的内容，翻译团队使用github issue和project来管理。具体方式为：

1. 未翻译的内容会以github issue的方式拆分为多个issue
2. 请在[github issue]中领取感兴趣的任务，并将issue assign到自己的github账号：切记必须这样做明确认领，避免其他人在不知情的情况下重复翻译同一个内容
3. 翻译完成后在微信群中联系其他人做review
4. review完成，关闭issue

过程中：

1. 如果觉得issue包含的内容太多，可以直接修改issue，缩小内容的范围，然后为减少的内容新开其他issue（可以新开一个或者多个）
2. 直接提交翻译后的内容，哪怕还没有review（甚至只翻译了一半）
3. 只使用master分支，暂时不拉分支，避免麻烦

[github issue]:https://github.com/doczhcn/istio/issues
[istio-source]:https://github.com/istio/istio.github.io/tree/master/_docs
[istio-publish]:https://istio.io/docs/
[chinese-source]:https://github.com/doczhcn/istio
[chinese-publish]:https://doczhcn.gitbooks.io/istio/

## 约定和术语表

### 术语表

```
service      服务
microservice 微服务
application  应用/应用程序
mutual TLS   双向TLS

next step         下一步
before you begin  开始之前
cleanup           清除
understanding What happened 理解原理
Further reading   进阶阅读

configure 配置
setting   设置
traffic   流量
authentication  认证
authorization   授权
```

### 约定

以下词汇不翻译：

```
sidecar
BookInfo
productpage
reviews
ratings
HTTP header
TLS
```
