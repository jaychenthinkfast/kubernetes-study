# submariner

集群网络互通方案，可以让跨集群 pod，service （ip)互通、service 可以导出到专用域名解析跨集群访问,
支持 vxlan ，ipsec

## 准备 2 个集群

* 使用 kubekey 进行快速集群创建，这里使用kubekey 进行部署

```text
https://github.com/kubesphere/kubekey/releases/download/v3.1.7/kubekey-v3.1.7-linux-amd64.tar.gz
```

* 配置文件参考

```
config-sample-a.yaml
config-sample-b.yaml
```

配置文件按需修改， 如果有公网域名建议配置可以方便后续外部访问部署Submariner组件，
配置特别需要注意两个集群的 pod，service网段不能重叠，calico 需设置成 vxlan 模式，

* 安装命令参考

```text
kk create cluster -f config-sample-a.yaml
kk create cluster -f config-sample-b.yaml
```

## 安装 Submariner CLI（subctl）

Submariner 提供了一个命令行工具 **subctl**，用于简化安装和配置。

* 在本地机器上下载最新版本的 **subctl**：

  ```
  curl -Ls https://get.submariner.io | bash
  ```

  输出
  ```
  Installing subctl version latest
  OS detected:           darwin
  Architecture detected: amd64
  Download URL:          https://github.com/submariner-io/releases/releases/download/v0.20.0/subctl-v0.20.0-darwin-amd64.tar.gz

    Downloading...
    Download attempt #1...
    subctl has been installed as /Users/chenjie/.local/bin/subctl
    This provides subctl version: v0.20.0

    please update your path (and consider adding it to ~/.profile):
    export PATH=$PATH:/Users/chenjie/.local/bin
  ```
* 将 **subctl** 添加到 PATH：
  也就是上面输出最后一行添加到 PATH
  比如我本地 mac 使用的 zsh 就添加到~/.zshrc 里即可
  添加后 source ~/.zshrc后生效

## 准备集群 kubeconfig

从集群.kube/config 文件保存到本地机器,比如

```text
kubeconfig-cluster-a
kubeconfig-cluster-b
```

## 部署 Submariner Broker

Submariner 需要一个 Broker（通常部署在其中一个集群上）来协调跨集群通信。

* 在集群 A 上部署 Broker：

```text
subctl deploy-broker --kubeconfig kubeconfig-cluster-a
```

输出

```text
 ✓ Setting up broker RBAC 
 ✓ Deploying the Submariner operator 
 ✓ Created operator CRDs
 ✓ Created operator namespace: submariner-operator
 ✓ Created operator service account and role
 ✓ Created submariner service account and role
 ✓ Created lighthouse service account and role
 ✓ Deployed the operator successfully
 ✓ Deploying the broker 
 ✓ Saving broker info to file "broker-info.subm" 
```

* 这会创建一个命名空间 submariner-k8s-broker，并生成 Broker 的配置文件（稍后用于连接其他集群）。
  在集群发现这个 ns 下创建一个 operator，由deployment 和 service组成
  ```text
  NAME                                       READY   STATUS    RESTARTS   AGE
  pod/submariner-operator-747cc649f8-pqd69   1/1     Running   0          6m20s

  NAME                                  TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
  service/submariner-operator-metrics   ClusterIP   10.233.21.162   <none>        8383/TCP   5m59s

  NAME                                  READY   UP-TO-DATE   AVAILABLE   AGE
  deployment.apps/submariner-operator   1/1     1            1           6m20s

  NAME                                             DESIRED   CURRENT   READY   AGE
  replicaset.apps/submariner-operator-747cc649f8   1         1         1       6m20s
  ```

## 加入集群到 Submariner

将两个集群加入 Submariner，使它们能够相互通信。

请确保集群间无防火墙规则或者保证集群间放行 icmp 和 udp 500,4500,4490 端口。

