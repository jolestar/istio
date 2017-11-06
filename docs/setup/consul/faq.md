# FAQ

* _我的应用程序无法运行，我应该在哪里调试？_

  Please ensure all required containers are running: etcd, istio-apiserver, consul, registrator, pilot.  If one of them is not running, you may find the {containerID} using `docker ps -a` and then use `docker logs {containerID}` to read the logs.   

* _最终我该如何使用 istioctl 取消 context 的设置？_

  Your ```kubectl``` is switched to use the istio context at the end of the `istio context-create` command.  You can use ```kubectl config get-contexts``` to obtain the list of contexts and ```kubectl config use-context {desired-context}``` to switch to use your desired context.
