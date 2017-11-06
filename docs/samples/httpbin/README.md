# Httpbin service

这个例子会运行[httpbin](https://httpbin.org/)作为Istio服务。Httpbin是一个著名的HTTP测试服务，可以用来测试所有类型的Istio特性。

用法:

1. 根据[安装指南](../../tasks/installing-istio.html)安装Istio。
2. 在Istio Service Mesh中启动httpbin服务：
~~~
kubectl apply -f <(istioctl kube-inject -f httpbin.yaml)
~~~

由于httpbin服务没有暴露在集群之外，所以无法直接curl，然而我们可以在集群内使用curl命令抓取httpbin:8000来验证这一服务是否正常工作，可以使用Docker hub镜像dockerqa/curl来完成这一任务：

~~~
kubectl run -i --rm --restart=Never dummy --image=dockerqa/curl:ubuntu-trusty --command -- curl --silent httpbin:8000/html
kubectl run -i --rm --restart=Never dummy --image=dockerqa/curl:ubuntu-trusty --command -- curl --silent httpbin:8000/status/500
time kubectl run -i --rm --restart=Never dummy --image=dockerqa/curl:ubuntu-trusty --command -- curl --silent httpbin:8000/delay/5
~~~

另外，还可以使用[ingress](../../tasks/traffic-management/ingress.html)或者启动[sleep服务](https://github.com/istio/istio/blob/master/samples/sleep)来调用httpbin。