* 集群A安装

  ```text
  subctl join --kubeconfig kubeconfig-cluster-a broker-info.subm --clusterid cluster-a --check-broker-certificate=false
  ```

  输出
  ```text
    ✓ broker-info.subm indicates broker is at https://hk.chenjie.info:6443
    ✓ Discovering network details
    Network plugin:  calico
    Service CIDRs:   [10.244.0.0/18]
    Cluster CIDRs:   [10.244.64.0/18]
    ✓ Retrieving the gateway nodes
    ✓ Retrieving all worker nodes
    ✓ Labeling node "hk4" as a gateway
    ✓ Gathering relevant information from Broker 
    ✓ Retrieving Globalnet information from the Broker
    ✓ Validating Globalnet configuration
    ✓ Retrieving ClustersetIP information from the Broker
    ✓ Validating ClustersetIP configuration
    ✓ Assigning ClustersetIP IPs
    ✓ Allocated clustersetip CIDR 243.0.0.0/20
    ✓ Updating the ClustersetIP information on the Broker
    ✓ Deploying the Submariner operator
    ✓ Created operator namespace: submariner-operator
    ✓ Creating SA for cluster
    ✓ Connecting to Broker
    ✓ Deploying submariner
    ✓ Submariner is up and running
  ```
* 集群B安装

  ```text
  subctl join --kubeconfig kubeconfig-cluster-b broker-info.subm --clusterid cluster-b  --check-broker-certificate=false
  ```

## 配置网关节点

Submariner 需要在每个集群中选择至少一个节点作为网关节点，用于处理跨集群流量。

* 默认情况下，Submariner 会自动选择一个节点。如果需要手动指定，可以给节点打上标签：

  ```text
  kubectl label nodes <node-name> submariner.io/gateway=true
  ```
* 检查集群连接：

  ```text
  subctl show all
  ```

  输出
  ```
  Cluster "cluster.local"
  ✓ Detecting broker(s)
  NAMESPACE               NAME                COMPONENTS                        GLOBALNET   GLOBALNET CIDR   DEFAULT GLOBALNET SIZE   DEFAULT DOMAINS   
  submariner-k8s-broker   submariner-broker   service-discovery, connectivity   no          242.0.0.0/8      65536

  ✓ Showing Connections
  GATEWAY   CLUSTER     REMOTE IP        NAT   CABLE DRIVER   SUBNETS                         STATUS    RTT avg.   
  hk4       cluster-b   45.121.215.112   yes   libreswan      10.244.0.0/18, 10.244.64.0/18   connected 3.640778ms 

  ✓ Showing Endpoints
  CLUSTER     ENDPOINT IP      PUBLIC IP        CABLE DRIVER   TYPE   
  cluster-a   83.229.122.142   83.229.122.142   libreswan      local  
  cluster-b   10.0.2.2         45.121.215.112   libreswan      remote

  ✓ Showing Gateways
  NODE   HA STATUS   SUMMARY                                
  hk     active      0 connections out of 1 are established

  ✓ Showing Network details
  Discovered network details via Submariner:
  Network plugin:  calico
  Service CIDRs:   [10.233.0.0/18]
  Cluster CIDRs:   [10.233.64.0/18]
  ClustersetIP CIDR:     243.0.0.0/20

  ✓ Showing versions
  COMPONENT                       REPOSITORY           CONFIGURED   RUNNING                     ARCH  
  submariner-gateway              quay.io/submariner   0.20.0       release-0.20-f0a5355cabfc   amd64   
  submariner-routeagent           quay.io/submariner   0.20.0       release-0.20-f0a5355cabfc   amd64   
  submariner-metrics-proxy        quay.io/submariner   0.20.0       release-0.20-8fde9372397b   amd64   
  submariner-operator             quay.io/submariner   0.20.0       release-0.20-44970648cf5c   amd64   
  submariner-lighthouse-agent     quay.io/submariner   0.20.0       release-0.20-c9e76a4aee91   amd64   
  submariner-lighthouse-coredns   quay.io/submariner   0.20.0       release-0.20-c9e76a4aee91   amd64
  ```

## 测试通信

* 在集群 A 中部署一个简单的 Nginx Pod 和 Service：

  ```
  kubectl apply -f deploy.yaml
  ```
* 导出 service

  ```
  subctl export service nginx-service
  ```

  输出

  ```
  ✓ Service exported successfully
  ```
* 在集群 B 中测试访问：

  ```
  kubectl run -it --rm test-pod --image=nginx --restart=Never -- curl nginx-service.default.svc.clusterset.local
  ```


## 原理

### 核心组件

