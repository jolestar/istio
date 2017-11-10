# FAQ

* _我的应用程序不能正常工作，我怎么进行故障排查？_

  请确保所有的必要的容器运行正常：etcd，istio-apiserver，consul，registrator，istio-pilot。如果其中哪个没有运行，你可以使用  `docker ps -a`  找到 {containerID} ，然后使用  `docker logs {containerID}` 去查看该容器的日志。

* _最后我如何通过 `istioctl`  重置上下文（context）？_

  使用 istio context-create 命令后，你的 kubectl 会被切换到 istio 的上下文。 你可以使用 kubectl config get-contexts 来获取上下文列表，并使用 kubectl config use-context {desired-context} 来切换到你期望使用的上下文。
