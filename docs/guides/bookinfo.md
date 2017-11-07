# BookInfo

该示例部署由四个单独的微服务组成的简单应用程序，用于演示Istio服务网格的各种功能。

## 概况

在本示例中，我们将部署一个简单的应用程序，显示书籍的信息，类似于网上书店的书籍条目。在页面上有书籍的描述、详细信息（ISBN、页数等）和书评。

BookInfo 应用程序包括四个独立的微服务：

* *productpage*：productpage(产品页面)微服务，调用 *details* 和 *reviews* 微服务来填充页面。
* *details*：details 微服务包含书籍的详细信息。
* *reviews*：reviews 微服务包含书籍的点评。它也调用 *ratings* 微服务。
* *ratings*：ratings 微服务包含随书评一起出现的评分信息。

有3个版本的 reviews 微服务：

* 版本v1不调用 ratings 服务。
* 版本v2调用 ratings ，并将每个评级显示为1到5个黑色星。
* 版本v3调用 ratings ，并将每个评级显示为1到5个红色星。

应用程序的端到端架构如下所示。

<img src="./img/bookinfo/noistio.svg" alt="BookInfo Application without Istio" title="BookInfo Application without Istio" />

该应用程序是多语言构建的，即这些微服务是用不同的语言编写的。值得注意的是，这些服务与 Istio 没有任何依赖关系，单这是个有趣的 Service Mesh 示例，特别是因为评论服务和众多的语言和版本。

## 开始之前

如果您还没有这样做，请按照与您的平台 [安装指南](../../docs/setup/index.md) 对应的说明安装Istio。

## 部署应用程序

使用 Istio 运行应用程序示例不需要修改应用程序本身。相反，我们只需要在支持 Istio 的环境中配置和运行服务， Envoy sidecar 将会注入到每个服务中。所需的命令和配置根据运行时环境的不同而有所不同，但在所有情况下，生成的部署将如下所示：

<img src="./img/bookinfo/noistio.svg" alt="BookInfo Application without Istio" title="BookInfo Application without Istio" />

所有的微服务都将与一个 Envoy sidecar 一起打包，拦截这些服务的入站和出站的调用请求，提供通过 Istio 控制平面从外部控制整个应用的路由，遥测收集和策略执行所需的 hook。

要启动该应用程序，请按照以下对应于您的 Istio 运行时环境的说明进行操作。

### 在 Kubernetes 中运行

> 注意：如果您使用 GKE，清确保您的集群至少有 4 个标准的 GKE 节点。如果您使用 Minikube，请确保您至少有 4GB 内存。

1. 将目录更改为 Istio 安装目录的根目录。

