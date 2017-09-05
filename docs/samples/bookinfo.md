# BookInfo

该示例部署由四个单独的微服务组成的简单应用程序，用于演示Istio服务网格的各种功能。

## 开始之前

* 如果您使用GKE，请确保您的集群至少有4个标准GKE节点。

* 按照 [安装指南](../tasks/installing-istio.md) 中的说明安装 Istio 。

## 概况

在本示例中，我们将部署一个简单的应用程序，显示有关书籍的信息，类似于在线书店的单个目录条目。在页面上显示的是书的描述，书籍详细信息（ISBN，页数等）和一点书评。

BookInfo 应用程序分为四个单独的微服务器：

* *productpage*. productpage(产品页面)微服务 调用 *details* 和 *reviews* 微服务来填充页面.
* *details*. details 微服务包含书籍的详细信息.
* *reviews*. reviews 微服务包含书籍的书评. 它也调用 *ratings* 微服务.
* *ratings*. ratings 微服务包含书籍的伴随书评的评级信息.

有3个版本的 reviews 微服务：

* 版本v1不调用 ratings 服务。
* 版本v2调用 ratings ，并将每个评级显示为1到5个黑色星。
* 版本v3调用 ratings ，并将每个评级显示为1到5个红色星。

应用程序的端到端架构如下所示。

<img src="./img/bookinfo/noistio.svg" alt="BookInfo Application without Istio" title="BookInfo Application without Istio" />

该应用程序是多语言的，即微服务是用不同的语言编写的。

## 启动应用程序

1. 将目录更改为Istio安装目录的根目录。

1. 构建应用程序容器：

    ```bash
    kubectl apply -f <(istioctl kube-inject -f samples/apps/bookinfo/bookinfo.yaml)
    ```

	上述命令启动四个微服务器并创建网关入口资源，如下图所示。

    reviews 微服务有3个版本：v1，v2和v3。

    > #### info::请注意
    >
    > 在实际部署中，随着时间的推移部署新版本的微服务，而不是同时部署所有版本。

	请注意，该 `istioctl kube-inject` 命令用于在创建部署之前修改 `bookinfo.yaml` 文件。这将把 Envoy 注入到 Kubernetes 资源,如 [这里](../reference/commands/istioctl.md#istioctl-kube-inject) 记载的。

    因此，所有的微型服务器现在都和一个能够管理呼入和呼出调用的 Envoy sidecar。更新后的图表如下所示：

	<img src="./img/bookinfo/withistio.svg" alt="BookInfo Application" title="BookInfo Application" />

1. 确认所有服务和 pod 已正确定义并运行：

    ```bash
    kubectl get services
    ```

	这将产生以下输出：

    ```bash
    NAME                       CLUSTER-IP   EXTERNAL-IP   PORT(S)              AGE
    details                    10.0.0.31    <none>        9080/TCP             6m
    istio-ingress              10.0.0.122   <pending>     80:31565/TCP         8m
    istio-pilot                10.0.0.189   <none>        8080/TCP             8m
    istio-mixer                10.0.0.132   <none>        9091/TCP,42422/TCP   8m
    kubernetes                 10.0.0.1     <none>        443/TCP              14d
    productpage                10.0.0.120   <none>        9080/TCP             6m
    ratings                    10.0.0.15    <none>        9080/TCP             6m
    reviews                    10.0.0.170   <none>        9080/TCP             6m
    ```

    而且

    ```bash
    kubectl get pods
    ```

    将产生:

    ```bash
    NAME                                        READY     STATUS    RESTARTS   AGE
    details-v1-1520924117-48z17                 2/2       Running   0          6m
    istio-ingress-3181829929-xrrk5              1/1       Running   0          8m
    istio-pilot-175173354-d6jm7                 2/2       Running   0          8m
    istio-mixer-3883863574-jt09j                2/2       Running   0          8m
    productpage-v1-560495357-jk1lz              2/2       Running   0          6m
    ratings-v1-734492171-rnr5l                  2/2       Running   0          6m
    reviews-v1-874083890-f0qf0                  2/2       Running   0          6m
    reviews-v2-1343845940-b34q5                 2/2       Running   0          6m
    reviews-v3-1813607990-8ch52                 2/2       Running   0          6m
    ```

1. 确定网关入口URL：

    ```bash
    kubectl get ingress -o wide
    ```

    ```bash
    NAME      HOSTS     ADDRESS                 PORTS     AGE
    gateway   *         130.211.10.121          80        1d
    ```

	如果您的 Kubernetes 集群在支持外部负载均衡器的环境中运行，并且Istio入口服务能够获取外部IP，则入站资源 ADDRESS 将等于入口服务外部IP。

    ```bash
    export GATEWAY_URL=130.211.10.121:80
    ```

	> 有时当服务无法获取外部IP时，入口 ADDRESS 可能会显示 NodePort 地址列表。在这种情况下，您可以使用任何地址以及 NodePort 访问入口。但是，如果集群具有防火墙，则还需要创建防火墙规则以允许TCP流量到NodePort。例如，在GKE中，您可以使用以下命令创建防火墙规则：

    ```bash
    gcloud compute firewall-rules create allow-book --allow tcp:$(kubectl get svc istio-ingress -o jsonpath='{.spec.ports[0].nodePort}')
    ```

	如果您的部署环境不支持外部负载均衡器（例如 minikube），则 ADDRESS 字段将为空。在这种情况下，您可以使用 service NodePort：

    ```bash
    export GATEWAY_URL=$(kubectl get po -l istio=ingress -o 'jsonpath={.items[0].status.hostIP}'):$(kubectl get svc istio-ingress -o 'jsonpath={.spec.ports[0].nodePort}')
    ```

1. 使用以下 curl 命令确认 BookInfo 应用程序正在运行:

    ```bash
    curl -o /dev/null -s -w "%{http_code}\n" http://${GATEWAY_URL}/productpage
    ```
    ```bash
    200
    ```

## 清理

在完成 BookInfo 示例后，您可以卸载它，如下所示：

1. 删除路由规则并终止应用程序pod

    ```bash
    samples/apps/bookinfo/cleanup.sh
    ```

1. 确认关机

    ```bash
    istioctl get route-rules   #-- there should be no more routing rules
    kubectl get pods           #-- the BookInfo pods should be deleted
    ```

## 下一步

现在您已经启动并运行了 BookInfo 示例，您可以将浏览器指向 `http://$GATEWAY_URL/productpage` 来看正在运行的应用程序，并使用 Istio 来控制流量路由，注入故障，限速等。

要开始，请查看 [请求路由任务](../tasks/request-routing.md)。

