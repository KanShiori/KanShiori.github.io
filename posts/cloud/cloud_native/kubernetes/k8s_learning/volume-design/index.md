# Kubernetes - 存储设计


## 1 Volume 基本概念

为了支持多种多样的存储，Kubernetes 抽象出了 Volume 的概念，规范了 Volume 的操作流程与接口。而各个存储插件都围绕着 Volume 实现对应的功能。

Kubernetes 把 Volume 的操作分解为了四种操作：

{{< image src="img1.png" height=400 >}}

* <important>Provision / Delete</important>
  
  **创建与删除 Volume**，也就是创建或删除实际的 Block Device。

  这时 Volume 的来源、容量、性能等参数是确定的。

  例如，AWS 创建与删除 EBS。

* <important>Attach / Detach</important>
  
  **将 Volume 接入到 Node 中，使得 Node 上可以看见对应的 Block Device**。

  这时 Volume 的设备名称、驱动等面向操作系统的信息是确定的。

  例如，AWS Attach 与 Detach EBS 到 EC2 Instance 上。

* <important>Mount / Unmount</important>

  **将 Volume 格式化并挂载到 Node Dir 上，使得 Volume 在 Node 上是可访问的了。**

  这时，Volume 的文件系统、访问目录等信息是确定的了。

  例如，Mount 挂载 Block Device。

* <important>SetUp / TearDown</important>
  
  **将 Volume Node Dir 挂载到容器中，使得 Volume 在容器中可访问。**

  例如，使用 Bind Mount 挂载目录到容器中。

另外，对于一些特殊类型的 Volume（例如 hostpath、secret、configmap 等），因为本身不需要 Attach 等流程，仅仅实现了 SetUp 步骤。

### 1.1 构建流程

下面以 PVC 为例，更加细节的说一下 Volume 的构建流程：

1. **Deploy PVC**
   
   用户 Deploy PVC 对象。

2. **Provision Volume**
   
   PV Controller 基于 StorageClass 创建一个 Volume（无法绑定已有 PV 的情况下），并创建 PV 对象。
   
3. **Bind PV** 
   
   PV Controller 将 PV 与 PVC 绑定。
   
4. **Attach Volume**
   
   Controller Manager 中的 AttachDetach Controller 将 Attach Volume 到 Node 上，Node 上看来是一个块设备。

5. **Mount Device**
   
   Kubelet 中的 Volume Manager 将块设备 Mount 到一个全局的目录。
   
6. **Setup Volume**
   
   Kubelet 中的 Volume Manager 准备 Pod 特定的目录，该目录后续用于挂载到各个容器中的目录。

## 2 Volume Plugin