2. 构建应用程序容器：

    如果您使用 **自动注入 sidecar** 的方式部署的集群，那么只需要使用 `kubectl` 命令部署服务：

    ```bash
    kubectl apply -f samples/bookinfo/kube/bookinfo.yaml
    ```

    如果您使用 **手动注入 sidecar** 的方式部署的集群，清使用下面的命令：

    ```bash
    kubectl apply -f <(istioctl kube-inject -f samples/apps/bookinfo/bookinfo.yaml)
    ```

    请注意，该 `istioctl kube-inject` 命令用于在创建部署之前修改 `bookinfo.yaml` 文件。这将把 Envoy 注入到 Kubernetes 资源,如 [这里](../reference/commands/istioctl.md#istioctl-kube-inject) 记载的。

    上述命令启动四个微服务并创建网关入口资源，如下图所示。3 个版本的评论的服务 v1、v2、v3 都已启动。

    > 请注意在实际部署中，随着时间的推移部署新版本的微服务，而不是同时部署所有版本。

3. 确认所有服务和 pod 已正确定义并运行：

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
# 确定 ingress IP 和端口

1. 如果您的 kubernetes 集群环境支持外部负载均衡器的话，可以使用下面的命令获取 ingress 的IP地址：

   ```bash
   kubectl get ingress -o wide
   ```

   输出如下所示：

   ```bash
   NAME      HOSTS     ADDRESS                 PORTS     AGE
   gateway   *         130.211.10.121          80        1d
   ```

   Ingress 服务的地址是：

   ```bash
   export GATEWAY_URL=130.211.10.121:80
   ```

2. *GKE*：如果服务无法获取外部 IP，`kubectl get ingress -o wide` 会显示工作节点的列表。在这种情况下，您可以使用任何地址以及 NodePort 访问入口。但是，如果集群具有防火墙，则还需要创建防火墙规则以允许TCP流量到NodePort，您可以使用以下命令创建防火墙规则：

   ```bash
   export GATEWAY_URL=<workerNodeAddress>:$(kubectl get svc istio-ingress -n istio-system -o jsonpath='{.spec.ports[0].nodePort}')
   gcloud compute firewall-rules create allow-book --allow tcp:$(kubectl get svc istio-ingress -n istio-system -o jsonpath='{.spec.ports[0].nodePort}')
   ```

3. *IBM Bluemix Free Tier*：在免费版的 Bluemix 的 kubernetes 集群中不支持外部负载均衡器。您可以使用工作节点的公共 IP，并通过 NodePort 来访问 ingress。工作节点的公共 IP可以通过如下命令获取：

    ```bash
    bx cs workers <cluster-name or id>
    export GATEWAY_URL=<public IP of the worker node>:$(kubectl get svc istio-ingress -n istio-system -o jsonpath='{.spec.ports[0].nodePort}')
    ```

4. *Minikube*：Minikube 不支持外部负载均衡器。您可以使用 ingress 服务的主机 IP 和 NodePort 来访问 ingress：

    ```bash
    export GATEWAY_URL=$(kubectl get po -l istio=ingress -o 'jsonpath={.items[0].status.hostIP}'):$(kubectl get svc istio-ingress -o 'jsonpath={.spec.ports[0].nodePort}')
    ```

### 在 Consul 或 Eureka 环境下使用 Docker 运行

1. 切换到 Istio 的安装根目录下。

2. 启动应用程序容器。

   1. 执行下面的命令测试 Consul：

      ```bash
       docker-compose -f samples/bookinfo/consul/bookinfo.yaml up -d
      ```

   2. 执行下面的命令测试 Eureka：

      ```bash
       docker-compose -f samples/bookinfo/eureka/bookinfo.yaml up -d
      ```

3. 确认所有容器都在运行：

   ```bash
   docker ps -a
   ```

   > 如果 Istio Pilot 容器终止了，重新执行上面的命令重新运行。

4. 设置 `GATEWAY_URL`:

   ```bash
   export GATEWAY_URL=localhost:9081
   ```

## 下一步

使用以下 `curl` 命令确认 BookInfo 应用程序正在运行:


```bash
curl -o /dev/null -s -w "%{http_code}\n" http://${GATEWAY_URL}/productpage
```
```bash
200
```

你也可以通过在浏览器中打开 `http://$GATEWAY_URL/productpage` 页面访问 Bookinfo 网页。如果您多次刷新浏览器将在 productpage 中看到评论的不同的版本，它们会按照 round robin（红星、黑星、没有星星）的方式展现，因为我们还没有使用 Istio 来控制版本的路由。

现在，您可以使用此示例来尝试 Istio 的流量路由、故障注入、速率限制等功能。要继续的话，请参阅 [Istio 指南](../../docs/guides/index.md)，具体取决于您的兴趣。[智能路由](../../docs/guides/intelligent-routing.md) 是初学者入门的好方式。

## 清理

在完成 BookInfo 示例后，您可以卸载它，如下所示：

### 卸载 Kubernetes 环境

1. 删除路由规则，终止应用程序 pod

   ```bash
   samples/bookinfo/kube/cleanup.sh
   ```

2. 确认关闭

   ```bash
   istioctl get routerules   #-- there should be no more routing rules
   kubectl get pods          #-- the BookInfo pods should be deleted
   ```

### 卸载 docker 环境

1. 删除路由规则和应用程序容器

   1. 若使用 Consul 环境安装，执行下面的命令：

      ```bash
      samples/bookinfo/consul/cleanup.sh
      ```

   2. 若使用 Eureka 环境安装，执行下面的命令：

      ```bash
      samples/bookinfo/eureka/cleanup.sh
      ```

2. 确认清理完成：

   ```bash
   istioctl get routerules   #-- there should be no more routing rules
   docker ps -a              #-- the BookInfo containers should be deleted
   ```