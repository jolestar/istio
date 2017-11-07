# FAQ

* 当想要自动注入 sidecar 时，如何检查我的己群是否启用 alpha 功能？

  自动注入 sidecar 依赖 [initilizer alpha 功能](https://kubernetes.io/docs/admin/extensible-admission-controllers/#enable-initializers-alpha-feature)。

  执行下列命令检查是否启用了 initializer（如果输出为空则说明 initializer 没有被启用）：

  ```bash
  kubectl api-versions | grep admissionregistration
  ```

  另外，在启动 kubernetes API server 时必须 [启用](https://kubernetes.io/docs/admin/extensible-admission-controllers/#enable-initializers-alpha-feature) Initializer 插件。如果 `Initializer` 插件启用失败的话，在创建初始化 deployment 的时候将会产生如下错误。

  > Deployment "istio-initializer" is invalid: metadata.initializers.pending: Invalid value: "null": must be non-empty when result is not set

* 自动注入 sidecar 无效，如何调试？

  确保您的群集已满足自动注入 sidecar 的 [前置条件](sidecar-injection.md#automatic-sidecar-injection)。如果您的微服务部署在 kube-system、kube-public 或 istio-system 命名空间中，则它们将无法自动注入 sidecar。请改用其他命名空间。

* 我可以将当前的 Istio v0.1.0 版本迁移到 v0.2.x 版本吗？

  不支持从 Istio 0.1.x 迁移到 0.2.x。您必须先卸载 Istio v0.1，包括已注入 sidecar 的 pod，然后重新安装 Istio v0.2。