Submariner 的功能依赖于几个关键组件：

1. **Broker**：一个中央协调点，负责存储和同步集群间的元数据（如 Pod CIDR、Service CIDR 等）。
2. **Gateway Engine**：运行在每个集群的网关节点上，负责建立和管理跨集群的网络隧道。
3. **Route Agent**：运行在每个集群的节点上，负责更新节点的路由表，确保跨集群流量正确转发。
4. **Service Discovery（Lighthouse）**：提供跨集群的 Service 发现机制，使 Service 可以通过统一的 DNS 名称访问。


### 工作原理

#### 1. **集群注册与元数据同步**

* **Broker 的作用**：
  * Submariner 在一个集群中部署 Broker（通常是第一个加入的集群），Broker 是一个 CRD（自定义资源定义）存储，记录所有加入 Submariner 的集群信息。
  * 每个集群在加入时会向 Broker 注册，提交自己的集群 ID、Pod CIDR、Service CIDR 和网关节点的公共 IP 等信息。
* **同步机制**：
  * Broker 通过 Kubernetes 的 API 机制将这些信息同步到其他集群，确保所有集群知道彼此的网络范围和访问入口。

#### 2. **网络隧道的建立**

* **网关节点的选择**：

  * 每个集群中至少有一个节点被指定为网关节点（通过标签 **submariner.io/gateway=true**），负责处理跨集群流量。
* **隧道技术**：

  * Gateway Engine 使用隧道协议（如 VXLAN 或 IPsec）在网关节点之间建立连接。
  * VXLAN：将 Pod 的流量封装成 UDP 数据包，通过网关节点传输到目标集群。
  * IPsec（可选）：提供加密通道，确保跨集群通信的安全性。
* **流量路径**：

  * 当集群 A 的 Pod A 需要访问集群 B 的 Pod B 时，流量会从 Pod A 的节点路由到集群 A 的网关节点，通过隧道传输到集群 B 的网关节点，再转发到 Pod B。

#### 3. **路由管理**

* **Route Agent 的作用**：
  * 在每个集群的每个节点上运行的 Route Agent 负责动态更新节点路由表。
  * 当 Broker 同步了其他集群的 Pod CIDR 后，Route Agent 会将这些 CIDR 添加到本地节点的路由规则中，指向本集群的网关节点。
* **路由过程**：
  * 假设集群 A 的 Pod CIDR 为 **10.233.0.0/18**，集群 B 的 Pod CIDR 为 **10.244.0.0/18**。
  * 集群 A 的节点路由表会添加规则：**10.244.0.0/18 -> 集群 A 的网关节点**。
  * 流量到达网关节点后，通过隧道转发到集群 B 的网关节点，最终到达目标 Pod。

#### 4. **Service 发现（Lighthouse）**

* **全局 Service 同步**：
  * Submariner 的 Lighthouse 组件通过监听 Kubernetes Service 资源，将导出的 Service 信息同步到其他集群。
  * 例如，集群 A 的 **nginx-service** 被导出后，Lighthouse 在集群 B 创建对应的 DNS 记录（如 **nginx-service.default.svc.clusterset.local**）。

* **DNS 解析**： 
  * 当集群 B 的 Pod 访问 **nginx-service.default.svc.clusterset.local** 时，Lighthouse 将其解析为集群 A 的 Service IP。
  * 流量随后通过网关节点和隧道转发到集群 A 的 Service。
  
```text
集群 A                                    集群 B
+---------------+                        +---------------+
| Pod A         |                        | Pod B         |
| (10.233.1.10) |                        | (10.244.1.20) |
+-------|-------+                        +-------|-------+
|                                        |
+-------v-------+                        +-------v-------+
| Worker Node   |                        | Worker Node   |
| Route:        |                        | Route:        |
| 10.244.0.0/18 |------------------------| 10.233.0.0/18 |
| -> Gateway    |                        | -> Gateway    |
+-------|-------+                        +-------|-------+
|                                        |
+-------v-------+    VXLAN/IPsec 隧道    +-------v-------+
| Gateway Node A| <--------------------->| Gateway Node B|
+---------------+                        +---------------+
```

## 参考
* https://xie.infoq.cn/article/2dfcd9239b38a2a80c51267e2