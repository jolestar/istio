# 拓展 Istio Mesh

将虚拟机或裸机集成到部署在 kubernetes 集群上的 Istio mesh 中的说明。

## 前置条件

* 按照 [安装指南](quick-start.md) 在 kubernetes 集群上安装 Istio service mesh。

* 机器必须具有到 mesh 端点的 IP 地址连接。这通常需要一个 VPC 或者 VPN，以及一个向端点提供直接路由（没有 NAT 或者防火墙拒绝）的容器网络。及其不需要访问有 Kubernetes 分配的 cluster IP。

* 虚拟机必须可以访问到 Istio 控制平面服务（如Pilot、Mixer、CA）和 Kubernetes DNS 服务器。通常使用 [内部负载均衡器](https://kubernetes.io/docs/concepts/services-networking/service/#internal-load-balancer) 来实现。

  您也可以使用 NodePort，在虚拟机上运行 Istio 的组件，或者使用自定义网络配置，有几个单独的文档会涵盖这些高级配置。

## 安装步骤

安装过程包括准备用于拓展的 mesh 和安装和配置虚拟机。

[install/tools/setupMeshEx.sh](https://raw.githubusercontent.com/istio/istio/master/install/tools/setupMeshEx.sh) ：这是一个帮助大家设置 kubernetes 环境的示例脚本。检查脚本内容和支持的环境变量（如 GCP_OPTS）。

[install/tools/setupIstioVM.sh](https://raw.githubusercontent.com/istio/istio/master/install/tools/setupIstioVM.sh)：这是一个用于配置主机环境的示例脚本。
您应该根据您的配置工具和DNS要求对其进行自定义。

准备要拓展的 Kubernetes 集群：

* 为 Kube DNS、Pilot、Mixer 和 CA 安装内部负载均衡器（ILB）。每个云供应商的配置都有所不同，根据具体情况修改注解。

> 0.2.7 版本的 YAML 文件的 DNS ILB 的 namespace 配置不正确。
> 使用 [这一个](https://raw.githubusercontent.com/istio/istio/master/install/kubernetes/mesh-expansion.yaml) 替代。
> `setupMeshEx.sh` 中也有错误。使用上面链接中的最新文件或者从 [GitHub.com/istio/istio](https://github.com/istio/istio/) 克隆。

```
kubectl apply -f install/kubernetes/mesh-expansion.yaml
```

* 生成要部署到虚拟机上的 Istio `cluster.env` 配置。该文件中包含要拦截的 cluster IP 地址范围。

```bash
export GCP_OPTS="--zone MY_ZONE --project MY_PROJECT"
```
```bash
install/tools/setupMeshEx.sh generateClusterEnv MY_CLUSTER_NAME
```

该示例生成的文件：

```bash
cat cluster.env
```
```
ISTIO_SERVICE_CIDR=10.63.240.0/20
```

* 产生虚拟机使用的 DNS 配置文件。这样可以让虚拟机上的应用程序解析到集群中的服务名称，这些名称将被 sidecar 拦截和转发。

```bash
# Make sure your kubectl context is set to your cluster
install/tools/setupMeshEx.sh generateDnsmasq
```

该示例生成的文件：

```bash
cat kubedns
```
```
server=/svc.cluster.local/10.150.0.7
address=/istio-mixer/10.150.0.8
address=/istio-pilot/10.150.0.6
address=/istio-ca/10.150.0.9
address=/istio-mixer.istio-system/10.150.0.8
address=/istio-pilot.istio-system/10.150.0.6
address=/istio-ca.istio-system/10.150.0.9
```

### 设置机器

例如，您可以使用下面的“一条龙”脚本复制和安装配置：

```bash
# 检查该脚本看看它是否满足您的需求
# 在 Mac 上，使用 brew install base64 或者 set BASE64_DECODE="/usr/bin/base64 -D"
export GCP_OPTS="--zone MY_ZONE --project MY_PROJECT"
```
```bash
install/tools/setupMeshEx.sh machineSetup VM_NAME
```

或者等效得手动安装步骤如下：

------ 手动安装步骤开始 ------

* 将配置文件和 Istio 的 Debian 文件复制到要加入到集群的每台机器上。重命名为 `/etc/dnsmasq.d/kubedns` 和`/var/lib/istio/envoy/cluster.env`。

* 配置和验证 DNS 配置。需要安装 `dnsmasq` 或者直接将其添加到 `/etc/resolv.conf` 中，或者通过 DHCP 脚本。验证配置是否有效，检查虚拟机是否可以解析和连接到 pilot，例如：

在虚拟机或外部主机上：
```bash
host istio-pilot.istio-system
```
产生的消息示例：
```bash
# Verify you get the same address as shown as "EXTERNAL-IP" in 'kubectl get svc -n istio-system istio-pilot-ilb'
istio-pilot.istio-system has address 10.150.0.6
```
检查是否可以解析  cluster IP。实际地址取决您的 deployment：
```bash
host istio-pilot.istio-system.svc.cluster.local.
```
该示例产生的消息：
```
istio-pilot.istio-system.svc.cluster.local has address 10.63.247.248
```
同样检查 istio-ingress：
```bash
host istio-ingress.istio-system.svc.cluster.local.
```
该示例产生的消息：
```
istio-ingress.istio-system.svc.cluster.local has address 10.63.243.30
```

* 验证连接性，检查迅即是否可以连接到 Pilot 的端点：

```bash
curl 'http://istio-pilot.istio-system:8080/v1/registration/istio-pilot.istio-system.svc.cluster.local|http-discovery'
```
```json
{
  "hosts": [
   {
    "ip_address": "10.60.1.4",
    "port": 8080
   }
  ]
}
```
```bash
# 在虚拟机上使用上面的地址。将直接连接到运行 istio-pilot 的 pod。
curl 'http://10.60.1.4:8080/v1/registration/istio-pilot.istio-system.svc.cluster.local|http-discovery'
```


* 提取出实话 Istio 认证的 secret 并将它复制到机器上。Istio 的默认安装中包括 CA，即使是禁用了自动 `mTLS` 设置（她为每个 service account 创建 secret，secret 命名为 `istio.<serviceaccount>`）也会生成 Istio secret。建议您执行此步骤，以便日后启用 mTLS，并升级到默认启用 mTLS 的未来版本。

```bash
# ACCOUNT 默认是 'default'，SERVICE_ACCOUNT 是环境变量
# NAMESPACE 默认为当前 namespace，SERVICE_NAMESPACE 是环境变量
# （这一步由 machineSetup 完成）
# 在 Mac 上执行 brew install base64 或者 set BASE64_DECODE="/usr/bin/base64 -D"
install/tools/setupMeshEx.sh machineCerts ACCOUNT NAMESPACE
```

生成的文件（`key.pem`, `root-cert.pem`, `cert-chain.pem`）必须拷贝到每台主机的 /etc/certs 目录，并且让 istio-proxy 可读。 

* 安装 Istio Debian 文件，启动 `istio` 和 `istio-auth-node-agent` 服务。
  从 [github releases](https://github.com/istio/istio/releases) 获取 Debian 安装包：

  ```bash
  # 注意：在软件源配置好后，下面的额命令可以使用 'apt-get' 命令替代。

  source istio.VERSION # defines version and URLs env var
  curl -L ${PILOT_DEBIAN_URL}/istio-agent.deb > ${ISTIO_STAGING}/istio-agent.deb
  curl -L ${AUTH_DEBIAN_URL}/istio-auth-node-agent.deb > ${ISTIO_STAGING}/istio-auth-node-agent.deb
  curl -L ${PROXY_DEBIAN_URL}/istio-proxy.deb > ${ISTIO_STAGING}/istio-proxy.deb

  dpkg -i istio-proxy-envoy.deb
  dpkg -i istio-agent.deb
  dpkg -i istio-auth-node-agent.deb

  systemctl start istio
  systemctl start istio-auth-node-agent
  ```

------ 手动安装步骤结束 ------

安装完成后，机器就能访问运行在 Kubernetes 集群上的服务或者其他的 mesh 拓展的机器。

```bash
   # Assuming you install bookinfo in 'bookinfo' namespace
   curl productpage.bookinfo.svc.cluster.local:9080
```
```
   ... html content ...
```

检查进程是否正在运行：
```bash
ps aux |grep istio
```
```
root      6941  0.0  0.2  75392 16820 ?        Ssl  21:32   0:00 /usr/local/istio/bin/node_agent --logtostderr
root      6955  0.0  0.0  49344  3048 ?        Ss   21:32   0:00 su -s /bin/bash -c INSTANCE_IP=10.150.0.5 POD_NAME=demo-vm-1 POD_NAMESPACE=default exec /usr/local/bin/pilot-agent proxy > /var/log/istio/istio.log istio-proxy
istio-p+  7016  0.0  0.1 215172 12096 ?        Ssl  21:32   0:00 /usr/local/bin/pilot-agent proxy
istio-p+  7094  4.0  0.3  69540 24800 ?        Sl   21:32   0:37 /usr/local/bin/envoy -c /etc/istio/proxy/envoy-rev1.json --restart-epoch 1 --drain-time-s 2 --parent-shutdown-time-s 3 --service-cluster istio-proxy --service-node sidecar~10.150.0.5~demo-vm-1.default~default.svc.cluster.local
```
检查 Istio auth-node-agent 是否健康：
```bash
sudo systemctl status istio-auth-node-agent
```
```
● istio-auth-node-agent.service - istio-auth-node-agent: The Istio auth node agent
   Loaded: loaded (/lib/systemd/system/istio-auth-node-agent.service; disabled; vendor preset: enabled)
   Active: active (running) since Fri 2017-10-13 21:32:29 UTC; 9s ago
     Docs: http://istio.io/
 Main PID: 6941 (node_agent)
    Tasks: 5
   Memory: 5.9M
      CPU: 92ms
   CGroup: /system.slice/istio-auth-node-agent.service
           └─6941 /usr/local/istio/bin/node_agent --logtostderr

Oct 13 21:32:29 demo-vm-1 systemd[1]: Started istio-auth-node-agent: The Istio auth node agent.
Oct 13 21:32:29 demo-vm-1 node_agent[6941]: I1013 21:32:29.469314    6941 main.go:66] Starting Node Agent
Oct 13 21:32:29 demo-vm-1 node_agent[6941]: I1013 21:32:29.469365    6941 nodeagent.go:96] Node Agent starts successfully.
Oct 13 21:32:29 demo-vm-1 node_agent[6941]: I1013 21:32:29.483324    6941 nodeagent.go:112] Sending CSR (retrial #0) ...
Oct 13 21:32:29 demo-vm-1 node_agent[6941]: I1013 21:32:29.862575    6941 nodeagent.go:128] CSR is approved successfully. Will renew cert in 29m59.137732603s
```

## 在拓展的 mesh 中的机器上运行服务

* 配置 sidecar 拦截端口。在  `/var/lib/istio/envoy/sidecar.env` 中通过 `ISTIO_INBOUND_PORTS` 环境变量配置。

  例如（运行服务的虚拟机）：

   ```bash
   echo "ISTIO_INBOUND_PORTS=27017,3306,8080" > /var/lib/istio/envoy/sidecar.env
   systemctl restart istio
   ```

* 手动配置 selector-less 的 service 和 endpoint。“selector-less” service 用于那些不依托 Kubernetes pod 的 service。

  例如，在有权限的机器上修改 Kubernetes 中的 service：
   ```bash
   # istioctl register servicename machine-ip portname:port
   istioctl -n onprem register mysql 1.2.3.4 3306
   istioctl -n onprem register svc1 1.2.3.4 http:7000
   ```

安装完成后，Kubernetes pod 和其它 mesh 扩展将能够访问集群上运行的服务。

## 整合到一起

请参阅 [拓展 BookInfo Mesh](../../../docs/guides/integrating-vms.md) 指南。
