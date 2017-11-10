# 使用 Docker 快速开始

使用  Docker Compose 安装和配置 Istio service mesh 的快速入门指南。


## 前置条件

* [Docker](https://docs.docker.com/engine/installation/#cloud)
* [Docker Compose](https://docs.docker.com/compose/install/)

## 安装步骤

1. 到 [Istio release](https://github.com/istio/istio/releases) 页面，下载适合自己的操作系统的最新安装文件。 如果你用的是 MacOS 或者 Linux ，也可以运行下面的命令自动下载并解压最新的发布包。
   ```bash
   curl -L https://git.io/getLatestIstio | sh -
   ```

2. 解压安装文件，并进入解压后的文件夹。安装目录包含以下内容：
    * 示例应用在  `samples/` 下面。
    *  `istioctl` 客户端在  `bin/` 文件夹下。 `istioctl` 用来创建路由以及策略。
    *  `istio.VERSION` 配置文件。

3. 添加 `istioctl` 客户端到你的 PATH 环境变量中。在 MacOS 或者 Linux 系统上可以运行以下命令：

   ```bash
   export PATH=$PWD/bin:$PATH
   ```

4. 跳转到 Istio 的安装根目录。

5.  启动 Istio 控制面板容器：

    ```bash
    docker-compose -f install/eureka/istio.yaml up -d
    ```

6. 确保所有的 docker 容器都运行正常：

   ```bash
   docker ps -a
   ```
   > 如果 Istio Pilot 容器终止了，确保你运行了 `istioctl context-create`  命令，然后重新运行前一步的 docker-compose 命令。

7. 配置  `istioctl` 使用 Istio API 服务的本地映射端口：

    ```bash
    istioctl context-create --context istio-local --api-server http://localhost:8080
    ```

## 部署应用

现在你可以部署你自己的应用或者安装程序中提供的示例应用，比如  [BookInfo](../../docs/guides/bookinfo.md)。

> 注意 1: 因为 Docker 安装环境下没有 pods 的概念，Istio 的 sidecar 和应用程序会运行在同一个容器中。我们使用 [Registrator](http://gliderlabs.github.io/registrator/latest/) 自动注册服务实例到 Console 服务注册中心。

> 注意 2: 应用的 HTTP 流量必须必须使用 HTTP/1.1 或者 HTTP/2.0 协议，因为 Istio 不支持 HTTP/1.0。 

```bash
docker-compose -f <your-app-spec>.yaml up -d
```

## 卸载

1. 卸载 Istio 核心组件只需要删除 docker 容器：

```bash
docker-compose -f install/eureka/istio.yaml down
```

## 下一步

* 参看 [BookInfo](../../docs/guides/bookinfo.md) 示例应用。