[**Volume 基本概念**](#1-volume-基本概念) 中所述的流程是 Kubernetes 作为控制面的流程，到具体调用 Provision Attach 等操作时，都会调用 Volume 对应的 Plugin 的实现。

目前 Kubernetes 的 Volume Plugin 可以分为三类：

* **内置 Volume Plugin**
  
  包括 ConfigMap、HostPath 等内置的存储类型，以及一些云厂商的云存储的，其 Volume Plugin 实现是在 Controller Manager 源码中的。
  
  这些称为 In-Tree Plugin，意思是在 Kubernetes 代码树中。

* **CSI (Container Storage Interface)**
  
  可扩展的接口，Volume 的四个操作被由不同的 Controller 负责：

    * Provision / Delete - CSI Plugin Watch PVC 资源并处理
    * Attach / Detach - CSI Plugin Watch VolumeAttachment 资源并处理
    * Mount / Unmount - Kubelet 调用 CSI Plugin 的 RPC 接口处理
    * SetUp / TearDown - Kubelet 处理 CSI Plugin 的 RPC 接口处理

* **FlexVolume** <important>(deprecated)</important>
  
  可扩展的接口，通过可执行文件进行交互，Controller Manager 调用可执行文件的命令来操作 Volume。

## 3 CSI 基本架构

对于 Kubernetes 层面，CSI Plugin 是一个 Controller + RPC Service：

{{< image src="img2.png" height=430 >}}

* **Controller** - 负责 Volume 的 Provision/Delete + Attach/Detach 操作，这时 Kubernetes 与 Controller 通过 PVC 与 VolumeAttachment 两个资源进行交互
* **RPC Service** - 提供给 Volume 的 Mount/Unmount 接口，Kubelet 通过调用 Mount/Unmount 接口与 RPC Service 交互。

为了简化 CSI Plugin 的实现复杂度，Kubernetes 提供了一些通用的容器，用以实现 Controller 大部分的控制逻辑，用户只需要实现更加底层的 Volume 操作逻辑即可。

CSI Plugin 的架构如下：

{{< image src="img3.png" height=300 >}}

* <important>Sidecar</important> - 一组实现了 CSI Plugin 的通用控制逻辑的容器
  
  例如 Watch PVC/VolumeAttachment 以触发 Volume 操作。
  
  这些容器由 Kubernetes 社区维护，以减少实现 CSI 的重复代码。

* <important>Driver</important> - 一个 gRPC Service，对外提供的三个 Service：

  * <important>CSI Identity</important> - 提供查询 CSI 插件信息的接口；
  * <important>CSI Controller</important> - 提供对 Volume 的管理接口；
  * <important>CSI Node</important> - 提供 Node 上操作接口；
  
  每个 Service 提供了具体 Volume 操作的接口，提供给 Sidecar 与 Kubelet 调用。

可以看到，这样用户只需要实现一个 CSI Driver 就可以构建一个 CSI Plugin 了。

### 3.1 Sidecar 容器

Sidecar 容器会包含<important> Watch 资源的工作，并执行 CSI Driver 的回调函数，然后更新相关的资源</important>。

通常，这些容器用于与 CSI Driver 的容器捆绑，作为一个 Pod 部署在一起。

* [**node-driver-registrar**](https://github.com/kubernetes-csi/node-driver-registrar)
  
  容器调用 CSI Node 的 `NodeGetInfo()` 接口，**将 CSI Node 注册到 Kubelet**。

* [**external-provisioner**](https://github.com/kubernetes-csi/external-provisioner)

  容器会 Watch `PVC` 资源，并调用 CSI Controller 的 `CreateVolume()` 接口来**创建一个 Volume**。

* [**external-attacher**](https://github.com/kubernetes-csi/external-attacher)
  
  容器会 Watch `VolumeAttachment` 资源，并调用 CSI Controller 的 `ControllerPublishVolume()` 与 `ControllerUnpublishVolume()` 接口，**将 Volume Attach 到指定的 Node**。

  {{< admonition note Note>}}
  external-attacher 仅仅用于 Attach 操作，而 Volume 的 MountDevice 与 Setup 操作，都是由 Kubelet 直接调用 CSI Node 插件的接口实现的。
  {{< /admonition >}}

* [**external-snapshotter**](https://github.com/kubernetes-csi/external-snapshotter)
  
  snapshot controller 容器会 Watch `VolumeSnapshot` 与 `VolumeSnapshotContent` 资源，snapshotter 容器会 Watch `VolumeSnapshotConent` 资源，并负责调用 CSI Controller 插件的 `CreateSnapshot()`、`DeleteSnapshot()` 与 `ListSnapshot()` 接口。

* [**external-resizer**](https://github.com/kubernetes-csi/external-resizer)
  
  容器 Watch `PersistentVolumeClaim` 资源的修改事件，当用户请求扩容 PVC 时，调用 CSI Controller 的 `ControllerExpandVolume()` 接口**扩容 Volume**。

* [**livenessprobe**](https://github.com/kubernetes-csi/livenessprobe)
  
  容器会监控 CSI 插件的监控状态，并通过 Liveness Probe 提供给 Kubernetes。

{{< admonition note Note>}}
查看官方文档 [**Kubernetes CSI Sidecar Containers**](https://kubernetes-csi.github.io/docs/sidecar-containers.html) 来了解完整的 Sidecar 容器。
{{< /admonition >}}

### 3.2 CSI Identity 服务

**`CSI Identity`** 服务用于**提供插件的一些信息**。

```protobuf
service Identity {
  // 返回插件的 name 与 version
  rpc GetPluginInfo(GetPluginInfoRequest)
    returns (GetPluginInfoResponse) {}
  // 得到插件拥有的功能
  rpc GetPluginCapabilities(GetPluginCapabilitiesRequest)
    returns (GetPluginCapabilitiesResponse) {}
  // 查询插件是否正在运行中
  rpc Probe (ProbeRequest)
    returns (ProbeResponse) {}
}
```

### 3.3 CSI Controller 服务

**`CSI Controller`** 服务**定义了对 Volume 的管理接口**，包括：Provision/Delete、Attach/Detach、Snapshot 等操作。这些操作都是属于 Controller 的操作，在 Master 节点上执行。

```protobuf
service Controller {
  // 创建一个 Volume
  rpc CreateVolume (CreateVolumeRequest)
    returns (CreateVolumeResponse) {}
  // 删除一个 Volume
  rpc DeleteVolume (DeleteVolumeRequest)
    returns (DeleteVolumeResponse) {}
  // Attach Volume 到指定的 Node
  rpc ControllerPublishVolume (ControllerPublishVolumeRequest)
    returns (ControllerPublishVolumeResponse) {}
  // Detach Volume
  rpc ControllerUnpublishVolume (ControllerUnpublishVolumeRequest)
    returns (ControllerUnpublishVolumeResponse) {}
  // 检查一个 Volume 的能力
  rpc ValidateVolumeCapabilities (ValidateVolumeCapabilitiesRequest)
    returns (ValidateVolumeCapabilitiesResponse) {}
  // 查询已经创建的所有的 Volume
  rpc ListVolumes (ListVolumesRequest)
    returns (ListVolumesResponse) {}
  // 查询存储池的容量
  rpc GetCapacity (GetCapacityRequest)
    returns (GetCapacityResponse) {}
  // 查询 Controller 支持的功能
  rpc ControllerGetCapabilities (ControllerGetCapabilitiesRequest)
    returns (ControllerGetCapabilitiesResponse) {}
  // 创建一个 Snapshot
  rpc CreateSnapshot (CreateSnapshotRequest)
    returns (CreateSnapshotResponse) {}
  // 删除一个 Snapshot
  rpc DeleteSnapshot (DeleteSnapshotRequest)
    returns (DeleteSnapshotResponse) {}
  // 查询所有创建的 Snapshot
  rpc ListSnapshots (ListSnapshotsRequest)
    returns (ListSnapshotsResponse) {}
  // 扩容一个 Volume
  rpc ControllerExpandVolume (ControllerExpandVolumeRequest)
    returns (ControllerExpandVolumeResponse) {}
  // 查询一个 Volume
  rpc ControllerGetVolume (ControllerGetVolumeRequest)
    returns (ControllerGetVolumeResponse) {
        option (alpha_method) = true;
    }
}
```

### 3.4 CSI Node 服务

**`CSI Node`** 提供**在 Node 上 Volume 的操作**，包括 MountDevice 与 Setup 等操作。

```protobuf
service Node {
  // 将 Volume 挂载到一个全局目录，即 MountDevice 操作
  rpc NodeStageVolume (NodeStageVolumeRequest)
    returns (NodeStageVolumeResponse) {}
  // 将 Volume 取消挂载全局目录
  rpc NodeUnstageVolume (NodeUnstageVolumeRequest)
    returns (NodeUnstageVolumeResponse) {}
  // 将 Volume 挂载到 Pod 目录，即 Setup 操作
  rpc NodePublishVolume (NodePublishVolumeRequest)
    returns (NodePublishVolumeResponse) {}
  // 将 Volume 取消挂载 Pod 目录
  rpc NodeUnpublishVolume (NodeUnpublishVolumeRequest)
    returns (NodeUnpublishVolumeResponse) {}
  // 返回 Volume 信息
  rpc NodeGetVolumeStats (NodeGetVolumeStatsRequest)
    returns (NodeGetVolumeStatsResponse) {}
  // 扩容 Volume
  rpc NodeExpandVolume(NodeExpandVolumeRequest)
    returns (NodeExpandVolumeResponse) {}
  // 返回 Node 插件功能，例如是否只是 stage/unstage
  rpc NodeGetCapabilities (NodeGetCapabilitiesRequest)
    returns (NodeGetCapabilitiesResponse) {}
  // 返回插件对应的节点信息
  rpc NodeGetInfo (NodeGetInfoRequest)
    returns (NodeGetInfoResponse) {}
}
```

## 4 CSI 相关资源对象

### 4.1 CSIDriver

**`CSIDriver`** 用于**让 Kubernetes 发现该 CSI 插件，并知晓其拥有的功能**。

```yaml
apiVersion: storage.k8s.io/v1
kind: CSIDriver
metadata:
  name: mycsidriver.example.com
spec:
  attachRequired: true
  podInfoOnMount: true
  fsGroupPolicy: File # added in Kubernetes 1.19, this field is GA as of Kubernetes 1.23
  volumeLifecycleModes: # added in Kubernetes 1.16, this field is beta
    - Persistent
    - Ephemeral
  tokenRequests: # added in Kubernetes 1.20. See status at https://kubernetes-csi.github.io/docs/token-requests.html#status
    - audience: "gcp"
    - audience: "" # empty string means defaulting to the `--api-audiences` of kube-apiserver
      expirationSeconds: 3600
  requiresRepublish: true # added in Kubernetes 1.20. See status at https://kubernetes-csi.github.io/docs/token-requests.html#status
```

相关字段含义如下：

* `name` - CSI Driver 的名字
* [**attachRequired**](https://kubernetes-csi.github.io/docs/skip-attach.html) - 表示 Volume 是否需要 Attach 操作（Kubernetes 是否需要创建 VolumeAttachment 资源）
* [**podInfoOnMount**](https://kubernetes-csi.github.io/docs/pod-info.html) - 表明当调用 Mount 操作时，是否需要传递额外的 Pod 信息给 CSI Driver
* [**fsGroupPolicy**](https://kubernetes-csi.github.io/docs/support-fsgroup.html) - 表示 CSI Driver 是否支持：当 Mount Volume 时，修改 Volume 的所有权与权限
* `volumeLifecycleModes` - Volume 支持的模式
* `tokenRequests` - Kubelet 调用 CSI Node 的 NodePublishVolume() 接口时是否需要提供 Pod 的 ServiceAccount
* `requiresRepublish` - 为 true 时，Kubelet 会周期性调用 CSI Node 的 NodePublishVolume() 接口

### 4.2 CSINode

CSI Driver 可以生成特定的关于 Node 的信息，将其保存到 **`CSINode`** 对象中，而不用将信息保存到 Kubernetes Node 对象中。Kubernetes 会创建 `CSINode` 对象，并根据已经注册的 CSIDriver 来填充 `CSINode` 信息。

```yaml
kind: CSINode
metadata:
  name: kube-node-5
  ownerReferences:
  - apiVersion: v1
    kind: Node
    name: kube-node-5
    uid: f442c38a-f0f3-4a79-8153-3e5bcd7c91a4
spec:
  drivers:
  - name: hostpath.csi.k8s.io
    nodeID: kube-node-5
    topologyKeys:
    - topology.hostpath.csi/node
```
* drivers - 该 Node 上运行的所有 CSI Driver
  * name - CSI Driver 的名字
  * nodeID - Driver 提供的该 Node ID
  * [**topologyKeys**](https://kubernetes-csi.github.io/docs/topology.html) - 针对于该 CSI 的 Topology Key

### 4.3 PersistentVolumeClaim

当用户部署对应 CSI 插件的 PVC 时，由 CSI Controller 来负责创建对应的 PV 并绑定到指定的 PVC。等待 PVC 与 PV 相互绑定后，Kubernetes 就继续进行 Attach 等流程了。

PV 定义中使用 `spec.csi` 字段来表明该 PV 是 CSI 提供的。
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv0003
spec:
  # ...
  csi:
    driver: hostpath.csi.k8s.io
    volumeHandle: 5a760aab-d579-11eb-ba5c-fa03df0e077d
    readonly: false
    fsType: Filesystem
    volumeAttributes:
      kind: standard
      storage.kubernetes.io/csiProvisionerIdentity: 1624246373313-8081-hostpath.csi.k8s.io-kube-node-5
    # controllerPublishSecretRef: []
    # nodeStageSecretRef: []
    # nodePublishSecretRef: []
```
* driver - CSI Driver 的名字
* volumeHandle - 插件提供的 Volume ID，用于 Kubernetes 与插件共同标识该 Volume
* readonly - 表明 Volume 是否是只读
* fsType - Volume 类型
* volumeAttributes - Volume 的一些元信息，将通过 volume_context 传递给 CSI 插件
* controllerPublishSecretRef - 调用 CSI Controller 的 ControllerPublishVolume() 与 ControllerUnpublishVolume() 时传递的 Secret 对象
* nodeStageSecretRef - 调用 CSI Node 的 NodeStageVolume() 时传递的 Secret 对象
* nodePublishSecretRef - 调用 CSI Node 的 NodePublishVolume() 时传递的 Secret 对象

### 4.4 VolumeAttachment

当 Kubernetes 需要 Attach Volume 时，其 Kubernetes 会创建 `VolumeAttachment` 资源表示需要进行 Attach，而 **CSI Controller 需要监听 VolumeAttachment 资源然后进行实际的 Attach 操作**。

```yaml
apiVersion: storage/v1
kind: VolumeAttachment
metadata:
  name: pv0003
spec:
  attacher: www.example.csi.com
  nodeName: node-1
  source:
    persistentVolumeName: pvc0003
    inlineVolumeSpec:
        # pvc spec
status:
  attached: true
  attachmentMetadata: {}
#   attachError:
#     time:
#     message:
#   detachError:
#     time:
#     message:
```
* spec
  * attacher - 需要执行 Attach 操作的 CSI Driver
  * nodeName - 目标 Node
  * source
    * persistentVolumeName - 指定被 Attach 的 PVC
    * inlineVolumeSpec - 使用内联的 PVC
* status
  * attached - 表明是否成功 Attach
  * attachmentMetadata - Attach 成功后填充的一些元信息，以提供给后续的操作
  * attachError - 上一次 Attach 失败信息
  * detachError - 上一次 Detach 失败信息





## 参考

* [**Blog：详解 Kubernetes Volume 的实现原理**](https://draveness.me/kubernetes-volume/)
* [**Blog：kubelet volume manager 源码分析**](http://bingerambo.com/posts/2021/02/kubelet-volume-manager%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90/)
* [**CSI Proto**](https://github.com/container-storage-interface/spec/blob/master/csi.proto)

