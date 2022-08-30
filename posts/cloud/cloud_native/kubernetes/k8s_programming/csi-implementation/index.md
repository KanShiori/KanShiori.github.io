# K8s 实现 - CSI 实现


> 介绍 Kubernetes 源码中 CSI 相关的实现。
> 
> Volume 的基本概念见 [**Kubernetes - 存储设计**](../../k8s_learning/volume)。

## 1 Interface

在 [**Kubernetes - 存储设计**](../../k8s_learning/volume#3-csi-基本架构) 中看到，具体实现 CSI 时需要用户实现三个 RPC Service：

{{< image src="img0.png" height=300 >}}

* CSI Identity - 提供查询 CSI 插件信息的接口；
* CSI Controller - 提供对 Volume 的管理接口；
* CSI Node - 提供 Node 上操作接口；

下面是各个 Service Interface，看一下具体要支持哪些 RPC 接口。

{{< admonition note Note>}}
具体的协议定义见 [**csi.proto**](https://github.com/container-storage-interface/spec/blob/master/csi.proto)
{{< /admonition >}}

### 1.1 CSI Identity

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

### 1.2 CSI Controller

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

### 1.3 CSI Node

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

## 2 CSI Volume Plugin

在 Kubernetes 实现中，都是围绕着 Volume Plugin 进行 Volume 的实际操作。Volume Plugin 中需要实现 Provision Attach 等操作。而 CSI Plugin 是更加底层的接口，提供相关的 RPC 接口。

可以看到，Volume Plugin 与 CSI Plugin 接口定义并不相同。因此问题在于，如何让 Volume Plugin 转换为 CSI Plugin 的调用。

为此，Kubernetes 源码中通过 `CSI Volume Plugin` 实现了 Adpator 的作用：

{{< image src="img1.png" height=130 >}}

* 对于 Controller 与 Kubelet，CSI Volume Plugin 提供了 [**Volume Plugin**](../volume-implementation/#1-base-interface) 的实现。
* 上层调用 CSI Volume Plugin 后，通过特定方式 “调用” Sidecar 与 CSI Driver。

### 2.1 CSI Volume Plugin 实现

源码中，CSI Volume Plugin 对应的就是 `csiPlugin` 结构体，其实现了 [**VolumePlugin Interface**](../volume-implementation/#1-base-interface)：

```go
// csiPlugin implement VolumePlugin
type csiPlugin struct {
	host                      volume.VolumeHost
	csiDriverLister           storagelisters.CSIDriverLister
	serviceAccountTokenGetter func(namespace, name string, tr *authenticationv1.TokenRequest) (*authenticationv1.TokenRequest, error)
	volumeAttachmentLister    storagelisters.VolumeAttachmentLister
}

func (p *csiPlugin) NewMounter(spec *volume.Spec, pod *api.Pod, _ volume.VolumeOptions) (volume.Mounter, error) 

func (p *csiPlugin) NewUnmounter(specName string, podUID types.UID) (volume.Unmounter, error)

func (p *csiPlugin) NewAttacher() (volume.Attacher, error)

func (p *csiPlugin) NewDeviceMounter() (volume.DeviceMounter, error) 

func (p *csiPlugin) NewDetacher() (volume.Detacher, error) 

// ...
```

* host - Kubelet 调用时使用，提供 CSI Plugin 的 Node Service 以及获取 Node 信息的接口。

### 2.2 Provision / Delete

CSI Plugin 也是基于 PVC 与 PV 来判断是否需要 Provision 与 Delete Volume 的。因此 csiPlugin 并没有实现 `ProvisionableVolumePlugin` 与 `DeletableVolumePlugin` Interface。

Volume 的创建与删除都是由 CSI Plugin 自行 Watch PVC 来处理的。当然，可以直接复用 Sidecar 容器 [**external-provisioner**](https://github.com/kubernetes-csi/external-provisioner) 来实现。

### 2.3 Attach / Detach

Attach 与 Detach 并没有通用的资源定义实现，In-Tree 插件都是直接调用代码完成的。

为此，csiPlugin 需要进行适配。所以 csiPlugin 实现了 `NewAttacher` 与 `NewDetacher` 方法：

```go
func (p *csiPlugin) NewAttacher() (volume.Attacher, error)

func (p *csiPlugin) NewDetacher() (volume.Detacher, error) 
```

Attacher 与 Detacher 都由 `csiAttacher` 实现，具体 Attach / Detach 函数就是创建或删除 VolumeAttachment 资源。

```go
type csiAttacher struct {
	plugin       *csiPlugin
	k8s          kubernetes.Interface
	watchTimeout time.Duration

	csiClient csiClient
}

func (c *csiAttacher) Attach(spec *volume.Spec, nodeName types.NodeName) (string, error) {
	// ...

	attachment, err := c.plugin.volumeAttachmentLister.Get(attachID)
	if err != nil && !apierrors.IsNotFound(err) {
		return "", errors.New(log("failed to get volume attachment from lister: %v", err))
	}

	if attachment == nil {
		// build attachment ...

		// create attachment
		_, err = c.k8s.StorageV1().VolumeAttachments().Create(context.TODO(), attachment, metav1.CreateOptions{})
	}

	// ...
	return "", nil
}

func (c *csiAttacher) Detach(volumeName string, nodeName types.NodeName) error {
	// get attachID ...

	// delete attachment
	if err := c.k8s.StorageV1().VolumeAttachments().Delete(context.TODO(), attachID, metav1.DeleteOptions{}); err != nil {
		//
	}

	// ...
}
```

### 2.4 Mount / Unmount

Mount 与 Unmount 接口在实现中由 `DeviceMounter` 与 `DeviceUnmounter` 提供。

csiPlugin 实现了 `DeviceMountableVolumePlugin`，提供获取接口：

```go
func (p *csiPlugin) NewDeviceMounter() (volume.DeviceMounter, error)

func (p *csiPlugin) NewDeviceUnmounter() (volume.DeviceUnmounter, error)
```

{{< admonition note Note>}}
在源码中，Mount / Unmount 对应的接口名称为 `MountDevice()` 与 `UnmountDevice()`，表示就是将 Device 挂载到 Global Dir。
{{< /admonition >}}

DeviceMounter 与 DeviceUnmounter 都由 `csiAttacher` 实现，具体 Mount / Unmount 函数就是创建 Global Dir，然后调用 Node Service 进行挂载。

```go
func (c *csiAttacher) MountDevice(spec *volume.Spec, devicePath string, deviceMountPath string, deviceMounterArgs volume.DeviceMounterArgs) error {
	// ...

	// create global dir
	if err = os.MkdirAll(deviceMountPath, 0750); err != nil {
		return errors.New(log("attacher.MountDevice failed to create dir %#v:  %v", deviceMountPath, err))
	}

	// ...

	// call Node Service
	fsType := csiSource.FSType
	err = csi.NodeStageVolume(ctx,
		csiSource.VolumeHandle,
		publishContext,
		deviceMountPath,
		fsType,
		accessMode,
		nodeStageSecrets,
		csiSource.VolumeAttributes,
		mountOptions,
		nodeStageFSGroupArg)

	return err
}

func (c *csiAttacher) UnmountDevice(deviceMountPath string) error {
	klog.V(4).Info(log("attacher.UnmountDevice(%s)", deviceMountPath))
	// ...

	// call Node Service
	err = csi.NodeUnstageVolume(ctx,
		volID,
		deviceMountPath)

	// delete global dir
	// Delete the global directory + json file
	removeMountDir(c.plugin, deviceMountPath)
	return nil
}
```

可有看到，Node 上由 Kubelet 调用是 Volume Plugin 时，直接是调用 CSI Node Service 的 RPC 接口的，也就是同步的调用。

当然，这里另一个问题是 Kubelet 如何对应的 CSI Node Service，这个后面再讨论。

### 2.5 SetUp / TearDown

csiPlugin 当然也实现最基本的 `VolumePlugin`：

```go
func (p *csiPlugin) NewMounter(spec *volume.Spec, pod *api.Pod, _ volume.VolumeOptions) (volume.Mounter, error)

func (p *csiPlugin) NewUnmounter(specName string, podUID types.UID) (volume.Unmounter, error)
```

Mounter 与 Umounter 都由 `csiMountMgr` 实现，具体的 SetUp / TearDown 会调用到 CSI Node Service 接口：

```go
type csiMountMgr struct {
	csiClientGetter
	k8s                 kubernetes.Interface
	plugin              *csiPlugin
	driverName          csiDriverName
	volumeLifecycleMode storage.VolumeLifecycleMode
	fsGroupPolicy       storage.FSGroupPolicy
	volumeID            string
	specVolumeID        string
	readOnly            bool
	supportsSELinux     bool
	spec                *volume.Spec
	pod                 *api.Pod
	podUID              types.UID
	publishContext      map[string]string
	kubeVolHost         volume.KubeletVolumeHost
	volume.MetricsProvider
}

func (c *csiMountMgr) SetUpAt(dir string, mounterArgs volume.MounterArgs) error {
	// ...

	// 创建 Pod 的目录
	// create target_dir before call to NodePublish
	parentDir := filepath.Dir(dir)
	if err := os.MkdirAll(parentDir, 0750); err != nil {
		return errors.New(log("mounter.SetUpAt failed to create dir %#v:  %v", parentDir, err))
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


func (c *csiMountMgr) TearDownAt(dir string) error {
	if err := csi.NodeUnpublishVolume(ctx, volID, dir); err != nil {
		return errors.New(log("mounter.TearDownAt failed: %v", err))
	}

	// Deprecation: Removal of target_path provided in the NodePublish RPC call
	// (in this case location `dir`) MUST be done by the CSI plugin according
	// to the spec. This will no longer be done directly as part of TearDown
	// by the kubelet in the future. Kubelet will only be responsible for
	// removal of json data files it creates and parent directories.
	if err := removeMountDir(c.plugin, dir); err != nil {
		return errors.New(log("mounter.TearDownAt failed to clean mount dir [%s]: %v", dir, err))
	}

	return nil
}
```

可有看到，Node 上由 Kubelet 调用是 Volume Plugin 时，直接是调用 CSI Node Service 的 RPC 接口的，也就是同步的调用。

当然，这里另一个问题是 Kubelet 如何对应的 CSI Node Service，这个后面再讨论。

## 3 Controller 层实现

在 [**CSI Volume Plugin**](#2-csi-volume-plugin) 中看到，Controller 层面的操作都是围绕着 Resource 异步实现的：

* Provision / Delete - 基于 PVC 资源
* Attach / Detach - 基于 VolumeAttachment 资源
* Snapshot - 基于 VolumeSnapshot 等资源

也就是说，Kubernetes 仅仅负责创建这些 Resource，具体的操作由对应的 CSI Controller 监听到 Resource Event 后去执行。

{{< image src="img2.png" height=200 >}}

## 4 Node 层实现

在 [**CSI Volume Plugin**](#2-csi-volume-plugin) 中看到，Node 层面的 Volume 操作都是直接调用 CSI Node Service 的 RPC 接口的。

因此，Kubelet 除了调用 CSI Volume Plugin 操作 Volume 外，还需要负责 CSI Node Service 的发现与管理。

Kubelet 中直接调用与管理 CSI Node Service 的实现如下：

{{< image src="img3.png" height=330 >}}

* 发现机制 - 基于 socket file 的创建与删除，每一个 socket file 对应与一个 CSI Node Service。

* 管理机制 - PluginManager 类负责 CSI Plugin 的注册与删除，并提供 CSI Node Service 的查找接口

### 4.1 Plugin Manager - 插件管理

与 Volume 实现中的写法类似，Plugin Manager 也是围绕着 `desiredStateOfWorld` 与 `actualStateOfWorld` 进行处理的。

Plugin Manager 核心逻辑由 Reconciler 实现：

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

当判断出 Plugin 的 Add / Remove 事件时，会调用 `RegisterPlugin()` / `DeRegisterPlugin()` 回调进行注册。

在 `RegisterPlugin()` 的调用链的底层，可以看到是基于 Node Service 的 `GetInfo()` 接口获取插件信息，完成注册的。

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

{{< admonition tip Tip>}}
注册 RPC Service 的操作可以由 [**node-driver-registrar**](https://github.com/kubernetes-csi/node-driver-registrar) 容器完成。例如，创建 socket 文件，提供 RPC Service 等。
{{< /admonition >}}

{{< admonition note "为什么要通过回调完成注册">}}
还是为了将控制逻辑与操作逻辑分开。例如对于 CSI Plugin 是基于 `GetInfo()` RPC 获取信息并完成注册的，可能对于其他 Plugin 会通过其他方式获取信息进行注册。
{{< /admonition >}}

### 4.2 发现插件 

为了 Kubelet 知晓存在哪些 CSI Plugiun，需要一个注册机制让 CSI Plugin 注册到 Kubelet 中。

Kubelet 会监控一个特定的目录的 socket file 的 Add / Remove 事件：

* Add - 表明 Plugin 发起注册，记录到 `desiredStateOfWorld`；
* Remove - 表明 Plugin 被移除，移除 `desiredStateOfWorld` 对应记录；

{{< admonition note "插件目录路径">}}
插件注册的根目录路径为 <important>\<kubelet-root>/plugins_registry</important>，默认为 **/var/lib/kubelet/plugins_registry**。
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

Watcher 的主要逻辑就是监控着插件目录的文件，根据 Add / Remove 事件更新 `desiredStateOfWorld`：

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
```

## 5 CSI Plugin 的实现

TODO

## 参考

* [**《Kubernetes CSI Developer Document》**](https://kubernetes-csi.github.io/docs/)
* [**CSI Spec**](https://github.com/container-storage-interface/spec/blob/master/spec.md)

