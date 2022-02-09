# K8s 学习 - 11 - CSI 概念与实现


## 1 设计原理

在 [**源码阅读 - Volume 实现**](../volume-implementation) 中看到，所有的存储插件实现的都是操作 Volume 的方法：
* Provision Volume
* Attach Volume
* Mount Device
* Setup Volume

CSI 整体体系的架构如下图
{{< find_img "img1.png" >}}

* **`Sidecar 容器`** 是一组 Kubernetes 社区维护的标准容器，以减少实现 CSI 的重复代码。
* **`CSI Driver`** 就是我们需要编写的 CSI 插件了，一个 CSI 插件只有一个二进制文件，以 gRPC 方式对外提供三个服务：
  * **`CSI Identity`**
  * **`CSI Controller`**
  * **`CSI Node`**

### 1.1 Sidecar Containers

Sidecar 容器会包含<important> Watch 资源的工作，并执行 CSI Driver 的回调函数，然后更新相关的资源</important>。

通常，这些容器用于与 CSI Driver 的容器捆绑，作为一个 Pod 部署在一起。

目前开发的 CSI Sidecar 容器包括：
* [**node-driver-registrar**](https://github.com/kubernetes-csi/node-driver-registrar)
  
  容器调用 CSI Node 的 `NodeGetInfo()` 接口，**将 CSI Node 注册到 Kubelet**。

* [**external-provisioner**](https://github.com/kubernetes-csi/external-provisioner)

  容器会 Watch `PersistentVolumeClaim` 资源，并调用 CSI Controller 的 `CreateVolume()` 接口来**创建一个 Volume**。

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

### 1.2 CSI Driver

CSI Driver 具体实现了对 Volume 的所有操作，本质上需要实现一个 gRPC Server，由 Kubernetes 调用具体的接口。

{{< admonition note Note>}}
具体的协议定义见 [**csi.proto**](https://github.com/container-storage-interface/spec/blob/master/csi.proto)
{{< /admonition >}}

#### 1.2.1 CSI Identity

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

#### 1.2.2 CSI Controller

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

#### 1.2.3 CSI Node

**`CSI Node`** 提供**在 Node 上的操作**，包括 MountDevice 与 Setup 等操作。
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

## 2 CSI 相关资源对象

### 2.1 CSIDriver

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
* `name` - CSI Driver 的名字
* [**attachRequired**](https://kubernetes-csi.github.io/docs/skip-attach.html) - 表示 Volume 是否需要 Attach 操作（Kubernetes 是否需要创建 VolumeAttachment 资源）
* [**podInfoOnMount**](https://kubernetes-csi.github.io/docs/pod-info.html) - 表明当调用 Mount 操作时，是否需要传递额外的 Pod 信息给 CSI Driver
* [**fsGroupPolicy**](https://kubernetes-csi.github.io/docs/support-fsgroup.html) - 表示 CSI Driver 是否支持：当 Mount Volume 时，修改 Volume 的所有权与权限
* `volumeLifecycleModes` - Volume 支持的模式
* `tokenRequests` - Kubelet 调用 CSI Node 的 NodePublishVolume() 接口时是否需要提供 Pod 的 ServiceAccount
* `requiresRepublish` - 为 true 时，Kubelet 会周期性调用 CSI Node 的 NodePublishVolume() 接口

### 2.2 CSINode

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

### 2.3 PersistentVolumeClaim

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

### 2.4 VolumeAttachment

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


## 3 CSI 实现

在 Kubernetes 实现中，CSI VolumePlugin 实现了 VolumePlugin 接口，提供了 Attach 与 SetUp 等操作，其本质就是创建资源或者调用特定 CSI 插件的接口，类似于一个中间层。

CSI 插件不需要注册到 Controller 层中，相互之间完全围绕着 Kubernetes 资源进行交互。

### 3.1 Controller 层实现

从上面可以看到，Controller 层主要执行了 Create/Delete、Attach/Detach 和 Snapshot 操作。Create/Delete 操作是由 CSI Controller 插件监听并执行，因此不需要 Kubernetes 中进行代码实现。

而 Attach/Detach 和 Snapshot 操作就是在 VolumePlugin 接口实现上，创建对应的 VolumeAttachment 和 VolumeSnapshot 等资源，然后等待 CSI Controller 执行操作。

```go
type csiPlugin struct {
	host                      volume.VolumeHost
	csiDriverLister           storagelisters.CSIDriverLister
	serviceAccountTokenGetter func(namespace, name string, tr *authenticationv1.TokenRequest) (*authenticationv1.TokenRequest, error)
	volumeAttachmentLister    storagelisters.VolumeAttachmentLister
}

// ProbeVolumePlugins returns implemented plugins
func ProbeVolumePlugins() []volume.VolumePlugin {
	p := &csiPlugin{
		host: nil,
	}
	return []volume.VolumePlugin{p}
}
```

#### 3.1.1 Attacher

csiPlugin 实现了 `CanAttach()` 接口，用于让 Kubernetes 知晓是否需要执行 Attach 操作。
```go
// CanAttach 判断 Volume 是否需要 Attach
func (p *csiPlugin) CanAttach(spec *volume.Spec) (bool, error) {
	// inline Volume 不支持
    inlineEnabled := utilfeature.DefaultFeatureGate.Enabled(features.CSIInlineVolume)
	if inlineEnabled {
		volumeLifecycleMode, err := p.getVolumeLifecycleMode(spec)
		if err != nil {
			return false, err
		}

		if volumeLifecycleMode == storage.VolumeLifecycleEphemeral {
			klog.V(5).Info(log("plugin.CanAttach = false, ephemeral mode detected for spec %v", spec.Name()))
			return false, nil
		}
	}

	// 取得 PV 定义与 CSI Driver name
	pvSrc, err := getCSISourceFromSpec(spec)
	if err != nil {
		return false, err
	}

	driverName := pvSrc.Driver

	// 检查该 CSI Driver 是否需要 Attach
	skipAttach, err := p.skipAttach(driverName)
	if err != nil {
		return false, err
	}

	return !skipAttach, nil
}

func (p *csiPlugin) skipAttach(driver string) (bool, error) {
	// ...

	// 得到对应的 CSIDriver 资源
	csiDriver, err := p.csiDriverLister.Get(driver)
	if err != nil {
		if apierrors.IsNotFound(err) {
			// CSIDriver 对象不存在，默认为需要 Attach
			return false, nil
		}
		return false, err
	}
	// 通过 spec.attachRequired 判断是否需要 Attach
	if csiDriver.Spec.AttachRequired != nil && *csiDriver.Spec.AttachRequired == false {
		return true, nil
	}
	return false, nil
}
```

### 3.2 Kubelet 层的实现

#### 3.2.1 Plugin Watcher - 注册插件 

与 Controller 层不同，Kubelet 需要调用 CSI Node 的接口，因此需要一个注册机制，让 CSI 插件注册到 Kubelet 中。

Plugin Manager 会创建并运行一个 Plugin Watcher，Plugin Watcher 会监控一个 socket 根目录及其子目录。一旦目录或子目录中创建了 socket 文件，就认定其是一个 “插件注册” 事件，将其 socket 文件路径加入到 desiredStateOfWorld，等待后续的协调处理。

{{< admonition note "根目录路径">}}
插件注册的根目录路径为 `<kubelet-root>/plugins_registry`，默认为 `/var/lib/kubelet/plugins_registry`。
{{< /admonition >}}

```go
// Watcher 监控 Plugin 的注册
type Watcher struct {
	path                string            // 根目录
	fs                  utilfs.Filesystem
	fsWatcher           *fsnotify.Watcher
	desiredStateOfWorld cache.DesiredStateOfWorld
}
```

Watcher.Start() 经过初始的扫描与处理后，会启动一个 Goroutine 监控文件系统事件，并处理。
```go
func (w *Watcher) Start(stopCh <-chan struct{}) error {
	// 创建根目录
	if err := w.init(); err != nil {
		return err
	}

	// 创建 fs watcher
	fsWatcher, err := fsnotify.NewWatcher()
	if err != nil {
		return fmt.Errorf("failed to start plugin fsWatcher, err: %v", err)
	}
	w.fsWatcher = fsWatcher

	// 扫描当前根目录
	if err := w.traversePluginDir(w.path); err != nil {
		klog.ErrorS(err, "Failed to traverse plugin socket path", "path", w.path)
	}

	// 启动监控文件系统的 Groutine
	go func(fsWatcher *fsnotify.Watcher) {
		for {
			select {
			case event := <-fsWatcher.Events:
				if event.Op&fsnotify.Create == fsnotify.Create {
					// 处理文件创建事件
					err := w.handleCreateEvent(event)
				} else if event.Op&fsnotify.Remove == fsnotify.Remove {
					// 处理文件删除事件
					w.handleDeleteEvent(event)
				}
				continue

			case err := <-fsWatcher.Errors:
				if err != nil {
					klog.ErrorS(err, "FsWatcher received error")
				}
				continue

			case <-stopCh:
				w.fsWatcher.Close()
				return
			}
		}
	}(fsWatcher)

	return nil
}

func (w *Watcher) traversePluginDir(dir string) error {
	// 添加根目录的监控
	err := w.fsWatcher.Add(dir)

// 遍历根目录处理
	return w.fs.Walk(dir, func(path string, info os.FileInfo, err error) error {
    // ...

		// do not call fsWatcher.Add twice on the root dir to avoid potential problems.
		if path == dir {
			return nil
		}

		switch mode := info.Mode(); {
		case mode.IsDir():
			// 根目录下的子目录页加入监控
			err := w.fsWatcher.Add(path)
		case mode&os.ModeSocket != 0:
			// socket 文件进行处理
			event := fsnotify.Event{
				Name: path,
				Op:   fsnotify.Create,
			}
      w.handleCreateEvent(event)
		}

		return nil
	})
}
```

`handleCreateEvent()` 与 `handleDeleteEvent()` 就是将 socket 文件路径 添加/移除 到 desiredStateOfWorld。
```go
func (w *Watcher) handleCreateEvent(event fsnotify.Event) error {
	// stat file
	fi, err := os.Stat(event.Name)
	
  // ...

	if !fi.IsDir() {
		isSocket, err := util.IsUnixDomainSocket(util.NormalizePath(event.Name))

		// 忽略其他类型文件
		if !isSocket {
			return nil
		}

    // 如果是 socket 文件，注册到 desiredStateOfWorld
		return w.handlePluginRegistration(event.Name)
	}

	// 如果是子目录，继续监控与处理
	return w.traversePluginDir(event.Name)
}

func (w *Watcher) handlePluginRegistration(socketPath string) error {
	err := w.desiredStateOfWorld.AddOrUpdatePlugin(socketPath)
	if err != nil {
		return fmt.Errorf("error adding socket path %s or updating timestamp to desired state cache: %v", socketPath, err)
	}
	return nil
}

func (w *Watcher) handleDeleteEvent(event fsnotify.Event) {
	socketPath := event.Name
	w.desiredStateOfWorld.RemovePlugin(socketPath)
}
```

#### 3.2.2 Plugin Manager - 插件管理

Kubelet 中实现了 Plugin Manager 进行插件的记录与管理，其还是依靠 actualStateOfWorld 与 desiredStateOfWorld 协调，根据 Add/Remove 事件调用注册的 RegisterPlugin() 与 DeRegisterPlugin() 回调。

其核心逻辑由 Reconciler 实现。
```go
// Reconciler runs a periodic loop to reconcile the desired state of the world
// with the actual state of the world by triggering register and unregister
// operations. Also provides a means to add a handler for a plugin type.
type Reconciler interface {
  // 运行 Reconciler
	Run(stopCh <-chan struct{})

  // 注册 Plugin 事件的回调
	AddHandler(pluginType string, pluginHandler cache.PluginHandler)
}

type reconciler struct {
	operationExecutor   operationexecutor.OperationExecutor
	loopSleepDuration   time.Duration
	desiredStateOfWorld cache.DesiredStateOfWorld
	actualStateOfWorld  cache.ActualStateOfWorld
	handlers            map[string]cache.PluginHandler  // Plugin 事件回调
	sync.RWMutex
}
```

周期性的协调流程依旧是对比 actualStateOfWorld 与 desiredStateOfWorld 进行处理，并调用对应注册的 PluginHandler。
```go
func (rc *reconciler) reconcile() {
  // 遍历 actualStateOfWorld
	for _, registeredPlugin := range rc.actualStateOfWorld.GetRegisteredPlugins() {

    // 判断插件还是否存在
		unregisterPlugin := false
		if !rc.desiredStateOfWorld.PluginExists(registeredPlugin.SocketPath) {
			unregisterPlugin = true
		} else {
			for _, dswPlugin := range rc.desiredStateOfWorld.GetPluginsToRegister() {
				if dswPlugin.SocketPath == registeredPlugin.SocketPath && dswPlugin.Timestamp != registeredPlugin.Timestamp {
					unregisterPlugin = true
					break
				}
			}
		}

    // 取消插件注册，底层调用注册的 rc.handlers[xx].DeRegisterPlugin()
		if unregisterPlugin {
			err := rc.operationExecutor.UnregisterPlugin(registeredPlugin, rc.actualStateOfWorld)
		}
	}

  // 遍历 desiredStateOfWorld
	for _, pluginToRegister := range rc.desiredStateOfWorld.GetPluginsToRegister() {
		if !rc.actualStateOfWorld.PluginExistsWithCorrectTimestamp(pluginToRegister) {
      // 注册插件，底层调用注册的 rc.handlers[xx].RegisterPlugin()
			err := rc.operationExecutor.RegisterPlugin(pluginToRegister.SocketPath, pluginToRegister.Timestamp, rc.getHandlers(), rc.actualStateOfWorld)
		}
	}
}
```

#### 3.2.3 Kubelet 与 Plugin 的交互

在最底层，我们可以看到核心的 Kubelet 与 Plugin 插件交互进行注册的逻辑。
```go
func (og *operationGenerator) GenerateRegisterPluginFunc(
	socketPath string,
	timestamp time.Time,
	pluginHandlers map[string]cache.PluginHandler,
	actualStateOfWorldUpdater ActualStateOfWorldUpdater) func() error {

	// registerPluginFunc 注册一个 CSI Plugin
	registerPluginFunc := func() error {
		// gRPC 连接 socket 文件
		client, conn, err := dial(socketPath, dialTimeoutDuration)
		if err != nil {
			return fmt.Errorf("RegisterPlugin error -- dial failed at socket %s, err: %v", socketPath, err)
		}
		defer conn.Close()

		ctx, cancel := context.WithTimeout(context.Background(), time.Second)
		defer cancel()

		// 调用 GetInfo 接口的插件的信息，包括
		//  + Type: CSI 或者 Device
		//	+ Name: 插件名字
		//  + Endpoint: 插件接口访问地址
		infoResp, err := client.GetInfo(ctx, &registerapi.InfoRequest{})

		// 插件对应类型的 Handler
		handler, ok := pluginHandlers[infoResp.Type]

		if infoResp.Endpoint == "" {
			infoResp.Endpoint = socketPath
		}

		// 调用 Handler 回调注册，也保存到 plugin manager
		err := handler.ValidatePlugin(infoResp.Name, infoResp.Endpoint, infoResp.SupportedVersions)

		err = actualStateOfWorldUpdater.AddPlugin(cache.PluginInfo{
			SocketPath: socketPath,
			Timestamp:  timestamp,
			Handler:    handler,
			Name:       infoResp.Name,
		})
		
		if err := handler.RegisterPlugin(infoResp.Name, infoResp.Endpoint, infoResp.SupportedVersions); err != nil {
			return og.notifyPlugin(client, false, fmt.Sprintf("RegisterPlugin error -- plugin registration failed with err: %v", err))
		}

		// 回调通知插件注册状态
		err := og.notifyPlugin(client, true, "")

		return nil
	}
	return registerPluginFunc
}
```

{{< admonition note Note>}}
可以看到，上面 Kubelet 使用一套基于 gRPC 的接口进行插件的注册，而实现这些接口就是 [**node-driver-registrar**](https://github.com/kubernetes-csi/node-driver-registrar) 容器的责任了。

也就是说，node-driver-registrar 为 CSI Plugin 实现了注册插件到 Kubelet 的逻辑，而 CSI Plugin 仅仅只需要关注实现 Volume 处理相关的逻辑即可。
{{< /admonition >}}

### 3.3 CSI Plugin 实现

从 [**源码阅读 - Volume 实现**](../volume-implementation/#5-kubelet-的处理) 中我们知晓了，Kubelet 底层还是使用某个 VolumePlugin 的实现的接口进行 Volume 操作。而在 CSI 情况下，其调用的就是 CSI VolumePlugin。

CSI VolumePlugin 本质上就是一个 Plugin Manager，其会根据 Volume 类型调用指定的 CSI Plugin 的接口。我们以 CSI Mounter 为例，可以看到其最后调用了 CSI Node 的 NodePublishVolume 接口。

```go
type csiPlugin struct {
	host                      volume.VolumeHost
	csiDriverLister           storagelisters.CSIDriverLister
	serviceAccountTokenGetter func(namespace, name string, tr *authenticationv1.TokenRequest) (*authenticationv1.TokenRequest, error)
	volumeAttachmentLister    storagelisters.VolumeAttachmentLister
}

func (c *csiMountMgr) SetUpAt(dir string, mounterArgs volume.MounterArgs) error {
	// 得到 CSI Node
	csi, err := c.csiClientGetter.Get()
	
	// 解析 Volume 配置

	// 创建 Pod 的目录
	parentDir := filepath.Dir(dir)
	if err := os.MkdirAll(parentDir, 0750); err != nil {
		return errors.New(log("mounter.SetUpAt failed to create dir %#v:  %v", parentDir, err))
	}

	// 读取 Secret
	nodePublishSecrets = map[string]string{}
	if secretRef != nil {
		nodePublishSecrets, err = getCredentialsFromSecret(c.k8s, secretRef)
	}

	// CSI Driver 的 `podInfoOnMount` 功能实现
	podInfoEnabled, err := c.plugin.podInfoEnabled(string(c.driverName))
	if podInfoEnabled {
		volAttribs = mergeMap(volAttribs, getPodInfoAttrs(c.pod, c.volumeLifecycleMode))
	}

	// CSI Driver 的 `tokenRequest` 功能实现
	serviceAccountTokenAttrs, err := c.podServiceAccountTokenAttrs()
	volAttribs = mergeMap(volAttribs, serviceAccountTokenAttrs)

	// CSI Driver 的 `fsGroupPolicy` 功能实现
	driverSupportsCSIVolumeMountGroup := false
	var nodePublishFSGroupArg *int64
	if utilfeature.DefaultFeatureGate.Enabled(features.DelegateFSGroupToCSIDriver) {
		driverSupportsCSIVolumeMountGroup, err = csi.NodeSupportsVolumeMountGroup(ctx)

		if driverSupportsCSIVolumeMountGroup {
			nodePublishFSGroupArg = mounterArgs.FsGroup
		}
	}

	// 调用 CSI Node 接口
	err = csi.NodePublishVolume(
		ctx,
		volumeHandle,
		readOnly,
		deviceMountPath,
		dir,
		accessMode,
		publishContext,
		volAttribs,
		nodePublishSecrets,
		fsType,
		mountOptions,
		nodePublishFSGroupArg,
	)

	// ...

	return nil
}

```

那最后的问题就是，CSI Plugin 是如何注册到 CSI VolumePlugin 中的？CSI VolumePlugin 实现了前面所说的 PluginHandler 接口，并在 Kubelet 启动时注册到了 Plugin Manager 中。

因此当新的 CSI Plugin 创建 socket 文件后，Kubelet 的 Plugin Manager 监听到创建事件，然后回调 CSI VolumePlugin 的 RegisterPlugin() 的接口，记录下来。

```go
// RegisterPlugin is called when a plugin can be registered
func (h *RegistrationHandler) RegisterPlugin(pluginName string, endpoint string, versions []string) error {
	// 记录下 CSI Plugin
	csiDrivers.Set(pluginName, Driver{
		endpoint:                endpoint,
		highestSupportedVersion: highestSupportedVersion,
	})

	// ...

	return nil
}
```

### 3.4 总结

{{< find_img "img3.png" >}}

## 4 实现一个 CSI 插件

## 参考

* [**《Kubernetes CSI Developer Document》**](https://kubernetes-csi.github.io/docs/)
* [**CSI Spec**](https://github.com/container-storage-interface/spec/blob/master/spec.md)

