# [WIP] TiDB Operator 学习 - ControllerManager 概述


## 1 概述
TiDB Operator 中的 ControllerManager 用于管理 TiDBCluster 这个 CustomResource。
> 下面 ControllerManager 都是指的 TiDB Operator 的 ControllerManager。

TiDBCluster 是一个抽象的概念，其稳定运行需要保证许多核心的组件的运行，包括：
* **Discovery** ：用于 PD 服务发现的组件，会根据 Discovery 返回的信息来拼接 PD 的启动参数。
* **PD** ：TiDB Cluster 的元信息管理模块与调度层，需要优先于其他所有组件（除了 Discovery）先启动。
* **TiKV** ：存储节点，每个节点也被称为 Store。
* **TiDB** ：SQL 层，对外暴露 MySQL 协议的连接入口，执行 SQL 解析与优化，生成分布式执行计划。
* **TiFlash**（可选）：特殊存储节点，与 TiKV 存储相同数据，但是以列式的形式存储，为支持 OLAP 而实现。
* **TiCDC**（可选）：数据增量备份工具，拉取 TiKV 变更日志并将数据还原到任意 TSO。
* **Pump**（可选）：收集 TiDB 的 binlog，并通过 Drainer 输出到下游。

要明确，ControllerManager 管理的是 TiDBCluster 这个自定义资源，而 TiDB Cluster 的稳定运行需要上述组件的稳定运行。

因此，整个 Controller Manager 宏观来看就是周期性执行 TiDBCluster  Sync，而此 TiDBCluster Sync 中会去执行所有的 Component Sync。
{{< admonition note "为什么叫 ControllerManager？">}}
猜想是因为 TiDBCluster 的运行要包含许多组件：例如核心的 PD TiDB TiCluster，每个组件有着一个  Controller 运行协调操作，所以整体的就称为 Controller Manager。
{{< /admonition >}}


## 2 背景知识
### 2.1 Kubernetes Controller 模式

虽然叫做 Controller Manager，但是其实使用的还是普通的 Controller 的模式：通过 Informer 监听 TiDBCluster 资源，通过 Queue 进行解耦，通过 ControlLoop 进行协调操作。
{{< find_img "img1.png" >}}

	
对应资源的 Informer 通过 Watch APIServer 监听着资源的事件，发生事件时调用 AddFunc UpdateFunc DeleteFunc。

同时，Informer 也会周期性的进行 List 更新本地缓存，并且主动调用一次 UpdateFunc。

AddFunc UpdateFunc DeleteFunc 实现中会将资源对象的 Key 放入 WorkQueue。

ControlLoop 不断轮询消费着 WorkQueue，取出 Key 后从 APIServer 得到对应资源信息，并进行 Sync 操作。


## 3 启动


## 4 Member Manager

### 4.1 Discovery
Discovery 是最先运行的组件，因为其核心组件 PD 需要使用 Discovery 来进行服务发现。

依赖组件：无
#### 4.1.1 资源组成
Discovery 的运行需要以下的资源对象：

Resource | Name | Used for
-|-|-
Role | \<cluster-name>-discovery | 提供查询 Secret 对象的权限、
RoleBinding | \<cluster-name>-discovery | 绑定 Role 与 ServiceAccount
ServiceAccount | \<cluster-name>-discovery | 需要查询 Secret 对象的权限 
ClusterIP Service | \<cluster-name>-discovery | 用于 PD 访问 Discovery
Deployment | \<cluster-name>-discovery | 控制一个 Pod 

### 4.2 PD
在 Discovery Pod 运行后，接着要启动的就是 PD 组件。而后其他组件之间没有强烈的依赖关系了。

依赖组件： Discovery
#### 4.2.1 资源组成
Resource | Name | Used for
-|-|-
Headless Service |	\<cluster-name>-pd-peer | 分布式系统，需要 Pod 间访问
ClusterIP Service |	\<cluster-name>-pd | 为集群内部提供组件查询与控制接口
StatefulSet | \<cluster-name>-pd |	使用 StatefulSet 来管理 Pod	
ConfigMap（可选）| \<cluster-name>-pd-\<digest> | 为 Pod 提供配置参数，设置 `spec.pd.config` 则创建

### 4.3 TiKV
TiKV 用于存储实际的数据库数据，多个 TiKV Pod 组成了一个分布式 KV 存储。

依赖组件：PD
#### 4.3.1 资源组成
Resource | Name | Used for
-|-|-
Headless Service|	\<cluster-name>-tikv-peer |分布式系统，需要 Pod 间访问
StatefulSet | \<cluster-name>-tikv |使用 StatefulSet 来管理 Pod。
ConfigMap（可选）| \<cluster-name>-pd-\<digest> | 为 Pod 提供配置参数，设置 `spec.tikv.config` 则创建

### 4.4 TiDB
TiDB 是无状态的 SQL 层，也是对外提供数据库服务的入口。

依赖组件：TiKV、Pump（如果使用）
#### 4.3.1 资源组成
Resource | Name | Used for
-|-|-
Headless Service | \<cluster-name>-tidb-peer | ？？
NodePort Service <br> 或者 <br> LoadBalance Service | \<cluster-name>-tidb | 对外提供数据库服务
StatefulSet | \<cluster-name>-tidb | 使用 StatefulSet 来管理 Pod	
ConfigMap（可选）| \<cluster-name>-tidb-\<digest> | 为 Pod 提供配置参数，设置 `spec.tidb.config` 则创建
Secret（可选） | \<cluster-name>-tidb-server-secret | 为开启 TLS 使用的根证书与私钥，设置 `spec.tlsCluster` 则创建

### 4.5 TiFlash
依赖组件：PD
#### 4.5.1 资源组成
Resource | Name | Used for
-|-|-
Headless Service | \<cluster-name>-tiflash-peer |
StatefulSet| \<cluster-name>-tiflash | 使用 StatefulSet 来管理 Pod	
ConfigMap（可选）| \<cluster-name>-tiflash-\<digest> | 为 Pod 提供配置参数，设置 `spec.tiflash.config` 则创建

### 4.6 Pump
依赖组件：无
#### 4.6.1 资源组成
Resource | Name | Used for
-|-|-
Headless Service | \<cluster-name>-pump | -
StatefulSet | \<cluster-name>-pump | 使用 StatefulSet 来管理 Pod	
ConfigMap（可选）| \<cluster-name>-pump-\<digest> | 为 Pod 提供配置参数，设置 `spec.pump.config` 则创建

{{< admonition bug WARN>}}
Pump 的 Headless Service 命名应该是个 bug。
{{< /admonition >}}

### 4.7 TiCDC
依赖组件 ：PD
#### 4.7.1 资源组成
Resource | Name | Used for
-|-|-
Headless Service | \<cluster-name>-ticdc |
StatefulSet | \<cluster-name>-ticdc | 使用 StatefulSet 来管理 Pod	
ConfigMap（可选）| \<cluster-name>-ticdc-\<digest> | 为 Pod 提供配置参数，设置 `spec.ticdc.config` 则创建


## 5 整体框架


