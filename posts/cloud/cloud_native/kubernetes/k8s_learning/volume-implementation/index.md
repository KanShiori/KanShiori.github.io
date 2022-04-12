# Kubernetes Volume 的实现


## 1 概述

以一个云存储作为 Volume 为例，其架构图如下：
{{< find_img "img2.png" >}}

1. **`Deploy PVC`** - 用户 Deploy PVC 对象。
2. **`Provision Volume`** - PV Controller 基于 StorageClass 创建一个 Volume（无法绑定已有 PV 的情况下），并创建 PV 对象。
3. **`Bind PV`** - PV Controller 将 PV 与 PVC 绑定。
4. **`Attach Volume`** - AttachDetach Controller Attach Volume 到 Node 上，Node 上看来是一个块设备。
5. **`Mount Device`** - Kubelet 将块设备 Mount 到一个全局的目录。
6. **`Setup Volume`** - Kubelet 准备 Pod 特定的目录，该目录后续用于挂载到各个容器中的目录。

上述过程是一个完成的过程，不同的 Volume 类型可能只会执行其中的几步，例如一些 “特殊” 的 Volume（例如 hostpath、secret、configmap 等），仅仅实现了 SetUp 步骤。
{{< find_img "img2-2.png" >}}

对应的，销毁流程就是创建流程的相反操作：
1. **`TearDown Volume`** - Kubelet 移除为 Pod 准备的目录。
2. **`Unmount Device`** - Unmount 全局目录。
3. **`Detach Volume`** - 从 Node 上 Detach Volume。
4. **`Delete`** - 特定条件下，删除该 Volume。

{{< admonition note Note>}}
上述步骤很重要，是后续所有组件执行的操作。
{{< /admonition >}}

## 2 Volume Plugin

无论是控制面的 Controller 还是 Kubelet，涉及到实际的 Volume 操作（Create/Delete、Attach/Detach 等）时，都会调用对应的 **`VolumePlugin`** 接口。VolumePlugin 为管理组件提供了 [**概述**](#1-概述) 中的具体操作接口。

目前 Kubernetes 内置了许多的 Volume Plugin，可以分为三类：

* **内置 Volume** - 包括 ConfigMap、HostPath、以及各个云厂商的 Volume，其代码是原生内置在 Controller Manager 中的；
* **CSI** - Container Storage Interface，在 Controller Mananger 中是一个独立的 Volume Plugin，由该 Plugin 来调用注册的 CSI Plugin；
* **FlexVolume** - 通过可执行文件实现的动态注册插件

### 2.1 基础接口

先了解下最基本执行各个操作的接口，这些接口都是用于特定 Pod 执行特定的 Volume 操作（Pod 与 Volume 都是确定的了），由 VolumePlugin 的方法得到。

可以看到，各个接口对应了在 [**概述**](#1-概述) 所对应的步骤。

* **Mounter**
  * `CanMount()` - SetUp() 调用前检查所需的组件（例如插件程序）是否准备好
  * `SetUp()` 与 `SetUpAt()` - 为 Pod 准备好对应的 Volume 目录
* **Unmounter**
  * `TearDown()` 与 `TearDownAt()` - 移除为 Pod 准备的 Volume 目录
* **CustomBlockVolumeMapper**
  * `SetUpDevice()` - 为 Pod 准备好 Device
  * `MapPodDevice()` - 将 Device 映射给 Pod
* **CustomBlockVolumeUnmapper**
  * `TearDownDevice()` - 移除为 Pod 准备好的 Device
  * `UnmapPodDevice()` - 取消 Device 映射
* **Provisioner**
  * `Provision()` - 创建一个 Volume
* **Deleter**
  * `Delete()` - 删除一个 Volume
* **Attacher**
  * `Attach()` - Attach Volume 到 Node 上
  * `VolumesAreAttached()` - 返回 Node 上的 attached Volume
* **Detacher**
  * `Detach()` - Detach Volume
* **DeviceMounter**
  * `MountDevice()` - 将 attached Volume 挂载到一个全局目录
* **DeviceUnmounter**
  * `UnmountDevice()` - 取消 attached Volume 的全局目录挂载

```go
// Volume represents a directory used by pods or hosts on a node. All method
// implementations of methods in the volume interface must be idempotent.
type Volume interface {
	// GetPath 返回 volume 需要的挂载路径
	GetPath() string
	// MetricsProvider 提供统计接口
	MetricsProvider
}

// BlockVolume interface provides methods to generate global map path
// and pod device map path.
type BlockVolume interface {
	// GetGlobalMapPath returns a global map path which contains
	// bind mount associated to a block device.
	// ex. plugins/kubernetes.io/{PluginName}/{DefaultKubeletVolumeDevicesDirName}/{volumePluginDependentPath}/{pod uuid}
	GetGlobalMapPath(spec *Spec) (string, error)
	// GetPodDeviceMapPath returns a pod device map path
	// and name of a symbolic link associated to a block device.
	// ex. pods/{podUid}/{DefaultKubeletVolumeDevicesDirName}/{escapeQualifiedPluginName}/, {volumeName}
	GetPodDeviceMapPath() (string, string)
	SupportsMetrics() bool
	MetricsProvider
}

// Mounter 接口为 Pod 提供 Volume 对应的目录
type Mounter interface {
	Volume

	// CanMount 优先于 Setup 调用，用于检查当前环境是否包含能够 Mount 的组件
	CanMount() error
	// SetUp 为 Pod 提供 Volume 的目录，目录路径由自己决定
	SetUp(mounterArgs MounterArgs) error
	// SetUpAt  为 Pod 提供 Volume 的目录，目录路径由 dir 指定
	SetUpAt(dir string, mounterArgs MounterArgs) error
	// GetAttributes 返回 Volume 的属性
	GetAttributes() Attributes
}

// Unmounter interface provides methods to cleanup/unmount the volumes.
type Unmounter interface {
	Volume

	// TearDown 取消为 Pod 提供的目录
	TearDown() error
	// TearDown 取消为 Pod 提供的指定目录
	TearDownAt(dir string) error
}

// BlockVolumeMapper interface is a mapper interface for block volume.
type BlockVolumeMapper interface {
	BlockVolume
}

// CustomBlockVolumeMapper interface provides custom methods to set up/map the volume.
type CustomBlockVolumeMapper interface {
	BlockVolumeMapper

	// SetUpDevice 准备为 Pod 提供的块设备
	SetUpDevice() (stagingPath string, err error)
	// MapPodDevice maps the block device to a path and return the path.
	MapPodDevice() (publishPath string, err error)
	// GetStagingPath returns path that was used for staging the volume
	// it is mainly used by CSI plugins
	GetStagingPath() string
}

// BlockVolumeUnmapper interface is an unmapper interface for block volume.
type BlockVolumeUnmapper interface {
	BlockVolume
}

// CustomBlockVolumeUnmapper interface provides custom methods to cleanup/unmap the volumes.
type CustomBlockVolumeUnmapper interface {
	BlockVolumeUnmapper
	// TearDownDevice 移除为 Pod 提供的块设备
	TearDownDevice(mapPath string, devicePath string) error
	// UnmapPodDevice removes traces of the MapPodDevice procedure.
	UnmapPodDevice() error
}

type Provisioner interface {
	// Provision 创建一个 Volume
	Provision(selectedNode *v1.Node, allowedTopologies []v1.TopologySelectorTerm) (*v1.PersistentVolume, error)
}

type Deleter interface {
	Volume
	// Deleter 移除一个 Volume
	Delete() error
}

type Attacher interface {
	DeviceMounter

	// Attach Volume 到指定的 Node 上
	Attach(spec *Spec, nodeName types.NodeName) (string, error)
	// VolumesAreAttached 查询 Node 上的 attached Volume
	VolumesAreAttached(specs []*Spec, nodeName types.NodeName) (map[*Spec]bool, error)
	// WaitForAttach 等待 attach 结束
	WaitForAttach(spec *Spec, devicePath string, pod *v1.Pod, timeout time.Duration) (string, error)
}

type DeviceMounter interface {
	// GetDeviceMountPath 返回 Node 上 Volume 对应的块设备路径
	GetDeviceMountPath(spec *Spec) (string, error)
	// MountDevice 将 Volume 对应的块设备挂载到一个全局的目录（不提供给 Pod）
	MountDevice(spec *Spec, devicePath string, deviceMountPath string, deviceMounterArgs DeviceMounterArgs) error
}

type BulkVolumeVerifier interface {
	// BulkVerifyVolumes checks whether the list of volumes still attached to the
	// the clusters in the node. It returns a map which maps from the volume spec to the checking result.
	// If an error occurs during check - error should be returned and volume on nodes
	// should be assumed as still attached.
	BulkVerifyVolumes(volumesByNode map[types.NodeName][]*Spec) (map[types.NodeName]map[*Spec]bool, error)
}

type Detacher interface {
	DeviceUnmounter
	// Detach Volume 在指定的 Node
	Detach(volumeName string, nodeName types.NodeName) error
}

type DeviceUnmounter interface {
	// UnmountDevice 取消 Volume 对应块设备的挂载
	UnmountDevice(deviceMountPath string) error
}
```

### 2.2 VolumePlugin 接口

**`VolumePlugin`** 是 Volume Plugin 的最基本的 interface。
```go
type Spec struct {
	Volume                          *v1.Volume
	PersistentVolume                *v1.PersistentVolume
	ReadOnly                        bool
	InlineVolumeSpecForCSIMigration bool
	Migrated                        bool
}

// VolumePlugin is an interface to volume plugins that can be used on a
// kubernetes node (e.g. by kubelet) to instantiate and manage volumes.
type VolumePlugin interface {
	// Init 初始化 plugin
	Init(host VolumeHost) error

	// Name 返回 plugin name
	// 命名方式为 "example.com/volume"，"kubernetes.io" 由 kubernete 内置 Volume 预留使用
	GetPluginName() string

	// GetVolumeName 基于 spec 返回一个唯一的 volume name 或者 id
	// 对于 Attach 与 Detach 操作会传递该 name 来让 plugin 识别 volume
	GetVolumeName(spec *Spec) (string, error)

	// CanSupport 返回 plugin 是否支持该 spec
	CanSupport(spec *Spec) bool

	// RequiresRemount 表明 plugin 是否支持 volume 的重新挂载
	RequiresRemount(spec *Spec) bool

	// NewMounter 创建一个 Mounter
	NewMounter(spec *Spec, podRef *v1.Pod, opts VolumeOptions) (Mounter, error)

	// NewUnmounter 创建一个 UnMounter
	NewUnmounter(name string, podUID types.UID) (Unmounter, error)

	// ConstructVolumeSpec 由 volume name 得到一个 spec
	ConstructVolumeSpec(volumeName, volumePath string) (*Spec, error)

	// SupportsMountOption 表明是否支持挂载选项
	SupportsMountOption() bool

	// SupportsBulkVolumeVerification checks if volume plugin type is capable
	// of enabling bulk polling of all nodes. This can speed up verification of
	// attached volumes by quite a bit, but underlying pluging must support it.
	SupportsBulkVolumeVerification() bool
}
```

{{< admonition note "Mounter 的作用">}}
Mounter 并不是进行块设备的挂载的（这是 DeviceMounter 的工作），而是<important>为特定的 Pod.Volume 准备一个目录</important>，而该目录后续由 kubelet 挂载给 Pod 的各个容器。
{{< /admonition >}}

VolumePlugin 是一个最基本的 interface，在 VolumePlugin 之上还根据功能有着各样的 Plugin interface。
* **`PersistentVolumePlugin`** - Volume 支持保存持久数据
* **`RecyclableVolumePlugin`** - 支持 Recycle Volume
* **`DeletableVolumePlugin`** - 支持 Delete Volume
* **`ProvisionableVolumePlugin`** - 支持 Provision Volume
* **`DeviceMountableVolumePlugin`** - 需要先挂载 Device，才可以将 Volume 挂载给 Pod
* **`AttachableVolumePlugin`** - 需要先 Attach 到 Node，再挂载 Device，才可以将 Volume 挂载给 Pod
* **`ExpandableVolumePlugin`** - Volume 支持从控制面触发 Expand
* **`NodeExpandableVolumePlugin`** - Volume 支持 Node 触发 Expand
* **`VolumePluginWithAttachLimits`** - 限制 Node 能够 Attach Volume 的数量
* **`BlockVolumePlugin`** - 支持 Block Volume

```go
// PersistentVolumePlugin is an extended interface of VolumePlugin and is used
// by volumes that want to provide long term persistence of data
type PersistentVolumePlugin interface {
	VolumePlugin
	// GetAccessModes describes the ways a given volume can be accessed/mounted.
	GetAccessModes() []v1.PersistentVolumeAccessMode
}

// RecyclableVolumePlugin is an extended interface of VolumePlugin and is used
// by persistent volumes that want to be recycled before being made available
// again to new claims
type RecyclableVolumePlugin interface {
	VolumePlugin

	// Recycle knows how to reclaim this
	// resource after the volume's release from a PersistentVolumeClaim.
	// Recycle will use the provided recorder to write any events that might be
	// interesting to user. It's expected that caller will pass these events to
	// the PV being recycled.
	Recycle(pvName string, spec *Spec, eventRecorder recyclerclient.RecycleEventRecorder) error
}

// DeletableVolumePlugin is an extended interface of VolumePlugin and is used
// by persistent volumes that want to be deleted from the cluster after their
// release from a PersistentVolumeClaim.
type DeletableVolumePlugin interface {
	VolumePlugin
	// NewDeleter creates a new volume.Deleter which knows how to delete this
	// resource in accordance with the underlying storage provider after the
	// volume's release from a claim
	NewDeleter(spec *Spec) (Deleter, error)
}

// ProvisionableVolumePlugin is an extended interface of VolumePlugin and is
// used to create volumes for the cluster.
type ProvisionableVolumePlugin interface {
	VolumePlugin
	// NewProvisioner creates a new volume.Provisioner which knows how to
	// create PersistentVolumes in accordance with the plugin's underlying
	// storage provider
	NewProvisioner(options VolumeOptions) (Provisioner, error)
}

// AttachableVolumePlugin is an extended interface of VolumePlugin and is used for volumes that require attachment
// to a node before mounting.
type AttachableVolumePlugin interface {
	DeviceMountableVolumePlugin
	NewAttacher() (Attacher, error)
	NewDetacher() (Detacher, error)
	// CanAttach tests if provided volume spec is attachable
	CanAttach(spec *Spec) (bool, error)
}

// DeviceMountableVolumePlugin is an extended interface of VolumePlugin and is used
// for volumes that requires mount device to a node before binding to volume to pod.
type DeviceMountableVolumePlugin interface {
	VolumePlugin
	NewDeviceMounter() (DeviceMounter, error)
	NewDeviceUnmounter() (DeviceUnmounter, error)
	GetDeviceMountRefs(deviceMountPath string) ([]string, error)
	// CanDeviceMount determines if device in volume.Spec is mountable
	CanDeviceMount(spec *Spec) (bool, error)
}

// ExpandableVolumePlugin is an extended interface of VolumePlugin and is used for volumes that can be
// expanded via control-plane ExpandVolumeDevice call.
type ExpandableVolumePlugin interface {
	VolumePlugin
	ExpandVolumeDevice(spec *Spec, newSize resource.Quantity, oldSize resource.Quantity) (resource.Quantity, error)
	RequiresFSResize() bool
}

// NodeExpandableVolumePlugin is an expanded interface of VolumePlugin and is used for volumes that
// require expansion on the node via NodeExpand call.
type NodeExpandableVolumePlugin interface {
	VolumePlugin
	RequiresFSResize() bool
	// NodeExpand expands volume on given deviceMountPath and returns true if resize is successful.
	NodeExpand(resizeOptions NodeResizeOptions) (bool, error)
}

// VolumePluginWithAttachLimits is an extended interface of VolumePlugin that restricts number of
// volumes that can be attached to a node.
type VolumePluginWithAttachLimits interface {
	VolumePlugin
	// Return maximum number of volumes that can be attached to a node for this plugin.
	// The key must be same as string returned by VolumeLimitKey function. The returned
	// map may look like:
	//     - { "storage-limits-aws-ebs": 39 }
	//     - { "storage-limits-gce-pd": 10 }
	// A volume plugin may return error from this function - if it can not be used on a given node or not
	// applicable in given environment (where environment could be cloudprovider or any other dependency)
	// For example - calling this function for EBS volume plugin on a GCE node should
	// result in error.
	// The returned values are stored in node allocatable property and will be used
	// by scheduler to determine how many pods with volumes can be scheduled on given node.
	GetVolumeLimits() (map[string]int64, error)
	// Return volume limit key string to be used in node capacity constraints
	// The key must start with prefix storage-limits-. For example:
	//    - storage-limits-aws-ebs
	//    - storage-limits-csi-cinder
	// The key should respect character limit of ResourceName type
	// This function may be called by kubelet or scheduler to identify node allocatable property
	// which stores volumes limits.
	VolumeLimitKey(spec *Spec) string
}

// BlockVolumePlugin is an extend interface of VolumePlugin and is used for block volumes support.
type BlockVolumePlugin interface {
	VolumePlugin
	// NewBlockVolumeMapper creates a new volume.BlockVolumeMapper from an API specification.
	// Ownership of the spec pointer in *not* transferred.
	// - spec: The v1.Volume spec
	// - pod: The enclosing pod
	NewBlockVolumeMapper(spec *Spec, podRef *v1.Pod, opts VolumeOptions) (BlockVolumeMapper, error)
	// NewBlockVolumeUnmapper creates a new volume.BlockVolumeUnmapper from recoverable state.
	// - name: The volume name, as per the v1.Volume spec.
	// - podUID: The UID of the enclosing pod
	NewBlockVolumeUnmapper(name string, podUID types.UID) (BlockVolumeUnmapper, error)
	// ConstructBlockVolumeSpec constructs a volume spec based on the given
	// podUID, volume name and a pod device map path.
	// The spec may have incomplete information due to limited information
	// from input. This function is used by volume manager to reconstruct
	// volume spec by reading the volume directories from disk.
	ConstructBlockVolumeSpec(podUID types.UID, volumeName, volumePath string) (*Spec, error)
}
```

用一个类图总结一下：

{{< image src="img3.png" >}}

### 2.3 DynamicPluginProber 接口

大部分内置的插件都以代码方式实现了 VolumePlugin 接口，以静态方式注册。<important>而 FlexVolume 是由 DynamicPluginProber 接口进行 Probe 后，以动态方式注册。
```go
type ProbeEvent struct {
	Plugin     VolumePlugin // VolumePlugin that was added/updated/removed. if ProbeEvent.Op is 'ProbeRemove', Plugin should be nil
	PluginName string
	Op         ProbeOperation // The operation to the plugin
}

type DynamicPluginProber interface {
	Init() error

	// If an error occurs, events are undefined.
	Probe() (events []ProbeEvent, err error)
}
```

调用 Probe() 接口会去插件目录查找插件的可执行文件，并创建 **`flexVolumePlugin`** 类，**flexVolumePlugin 即实现了 VolumePlugin 接口**，可以注册到 VolumePluginMgr 中。

{{< admonition note "插件目录">}}
Probe() 接口查找的目录格式为 `<plugindir>/<vendor~driver>/<driver>`，`<plugindir>` 在 Kubelet 由参数 `--volume-plugin-dir` 指定，在 Controller Manager 由参数 `--flex-volume-plugin-dir`。

默认为路径 `/usr/libexec/kubernetes/kubelet-plugins/volume/exec`。
{{< /admonition >}}

DynamicPluginProber 周期性进行 Probe() 操作，注册新发现的 flexVolumePlugin，移除不存在的 flexVolumePlugin。

当调用 flexVolumePlugin 的操作接口（NewMounter、NewAttacher 等），会转发调用各个可执行文件的命令。例如：
```bash
$ ./uds
Usage:
  flexvoldrv [command]

Available Commands:
  help        Help about any command
  init        Flex volume init command.
  mount       Flex volume unmount command.
  unmount     Flex volume unmount command.
  version     Print version
```

### 2.4 VolumePluginMgr 类

**所有的 VolumePlugin 会注册到 VolumePluginMgr 对象中管理。**
```go
type VolumePluginMgr struct {
	mutex                     sync.RWMutex
	plugins                   map[string]VolumePlugin  // 静态注册的 VolumePlugin
	prober                    DynamicPluginProber
	probedPlugins             map[string]VolumePlugin // probe 探测出的 VolumePlugin
	loggedDeprecationWarnings sets.String
	Host                      VolumeHost
}
```

而大多数 Controller 使用时，就会先通过各个 Find 函数来查找可用的 VolumePlugin，然后调用 VolumePlugin 的方法进行实际的操作。
```go
// FindPluginBySpec 查找能够支持 spec 的 VolumePlugin
func (pm *VolumePluginMgr) FindPluginBySpec(spec *Spec) (VolumePlugin, error) {
	pm.mutex.RLock()
	defer pm.mutex.RUnlock()

	if spec == nil {
		return nil, fmt.Errorf("could not find plugin because volume spec is nil")
	}

	// 调用静态注册的 VolumePlugin.CanSupport() 函数筛选
	matches := []VolumePlugin{}
	for _, v := range pm.plugins {
		if v.CanSupport(spec) {
			matches = append(matches, v)
		}
	}

	// 进行一次 Probe Plugin
	pm.refreshProbedPlugins()

	// 调用动态注册的 VolumePlugin.CanSupport() 函数筛选
	for _, plugin := range pm.probedPlugins {
		if plugin.CanSupport(spec) {
			matches = append(matches, plugin)
		}
	}

	// 没有任何 Volume Plugin 能够处理
	if len(matches) == 0 {
		return nil, fmt.Errorf("no volume plugin matched")
	}

	// 多个 Volume Plugin 能够处理
	if len(matches) > 1 {
		matchedPluginNames := []string{}
		for _, plugin := range matches {
			matchedPluginNames = append(matchedPluginNames, plugin.GetPluginName())
		}
		return nil, fmt.Errorf("multiple volume plugins matched: %s", strings.Join(matchedPluginNames, ","))
	}

	// 返回唯一的 Volume Plugin
	// Issue warning if the matched provider is deprecated
	pm.logDeprecation(matches[0].GetPluginName())
	return matches[0], nil
}

// FindPluginByName 通过 plugin name 查找 Volume Plugin
func (pm *VolumePluginMgr) FindPluginByName(name string) (VolumePlugin, error) {
	pm.mutex.RLock()
	defer pm.mutex.RUnlock()

	// 查找静态注册的 VolumePlugin
	// Once we can get rid of legacy names we can reduce this to a map lookup.
	matches := []VolumePlugin{}
	if v, found := pm.plugins[name]; found {
		matches = append(matches, v)
	}

	// 进行一次 Probe Plugin
	pm.refreshProbedPlugins()

	// 查找动态注册的 VolumePlugin
	if plugin, found := pm.probedPlugins[name]; found {
		matches = append(matches, plugin)
	}

	// 没有任何 VolumePlugin 或者多个 VolumePlugin 视为错误
	if len(matches) == 0 {
		return nil, fmt.Errorf("no volume plugin matched name: %s", name)
	}
	if len(matches) > 1 {
		matchedPluginNames := []string{}
		for _, plugin := range matches {
			matchedPluginNames = append(matchedPluginNames, plugin.GetPluginName())
		}
		return nil, fmt.Errorf("multiple volume plugins matched: %s", strings.Join(matchedPluginNames, ","))
	}

	// 返回唯一的 VolumePlugin
	// Issue warning if the matched provider is deprecated
	pm.logDeprecation(matches[0].GetPluginName())
	return matches[0], nil
}

func (pm *VolumePluginMgr) FindPersistentPluginBySpec(spec *Spec) (PersistentVolumePlugin, error) {
	// ...
}

// 其他各个类型的 VolumePlugin ...
```


## 3 PV Controller

**`PV Controller`** 主要<important>负责 PV 与 PVC 的绑定，并负责了 PV 的创建与销毁</important>。

PV Controller 就是一个控制器，**监听着 PV 与 PVC 的事件，并对应进行 PV 与 PVC 的协调处理**。

### 3.1 触发逻辑

Controller 基本都是通过 Informer 实现，周期性与事件触发。

```go
// NewController 创建一个 PV Controller 对象
func NewController(p ControllerParameters) (*PersistentVolumeController, error) {
	// 构建 Controller 对象，基本的参数都来自于 ControllerParameters
	controller := &PersistentVolumeController{
		volumes:                       newPersistentVolumeOrderedIndex(),
		claims:                        cache.NewStore(cache.DeletionHandlingMetaNamespaceKeyFunc),
		kubeClient:                    p.KubeClient,
		eventRecorder:                 eventRecorder,
		runningOperations:             goroutinemap.NewGoRoutineMap(true /* exponentialBackOffOnError */),
		cloud:                         p.Cloud,
		enableDynamicProvisioning:     p.EnableDynamicProvisioning,
		clusterName:                   p.ClusterName,
		createProvisionedPVRetryCount: createProvisionedPVRetryCount,
		createProvisionedPVInterval:   createProvisionedPVInterval,
		claimQueue:                    workqueue.NewNamed("claims"),
		volumeQueue:                   workqueue.NewNamed("volumes"),
		resyncPeriod:                  p.SyncPeriod,
		operationTimestamps:           metrics.NewOperationStartTimeCache(),
	}

	// 初始化所有的 Volume Plugin
	if err := controller.volumePluginMgr.InitPlugins(p.VolumePlugins, nil /* prober */, controller); err != nil {
		return nil, fmt.Errorf("could not initialize volume plugins for PersistentVolume Controller: %w", err)
	}

	// 监听 PV 事件，注册回调
	p.VolumeInformer.Informer().AddEventHandler(
		cache.ResourceEventHandlerFuncs{
			AddFunc:    func(obj interface{}) { controller.enqueueWork(controller.volumeQueue, obj) },
			UpdateFunc: func(oldObj, newObj interface{}) { controller.enqueueWork(controller.volumeQueue, newObj) },
			DeleteFunc: func(obj interface{}) { controller.enqueueWork(controller.volumeQueue, obj) },
		},
	)
	controller.volumeLister = p.VolumeInformer.Lister()
	controller.volumeListerSynced = p.VolumeInformer.Informer().HasSynced

	// 监听 PVC 事件，注册回调
	p.ClaimInformer.Informer().AddEventHandler(
		cache.ResourceEventHandlerFuncs{
			AddFunc:    func(obj interface{}) { controller.enqueueWork(controller.claimQueue, obj) },
			UpdateFunc: func(oldObj, newObj interface{}) { controller.enqueueWork(controller.claimQueue, newObj) },
			DeleteFunc: func(obj interface{}) { controller.enqueueWork(controller.claimQueue, obj) },
		},
	)
	controller.claimLister = p.ClaimInformer.Lister()
	controller.claimListerSynced = p.ClaimInformer.Informer().HasSynced

	// 构建对象其他属性
    // ...

	return controller, nil
}

// enqueueWork enqueueWork 将对象的 key 放入队列
func (ctrl *PersistentVolumeController) enqueueWork(queue workqueue.Interface, obj interface{}) {
	// Beware of "xxx deleted" events
	if unknown, ok := obj.(cache.DeletedFinalStateUnknown); ok && unknown.Obj != nil {
		obj = unknown.Obj
	}
	objName, err := controller.KeyFunc(obj)
	if err != nil {
		return
	}
	queue.Add(objName)
}
```

可以看到，PV/PVC 的事件回调都放入了对应的 `controller.volumeQueue`/`controller.claimQueue` 队列。

在 PV Controller 的 Run 函数时，启动了三个协程进行处理，分别进行：
* resync
* 处理 volumeQueue
* 处理 claimQueue。
```go
// Run 启动 Controller 的控制循环
func (ctrl *PersistentVolumeController) Run(stopCh <-chan struct{}) {
	defer utilruntime.HandleCrash()
	defer ctrl.claimQueue.ShutDown()
	defer ctrl.volumeQueue.ShutDown()

	// 等待 cache 填充
	if !cache.WaitForNamedCacheSync("persistent volume", stopCh, ctrl.volumeListerSynced, ctrl.claimListerSynced, ctrl.classListerSynced, ctrl.podListerSynced, ctrl.NodeListerSynced) {
		return
	}

	// 初始化 ctrl.Store
	ctrl.initializeCaches(ctrl.volumeLister, ctrl.claimLister)

	// 执行各个工作函数
	go wait.Until(ctrl.resync, ctrl.resyncPeriod, stopCh)
	go wait.Until(ctrl.volumeWorker, time.Second, stopCh)
	go wait.Until(ctrl.claimWorker, time.Second, stopCh)

	<-stopCh
}
```

先看下 resync，其作用就是为了比 Informer 更短周期的进行周期性的控制循环。因为 Shared Informer 的触发周期是所有使用者相同的，所以自己实现了一个 resync 函数。
```go
// resync supplements short resync period of shared informers - we don't want
// all consumers of PV/PVC shared informer to have a short resync period,
// therefore we do our own.
func (ctrl *PersistentVolumeController) resync() {
	// List 所有 PVC，放入队列
	pvcs, err := ctrl.claimLister.List(labels.NewSelector())
	if err != nil {
		return
	}
	for _, pvc := range pvcs {
		ctrl.enqueueWork(ctrl.claimQueue, pvc)
	}

	// List 所有 PV，放入队列
	pvs, err := ctrl.volumeLister.List(labels.NewSelector())
	if err != nil {
		return
	}
	for _, pv := range pvs {
		ctrl.enqueueWork(ctrl.volumeQueue, pv)
	}
}
```

看下如何处理 PV 事件。可以看到，后续调用 `updateVolume()` 与 `deleteVolume()` 方法进行处理。
```go
func (ctrl *PersistentVolumeController) volumeWorker() {
	// 工作函数
	workFunc := func() bool {
		// 从队列中取出元素
		keyObj, quit := ctrl.volumeQueue.Get()
		if quit {
			return true
		}
		defer ctrl.volumeQueue.Done(keyObj)
		key := keyObj.(string)

		// 解析出 namespace 与 name
		_, name, err := cache.SplitMetaNamespaceKey(key)
		if err != nil {
			return false
		}

		// 查询 PV
		volume, err := ctrl.volumeLister.Get(name)
		if err == nil {
			// 查询的到 PV，那么事件为 add/update/sync，调用 updateVolume 处理
			ctrl.updateVolume(volume)
			return false
		}
		if !errors.IsNotFound(err) {
			return false
		}

		// PV 不存在，说明 PV 对象已经被删除，那么 Controller 负责执行删除的逻辑
        // 通过自己的 Store，可以保证 Controller 能够在删除失败的情况下不断重试。

		// 从自己的 Store 中查询
		volumeObj, found, err := ctrl.volumes.store.GetByKey(key)
		if err != nil {
			return false
		}
        // 如果不存在所有 Controller 已经处理了该 PV 的删除
		if !found {
			// The controller has already processed the delete event and
			// deleted the volume from its cache
			return false
		}

		// 如果 Store 中找到，说明 PV 删除还未处理，调用 deleteVolume
		volume, ok := volumeObj.(*v1.PersistentVolume)
		if !ok {
			return false
		}
		ctrl.deleteVolume(volume)
		return false
	}

	// 永久执行工作函数，直到退出
	for {
		if quit := workFunc(); quit {
			return
		}
	}
}
```

PVC 的处理也是类似，后续调用 `updateClaim()` 与 `deleteClaim()` 进行处理。
```go
func (ctrl *PersistentVolumeController) claimWorker() {
	workFunc := func() bool {
		keyObj, quit := ctrl.claimQueue.Get()
		if quit {
			return true
		}
		defer ctrl.claimQueue.Done(keyObj)
		key := keyObj.(string)

		namespace, name, err := cache.SplitMetaNamespaceKey(key)
		if err != nil {
			return false
		}
		claim, err := ctrl.claimLister.PersistentVolumeClaims(namespace).Get(name)
		if err == nil {
			ctrl.updateClaim(claim)
			return false
		}
		if !errors.IsNotFound(err) {
			return false
		}

		claimObj, found, err := ctrl.claims.GetByKey(key)
		if err != nil {
			return false
		}
		if !found {
			return false
		}
		claim, ok := claimObj.(*v1.PersistentVolumeClaim)
		if !ok {
			return false
		}
		ctrl.deleteClaim(claim)
		return false
	}
	for {
		if quit := workFunc(); quit {
			return
		}
	}
}
```

最后看下各个 updateXXX 如何处理。最终，就是调用` syncVolume()` `syncClaim()` 函数。
```go
func (ctrl *PersistentVolumeController) updateVolume(volume *v1.PersistentVolume) {
    // 更新 Store
    new, err := ctrl.storeVolumeUpdate(volume)
	if err != nil {
	}
	if !new {
		return
	}

    // 执行 sync
	err = ctrl.syncVolume(volume)
	if err != nil {
		if errors.IsConflict(err) {
			// Version conflict error happens quite often and the controller
			// recovers from it easily.
			klog.V(3).Infof("could not sync volume %q: %+v", volume.Name, err)
		} else {
			klog.Errorf("could not sync volume %q: %+v", volume.Name, err)
		}
	}
}

func (ctrl *PersistentVolumeController) updateClaim(claim *v1.PersistentVolumeClaim) {
    // 更新 Store
	new, err := ctrl.storeClaimUpdate(claim)
	if err != nil {
	}
	if !new {
		return
	}
    // 执行 sync
	err = ctrl.syncClaim(claim)
	if err != nil {
		if errors.IsConflict(err) {
			// Version conflict error happens quite often and the controller
			// recovers from it easily.
			klog.V(3).Infof("could not sync claim %q: %+v", claimToClaimKey(claim), err)
		} else {
			klog.Errorf("could not sync volume %q: %+v", claimToClaimKey(claim), err)
		}
	}
}
```

### 3.2 PV 的控制循环

PV 的控制循环主要**负责了 PV 的状态更新，以及 PV 的回收**。而 PV 创建与绑定相关都是由后续的 PVC 的控制循环处理。

PV 控制循环大概可以分为几个步骤：
* 检查 PV 的绑定情况：
  1. PV 还未绑定 PVC (PV spec.claimRef == nil)，更新 PV status.phase 为 "Available"。
  2. PV 绑定了 PVC (PV spec.claimRef == nil)：
     1. 绑定的 PVC 还未创建 (PV spec.claimRef.UID == "")，更新 PV status.phase 为 "Available"，后续不执行。
     2. 绑定的 PV 创建过了 (PV spec.claimRef.UID != "")，检查实际的 PV 是否存在。
   
        会查询三个地方，一个查询成功即表明存在
           * Controller 自己维护的 Store
           * Informer.Cache
           * APiServer
* 如果 PV 已经绑定了 PVC，但是 PVC 不存在，说明 PVC 已经被删除了
  1. 更新 PV status.phase 为 "Released"。
  2. 根据 PV spec.persistentVolumeReclaimPolicy，执行对 PV 的操作：
     * "Retain" - 不执行任何操作
     * "Recycle" - 调用 CSI 插件回收 Volume，解除 PV 与 PVC 的绑定
     * "Delete" - 调用 CSI 插件删除 Volume，并删除 PV 对象
* 如果 PV 已经绑定了 PVC，但是 PVC 还未绑定 PV
  1. 检查 PVC 与 PV 的 VolumeMode 是否匹配
  2. 触发一次 syncClaim
* 如果 PV 与 PVC 相互绑定，那么更新 PV status.phase 为 "Bound"。
* 如果 PV 与 PVC 绑定不对称，那么解除绑定，允许的情况下还会删除 PV。

```go
func (ctrl *PersistentVolumeController) syncVolume(volume *v1.PersistentVolume) error {
	// ..

	if volume.Spec.ClaimRef == nil {
		// PV Spec 没有指定绑定 PVC，那么更新 Volume Phase 为 "Available"
		if _, err := ctrl.updateVolumePhase(volume, v1.VolumeAvailable, ""); err != nil {
			return err
		}
		return nil
	} else /* pv.Spec.ClaimRef != nil */ {
		// PV Spec 指定了绑定了 PVC

		// 绑定的 PVC 还未存在（没有 UID），表明给 PV 是预留的，更新 Volume Phase 为 "Available"
		if volume.Spec.ClaimRef.UID == "" {
			if _, err := ctrl.updateVolumePhase(volume, v1.VolumeAvailable, ""); err != nil {
				// Nothing was saved; we will fall back into the same
				// condition in the next call to this method
				return err
			}
			return nil
		}

		// 到这里，说明绑定的 PVC 已经存在
		// Get the PVC by _name_
		var claim *v1.PersistentVolumeClaim
		claimName := claimrefToClaimKey(volume.Spec.ClaimRef)

		// 查询 PVC 是否还存在
		obj, found, err := ctrl.claims.GetByKey(claimName)
		if err != nil {
			return err
		}
		if !found {
			// 如果 PVC 在自己的 store 不存在，进行 double check
			// If the PV was created by an external PV provisioner or
			// bound by external PV binder (e.g. kube-scheduler), it's
			// possible under heavy load that the corresponding PVC is not synced to
			// controller local cache yet. So we need to double-check PVC in
			//   1) informer cache
			//   2) apiserver if not found in informer cache
			// to make sure we will not reclaim a PV wrongly.
			// Note that only non-released and non-failed volumes will be
			// updated to Released state when PVC does not exist.
			if volume.Status.Phase != v1.VolumeReleased && volume.Status.Phase != v1.VolumeFailed {
				obj, err = ctrl.claimLister.PersistentVolumeClaims(volume.Spec.ClaimRef.Namespace).Get(volume.Spec.ClaimRef.Name)
				if err != nil && !apierrors.IsNotFound(err) {
					return err
				}
				found = !apierrors.IsNotFound(err)
				if !found {
					obj, err = ctrl.kubeClient.CoreV1().PersistentVolumeClaims(volume.Spec.ClaimRef.Namespace).Get(context.TODO(), volume.Spec.ClaimRef.Name, metav1.GetOptions{})
					if err != nil && !apierrors.IsNotFound(err) {
						return err
					}
					found = !apierrors.IsNotFound(err)
				}
			}
		}
		if !found {
			klog.V(4).Infof("synchronizing PersistentVolume[%s]: claim %s not found", volume.Name, claimrefToClaimKey(volume.Spec.ClaimRef))
			// Fall through with claim = nil
		} else {
			var ok bool
			claim, ok = obj.(*v1.PersistentVolumeClaim)
			if !ok {
				return fmt.Errorf("cannot convert object from volume cache to volume %q!?: %#v", claim.Spec.VolumeName, obj)
			}
			klog.V(4).Infof("synchronizing PersistentVolume[%s]: claim %s found: %s", volume.Name, claimrefToClaimKey(volume.Spec.ClaimRef), getClaimStatusForLogging(claim))
		}
		if claim != nil && claim.UID != volume.Spec.ClaimRef.UID {
			claim = nil
		}

		if claim == nil {
			// 如果最终检查的该 PVC 还是不存在，那么 PVC 已经被删除了

			// 更新 Volume Phase 为 "Released"
			if volume.Status.Phase != v1.VolumeReleased && volume.Status.Phase != v1.VolumeFailed {
				if volume, err = ctrl.updateVolumePhase(volume, v1.VolumeReleased, ""); err != nil {
					// Nothing was saved; we will fall back into the same condition
					// in the next call to this method
					return err
				}
			}

			// 根据 ReclaimPolicy 决定是否销毁 PV
			if err = ctrl.reclaimVolume(volume); err != nil {
				return err
			}

			// 如果 ReclaimPolicy 为保留，后续操作不执行
			if volume.Spec.PersistentVolumeReclaimPolicy == v1.PersistentVolumeReclaimRetain {
				klog.V(4).Infof("PersistentVolume[%s] references a claim %q (%s) that is not found", volume.Name, claimrefToClaimKey(volume.Spec.ClaimRef), volume.Spec.ClaimRef.UID)
			}
			return nil
		} else if claim.Spec.VolumeName == "" {
			// PVC 还未绑定 PV（PV 已经绑定了 PVC）

			// 检查 PV 与 PVC 的 VolumeMode（Filesystem/Block） 是否匹配
			// 如果不匹配不执行后续的 sync 操作
			if pvutil.CheckVolumeModeMismatches(&claim.Spec, &volume.Spec) {
				// Skipping syncClaim
				return nil
			}

			// 后续由 PVC 的控制循环处理
			if metav1.HasAnnotation(volume.ObjectMeta, pvutil.AnnBoundByController) {
				// The binding is not completed; let PVC sync handle it
				klog.V(4).Infof("synchronizing PersistentVolume[%s]: volume not bound yet, waiting for syncClaim to fix it", volume.Name)
			} else {
				// Dangling PV; try to re-establish the link in the PVC sync
				klog.V(4).Infof("synchronizing PersistentVolume[%s]: volume was bound and got unbound (by user?), waiting for syncClaim to fix it", volume.Name)
			}
			ctrl.claimQueue.Add(claimToClaimKey(claim))
			return nil
		} else if claim.Spec.VolumeName == volume.Name {
			// PVC 绑定的就是该 PV，更新 PV Phase 为 "Bound"
			if _, err = ctrl.updateVolumePhase(volume, v1.VolumeBound, ""); err != nil {
				return err
			}
			return nil
		} else {
			// PVC 绑定的不是该 PV

			if metav1.HasAnnotation(volume.ObjectMeta, pvutil.AnnDynamicallyProvisioned) && volume.Spec.PersistentVolumeReclaimPolicy == v1.PersistentVolumeReclaimDelete {
				// 如果 PV 是动态创建的（由 StorageClass 而创建的），并且允许删除，那么回收 PV
				if volume.Status.Phase != v1.VolumeReleased && volume.Status.Phase != v1.VolumeFailed {
					// Also, log this only once:
					klog.V(2).Infof("dynamically volume %q is released and it will be deleted", volume.Name)
					if volume, err = ctrl.updateVolumePhase(volume, v1.VolumeReleased, ""); err != nil {
						// Nothing was saved; we will fall back into the same condition
						// in the next call to this method
						return err
					}
				}
				if err = ctrl.reclaimVolume(volume); err != nil {
					// Deletion failed, we will fall back into the same condition
					// in the next call to this method
					return err
				}
				return nil
			} else {
				// PV 不能允许被删除，那么就仅仅将绑定关系解除
				// Volume is bound to a claim, but the claim is bound elsewhere
				// and it's not dynamically provisioned.
				if metav1.HasAnnotation(volume.ObjectMeta, pvutil.AnnBoundByController) {
					// This is part of the normal operation of the controller; the
					// controller tried to use this volume for a claim but the claim
					// was fulfilled by another volume. We did this; fix it.
					if err = ctrl.unbindVolume(volume); err != nil {
						return err
					}
					return nil
				} else {
					// The PV must have been created with this ptr; leave it alone.
					// This just updates the volume phase and clears
					// volume.Spec.ClaimRef.UID. It leaves the volume pre-bound
					// to the claim.
					if err = ctrl.unbindVolume(volume); err != nil {
						return err
					}
					return nil
				}
			}
		}
	}
}
```

### 3.3 PVC 的控制循环

PVC 的控制循环**负责 PVC 与 PV 的绑定，如果 PVC 指定 StorageClass 还可能创建一个 PV**。

PVC 控制循环分为两种情况：
* PVC 已经由 Controller 绑定了 PV（判断 PVC annotation "pv.kubernetes.io/bind-completed"），执行 `syncBoundClaim()` 方法。
* PVC 还未由 Controller 绑定了 PV（判断 PVC annotation "pv.kubernetes.io/bind-completed"），执行 `syncUnboundClaim()` 方法。

先看简单的 `syncBoundClaim()` 逻辑，主要就是检查 PV 与 PVC 的绑定关系是否正常。
1. PVC spec.volumeName == ""，说明绑定关系丢失了，更新 PVC status.phase 为 "Lost"。
2. 查询对应绑定的 PV，如果 PV 不存在，说明 PV 丢失，更新 PVC status.phase 为 "Lost"。
3. 对应的 PV 存在，但是 PV 未绑定该 PVC（PV spec.claimRef == nil），那么执行绑定操作。
4. 对应的 PV 存在，但是 PV 绑定的不是该 PVC，说明绑定关系丢失了，更新 PVC status.phase 为 "Lost"。

```go
func (ctrl *PersistentVolumeController) syncBoundClaim(claim *v1.PersistentVolumeClaim) error {
	// Annotation 表明 PVC 已经绑定了 PV，但是 PVC Spec.VolumeName 为空，说明 PV 丢失了，更新 Phase 为 "Lost"
	if claim.Spec.VolumeName == "" {
		if _, err := ctrl.updateClaimStatusWithEvent(claim, v1.ClaimLost, nil, v1.EventTypeWarning, "ClaimLost", "Bound claim has lost reference to PersistentVolume. Data on the volume is lost!"); err != nil {
			return err
		}
		return nil
	}

	// 查询 PVC 绑定的 PV
	obj, found, err := ctrl.volumes.store.GetByKey(claim.Spec.VolumeName)
	if err != nil {
		return err
	}

	if !found {
		// 对应的 PV 不存在，表明 PV 丢失了，更新 Phase 为 "Lost"
		if _, err = ctrl.updateClaimStatusWithEvent(claim, v1.ClaimLost, nil, v1.EventTypeWarning, "ClaimLost", "Bound claim has lost its PersistentVolume. Data on the volume is lost!"); err != nil {
			return err
		}
		return nil
	} else {
		volume, ok := obj.(*v1.PersistentVolume)
		if !ok {
			return fmt.Errorf("cannot convert object from volume cache to volume %q!?: %#v", claim.Spec.VolumeName, obj)
		}

		if volume.Spec.ClaimRef == nil {
			// 对应 PV 还未绑定，那么将对应 PV 绑定到自身 PVC
			if err = ctrl.bind(volume, claim); err != nil {
				return err
			}
			return nil
		} else if volume.Spec.ClaimRef.UID == claim.UID {
			// PV 与 PVC 都相互绑定了，还是调用一个 bind
			// NOTE: syncPV can handle this so it can be left out.
			// NOTE: bind() call here will do nothing in most cases as
			// everything should be already set.
			if err = ctrl.bind(volume, claim); err != nil {
				return err
			}
			return nil
		} else {
			// 对应的 PV 绑定的不是该 PVC，更新 Phase 为 "Lost"
			if _, err = ctrl.updateClaimStatusWithEvent(claim, v1.ClaimLost, nil, v1.EventTypeWarning, "ClaimMisbound", "Two claims are bound to the same volume, this one is bound incorrectly"); err != nil {
				return err
			}
			return nil
		}
	}
}
```

`syncUnboundClaim()` 主要负责 PV 与 PVC 的绑定关系，如果未绑定会查找合适的 PV。
* PV spec.volumeName == ""，尝试绑定一个合适的 PV
    1. 无法查找到合适的 PV
       1. 如果 PVC 指定了 StorageClass，那么尝试创建一个 PV。
       2. 其他情况，PVC status.phase 为 "Pending"。
    2. 找到了合适的 PV，执行绑定操作
* PV spec.volumeName != ""，说明是用户部署时指定了 PV，那么执行绑定操作。

```go
func (ctrl *PersistentVolumeController) syncUnboundClaim(claim *v1.PersistentVolumeClaim) error {
	if claim.Spec.VolumeName == "" {
		// 确实未绑定 PV

		// 确认 BindingMode 是否为 "WaitForFirstConsumer"
		delayBinding, err := pvutil.IsDelayBindingMode(claim, ctrl.classLister)
		if err != nil {
			return err
		}

		// 查找合适的 PV
		volume, err := ctrl.volumes.findBestMatchForClaim(claim, delayBinding)
		if err != nil {
			return fmt.Errorf("error finding PV for claim %q: %w", claimToClaimKey(claim), err)
		}
		if volume == nil {
			// 没找到合适的 PV

			switch {
			case delayBinding && !pvutil.IsDelayBindingProvisioning(claim):
				// 延迟绑定，并且还未调度到 Node（通过 PVC annotation "volume.kubernetes.io/selected-node" 判断）
				if err = ctrl.emitEventForUnboundDelayBindingClaim(claim); err != nil {
					return err
				}
			case storagehelpers.GetPersistentVolumeClaimClass(claim) != "":
				// PVC 指定了 StorageClass，那么尝试创建一个 PV
				if err = ctrl.provisionClaim(claim); err != nil {
					return err
				}
				return nil
			default:
				ctrl.eventRecorder.Event(claim, v1.EventTypeNormal, events.FailedBinding, "no persistent volumes available for this claim and no storage class is set")
			}

			// 更新 PVC 状态为 Pending
			if _, err = ctrl.updateClaimStatus(claim, v1.ClaimPending, nil); err != nil {
				return err
			}
			return nil
		} else /* pv != nil */ {
			// 找了一个合适的 PV，执行绑定操作

			claimKey := claimToClaimKey(claim)
			if err = ctrl.bind(volume, claim); err != nil {
				return err
			}
			return nil
		}
	} else /* pvc.Spec.VolumeName != nil */ {
		// PVC 已经指定绑定的 PV，说明是用户指定的

		// 查找对应 PV 是否存在
		obj, found, err := ctrl.volumes.store.GetByKey(claim.Spec.VolumeName)
		if err != nil {
			return err
		}
		if !found {
			// 不存在，PVC Phase 更新为 Pending
			if _, err = ctrl.updateClaimStatus(claim, v1.ClaimPending, nil); err != nil {
				return err
			}
			return nil
		} else {
			// 存在，执行绑定操作

			// ...
		}
	}
}
```

## 4 AttachDetach Controller

前面 PV Controller 负责了 PV 的创建与销毁，以及 PV 与 PVC 的绑定与解绑。当 PVC 绑定后，并被 Pod 使用后，<important>由 AttachDetach Controller 负责将其 Attach 到 Node 上，以及负责从 Node 上 Detach。

{{< admonition note Note>}}
Attach 表示将 Volume 提供到 Node 上，以便后续 Node 上的 kubelet 进行处理。 
{{< /admonition >}}

整体 AttachDetach Controller 还是基于 Controller 模式，但是其核心是依赖于两个 map 的数据进行处理：`actualStateOfWorld` 与 `desiredStateOfWorld`。处理流程就是，根据当前状态，执行操作，向期望状态靠拢。

因此，AttachDetach Controller 的核心逻辑可以分为三个部分：
* 更新 `desiredStateOfWorld`
* 更新 `actualStateOfWorld`
* Reconcile

### 4.1 更新 desiredStateOfWorld

`desiredStateOfWorld` **记录 Volume 的期望状态**，即哪些 Volume 需要 Attach 到哪些 Node。

所有的期望状态来自于 Pod 的 Spec 定义，状态更新来自于：
* 监听 Pod/PVC 的事件，
* 周期性遍历所有的 Pod

总之最终都会收敛到同一个函数 `ProcessPodVolumes()` 进行处理。

```go
func ProcessPodVolumes(pod *v1.Pod, addVolumes bool, desiredStateOfWorld cache.DesiredStateOfWorld, volumePluginMgr *volume.VolumePluginMgr, pvcLister corelisters.PersistentVolumeClaimLister, pvLister corelisters.PersistentVolumeLister, csiMigratedPluginManager csimigration.PluginManager, csiTranslator csimigration.InTreeToCSITranslator) {
	if pod == nil {
		return
	}

	// 如果 Pod 没有使用 Volume，啥也不干
	if len(pod.Spec.Volumes) <= 0 {
		klog.V(10).Infof("Skipping processing of pod %q/%q: it has no volumes.",
			pod.Namespace,
			pod.Name)
		return
	}

	// 查询是否调度到了一个存在的 Node
	nodeName := types.NodeName(pod.Spec.NodeName)
	if nodeName == "" {
		return
	} else if !desiredStateOfWorld.NodeExists(nodeName) {
		// If the node the pod is scheduled to does not exist in the desired
		// state of the world data structure, that indicates the node is not
		// yet managed by the controller. Therefore, ignore the pod.
		return
	}

	// 处理 Pod Spec 中的 Volume
	for _, podVolume := range pod.Spec.Volumes {
		// 创建 Volume 对象
		volumeSpec, err := CreateVolumeSpec(podVolume, pod, nodeName, volumePluginMgr, pvcLister, pvLister, csiMigratedPluginManager, csiTranslator)
		if err != nil {
			klog.V(10).Infof(
				"Error processing volume %q for pod %q/%q: %v",
				podVolume.Name,
				pod.Namespace,
				pod.Name,
				err)
			continue
		}

		// 找到对应的 CSI Plugin
		attachableVolumePlugin, err :=
			volumePluginMgr.FindAttachablePluginBySpec(volumeSpec)
		if err != nil || attachableVolumePlugin == nil {
			klog.V(10).Infof(
				"Skipping volume %q for pod %q/%q: it does not implement attacher interface. err=%v",
				podVolume.Name,
				pod.Namespace,
				pod.Name,
				err)
			continue
		}

		uniquePodName := util.GetUniquePodName(pod)
		if addVolumes {
			// 添加 Volume 到 desiredStateOfWorld
			_, err := desiredStateOfWorld.AddPod(
				uniquePodName, pod, volumeSpec, nodeName)

		} else {
			// 移除 Volume 从 desiredStateOfWorld
			// Remove volume from desired state of world
			uniqueVolumeName, err := util.GetUniqueVolumeNameFromSpec(
				attachableVolumePlugin, volumeSpec)
			if err != nil {
				continue
			}
			desiredStateOfWorld.DeletePod(
				uniquePodName, uniqueVolumeName, nodeName)
		}
	}

	return
}
```

### 4.2 更新 actualStateOfWorld

`actualStateOfWorld` **记录 Volume 的当前状态**，即 Node 上的 Attached Volume。

其状态更新本质来自于：
* 执行 Volume 操作后，更新 `actualStateOfWorld`；
* 启动时与后续周期性执行 sync，更新 `actualStateOfWorld`；
* 监听 Node 的 Add/Update/Delete 事件，更新 `actualStateOfWorld`；

最终操作会收敛到函数 `processVolumesInUse()`。
```go
// nodeAdd 处理 Node 添加事件
func (adc *attachDetachController) nodeAdd(obj interface{}) {
	node, ok := obj.(*v1.Node)

	// 转为 Node 的更新处理
	nodeName := types.NodeName(node.Name)
	adc.nodeUpdate(nil, obj)

	// 更新 actualStateOfWorld 的 Node
	adc.actualStateOfWorld.SetNodeStatusUpdateNeeded(nodeName)
}

// nodeUpdate 处理 Node 更新事件
func (adc *attachDetachController) nodeUpdate(oldObj, newObj interface{}) {
	node, ok := newObj.(*v1.Node)

	// Node 添加到 desiredStateOfWorld
	nodeName := types.NodeName(node.Name)
	adc.addNodeToDswp(node, nodeName)

	adc.processVolumesInUse(nodeName, node.Status.VolumesInUse)
}

// nodeDelete 处理 Node Delete 事件
func (adc *attachDetachController) nodeDelete(obj interface{}) {
	node, ok := obj.(*v1.Node)

	// 从 desiredStateOfWorld 删除 Node
	nodeName := types.NodeName(node.Name)
	err := adc.desiredStateOfWorld.DeleteNode(nodeName)
	
	adc.processVolumesInUse(nodeName, node.Status.VolumesInUse)
}

func (adc *attachDetachController) processVolumesInUse(
	nodeName types.NodeName, volumesInUse []v1.UniqueVolumeName) {
	klog.V(4).Infof("processVolumesInUse for node %q", nodeName)

	// 获取 Node 上已经 Attached Volume
	for _, attachedVolume := range adc.actualStateOfWorld.GetAttachedVolumesForNode(nodeName) {
		mounted := false

		// 检查 Volume 是否是 Attached
		for _, volumeInUse := range volumesInUse {
			if attachedVolume.VolumeName == volumeInUse {
				mounted = true
				break
			}
		}

		// 记录到 actualStateOfWorld
		err := adc.actualStateOfWorld.SetVolumeMountedByNode(attachedVolume.VolumeName, nodeName, mounted)
		if err != nil {
			klog.Warningf(
				"SetVolumeMountedByNode(%q, %q, %v) returned an error: %v",
				attachedVolume.VolumeName, nodeName, mounted, err)
		}
	}
}
```

启动时与周期调用 Reconcile 时，都会执行一部分 sync 逻辑，其中会遍历所有 Node 的 Volume，进行检查并更新 actualStateOfWorld。
```go
func (rc *reconciler) sync() {
	defer rc.updateSyncTime()
	rc.syncStates()
}

func (rc *reconciler) updateSyncTime() {
	rc.timeOfLastSync = time.Now()
}

func (rc *reconciler) syncStates() {
	volumesPerNode := rc.actualStateOfWorld.GetAttachedVolumesPerNode()
	rc.attacherDetacher.VerifyVolumesAreAttached(volumesPerNode, rc.actualStateOfWorld)
}
```

### 4.3 Reconcile

核心的 Reconcile 中主要执行了两个操作：
* **`Detach Volume`** - Attached Volume 存在于 `actualStateOfWorld`，但是不存在于 `desiredStateOfWorld`
* **`Attach Volume`** - Volume 存在于 `desiredStateOfWorld`，但是 `actualStateOfWorld` 中状态为 Unattached

```go
func (rc *reconciler) reconcile() {
	for _, attachedVolume := range rc.actualStateOfWorld.GetAttachedVolumes() {
		if !rc.desiredStateOfWorld.VolumeExists(
			attachedVolume.VolumeName, attachedVolume.NodeName) {

			// 如果 Attached Volume 存在于 actualStateOfWorld，但是不存在于 desiredStateOfWorld
			// 表明 Volume 需要 detach

			// ...

			attachState := rc.actualStateOfWorld.GetAttachState(attachedVolume.VolumeName, attachedVolume.NodeName)
			if attachState == cache.AttachStateDetached {
				// 如果 attachState 已经是 Detached 了，跳过处理
				continue
			}

			// ...

			// 将 Volume 标记为 detached
			err = rc.actualStateOfWorld.RemoveVolumeFromReportAsAttached(attachedVolume.VolumeName, attachedVolume.NodeName)

			// 更新 Node Status
			err = rc.nodeStatusUpdater.UpdateNodeStatuses()
			if err != nil {
				continue
			}

			// 执行 detach Volume
			klog.V(5).Infof(attachedVolume.GenerateMsgDetailed("Starting attacherDetacher.DetachVolume", ""))
			verifySafeToDetach := !timeout
			err = rc.attacherDetacher.DetachVolume(attachedVolume.AttachedVolume, verifySafeToDetach, rc.actualStateOfWorld)

			// ...
		}
	}

	// attach Volume
	rc.attachDesiredVolumes()

	// Update Node Status
	err := rc.nodeStatusUpdater.UpdateNodeStatuses()
}


func (rc *reconciler) attachDesiredVolumes() {
	// Ensure volumes that should be attached are attached.
	for _, volumeToAttach := range rc.desiredStateOfWorld.GetVolumesToAttach() {
		// 遍历 desiredStateOfWorld

		// ...

		attachState := rc.actualStateOfWorld.GetAttachState(volumeToAttach.VolumeName, volumeToAttach.NodeName)
		if attachState == cache.AttachStateAttached {
			// Volume/Node exists, touch it to reset detachRequestedTime
			rc.actualStateOfWorld.ResetDetachRequestTime(volumeToAttach.VolumeName, volumeToAttach.NodeName)
			continue
		}

		// ...

		err := rc.attacherDetacher.AttachVolume(volumeToAttach.VolumeToAttach, rc.actualStateOfWorld)
	}
}
```

## 5 Kubelet 的处理

Kubelet 中对 Volume 的处理也是基于 [**AttachDetach Controller**](#4-attachdetach-controller) 的方式，依赖于两个 map `的数据进行周期性协调：actualStateOfWorld` 与 `desiredStateOfWorld`。

我们不再说明 `actualStateOfWorld` 与 `desiredStateOfWorld` 的填充，只看下其 Reconcile 流程。

### 5.1 Reconcile

reconcile() 整体逻辑就是三步：
1. `unmountVolumes()`
2. `mountAttachVolumes()`
3. `unmountDetachDevices()`
```go
func (rc *reconciler) reconcile() {
	// Unmounts are triggered before mounts so that a volume that was
	// referenced by a pod that was deleted and is now referenced by another
	// pod is unmounted from the first pod before being mounted to the new
	// pod.
	rc.unmountVolumes()

	// Next we mount required volumes. This function could also trigger
	// attach if kubelet is responsible for attaching volumes.
	// If underlying PVC was resized while in-use then this function also handles volume
	// resizing.
	rc.mountAttachVolumes()

	// Ensure devices that should be detached/unmounted are detached/unmounted.
	rc.unmountDetachDevices()
}
```

#### 5.1.1 unmountVolumes

`unmountVolumes` **负责 unmount 需要的 volume**，通过判断 mounted volume 是否存在于 `actualStateOfWorld`。
```go
func (rc *reconciler) unmountVolumes() {
	// 遍历当前节点所有的 mounted volume
	for _, mountedVolume := range rc.actualStateOfWorld.GetAllMountedVolumes() {
		// 检查该 volume 与 pod 是否存在于 desiredStateOfWorld
		if !rc.desiredStateOfWorld.PodExistsInVolume(mountedVolume.PodName, mountedVolume.VolumeName) {
			// 进行 unmount volume 操作（底层调用的是 VolumePlugin.TearDown）
			err := rc.operationExecutor.UnmountVolume(
				mountedVolume.MountedVolume, rc.actualStateOfWorld, rc.kubeletPodsDir)
			if err != nil &&
				!nestedpendingoperations.IsAlreadyExists(err) &&
				!exponentialbackoff.IsExponentialBackoff(err) {
				// Ignore nestedpendingoperations.IsAlreadyExists and exponentialbackoff.IsExponentialBackoff errors, they are expected.
				// Log all other errors.
				klog.ErrorS(err, mountedVolume.GenerateErrorDetailed(fmt.Sprintf("operationExecutor.UnmountVolume failed (controllerAttachDetachEnabled %v)", rc.controllerAttachDetachEnabled), err).Error())
			}
			if err == nil {
				klog.InfoS(mountedVolume.GenerateMsgDetailed("operationExecutor.UnmountVolume started", ""))
			}
		}
	}
}
```

`operationExecutor.UnmountVolume()` 是对 unmount 进行了封装，其底层会查找支持的 VolumePlugin，并调用 **`VolumePlugin.TearDown()`** 接口。

#### 5.1.2 mountAttachVolumes

`mountAttachVolumes` 负责准备 volume。有两种可能的情况：
* 如果 volume 还未 attach，而 kubelet 允许执行 attach，那么**调用 `Attach` 操作**。
* 如果 volume 还未 mount，那么**执行 `MountDevice` 与 `SetUp` 操作**。
```go
func (rc *reconciler) mountAttachVolumes() {
	// 遍历 desiredStateOfWorld
	// Ensure volumes that should be attached/mounted are attached/mounted.
	for _, volumeToMount := range rc.desiredStateOfWorld.GetVolumesToMount() {
		// 查询 actualStateOfWorld 对应的 volume 状态
		volMounted, devicePath, err := rc.actualStateOfWorld.PodExistsInVolume(volumeToMount.PodName, volumeToMount.VolumeName)
		volumeToMount.DevicePath = devicePath

		// 根据 volMounted 与 err，执行操作

		if cache.IsVolumeNotAttachedError(err) {
			// 如果 err 表明 volume 还未被 attache

			if rc.controllerAttachDetachEnabled || !volumeToMount.PluginIsAttachable {
				// volume 未被 attach，而 kubelet 不允许执行 attach 操作，那么等待 Contrller 进行 attach
				err := rc.operationExecutor.VerifyControllerAttachedVolume(
					volumeToMount.VolumeToMount,
					rc.nodeName,
					rc.actualStateOfWorld)
				// log
			} else {
				// volume 未被 attach，而 kubelet 允许进行 attach 操作，那么由 kubelet 调用 attach 操作
				volumeToAttach := operationexecutor.VolumeToAttach{
					VolumeName: volumeToMount.VolumeName,
					VolumeSpec: volumeToMount.VolumeSpec,
					NodeName:   rc.nodeName,
				}
				err := rc.operationExecutor.AttachVolume(volumeToAttach, rc.actualStateOfWorld)
				// log
			}
		} else if !volMounted || cache.IsRemountRequiredError(err) {
			// 如果 volume 还未被 mount，或者 err 需要重新 mount

			// 调用 MountVolume，底层会进行 MountDevice + SetUp 操作
			remountingLogStr := ""
			isRemount := cache.IsRemountRequiredError(err)
			if isRemount {
				remountingLogStr = "Volume is already mounted to pod, but remount was requested."
			}
			err := rc.operationExecutor.MountVolume(
				rc.waitForAttachTimeout,
				volumeToMount.VolumeToMount,
				rc.actualStateOfWorld,
				isRemount)
			// log
		} else if cache.IsFSResizeRequiredError(err) && utilfeature.DefaultFeatureGate.Enabled(features.ExpandInUsePersistentVolumes) {
			// 如果 volume 为需要扩容

			// 调用 ExpandInUseVolume 进行扩容
			klog.V(4).InfoS(volumeToMount.GenerateMsgDetailed("Starting operationExecutor.ExpandInUseVolume", ""), "pod", klog.KObj(volumeToMount.Pod))
			err := rc.operationExecutor.ExpandInUseVolume(
				volumeToMount.VolumeToMount,
				rc.actualStateOfWorld)
			// log
		}
	}
}
```

#### 5.1.3 unmountDetachDevices

`unmountDetachDevices` 调用 **`Unmount Device` 操作解除全局目录的挂载，允许情况下还可能调用 Detach 操作移除 Volume**。
```go
func (rc *reconciler) unmountDetachDevices() {
	// 遍历 actualStateOfWorld 所有 unmounted volume
	for _, attachedVolume := range rc.actualStateOfWorld.GetUnmountedVolumes() {

		if !rc.desiredStateOfWorld.VolumeExists(attachedVolume.VolumeName) &&
			!rc.operationExecutor.IsOperationPending(attachedVolume.VolumeName, nestedpendingoperations.EmptyUniquePodName, nestedpendingoperations.EmptyNodeName) {
				// 如果 Volume 不存在于 desiredStateOfWorld

			if attachedVolume.DeviceMayBeMounted() {
				// Device 还是 mounted 状态，调用 Unmount Device

				err := rc.operationExecutor.UnmountDevice(
					attachedVolume.AttachedVolume, rc.actualStateOfWorld, rc.hostutil)
				
				// log
			} else {

				// Device 已经 unmounted 了，如果是有 Kubelet 负责 attach 的，进行 Detach 操作

				if rc.controllerAttachDetachEnabled || !attachedVolume.PluginIsAttachable {
					rc.actualStateOfWorld.MarkVolumeAsDetached(attachedVolume.VolumeName, attachedVolume.NodeName)
					klog.InfoS(attachedVolume.GenerateMsgDetailed("Volume detached", fmt.Sprintf("DevicePath %q", attachedVolume.DevicePath)))
				} else {
					// 调用 Detach
					klog.V(5).InfoS(attachedVolume.GenerateMsgDetailed("Starting operationExecutor.DetachVolume", ""))
					err := rc.operationExecutor.DetachVolume(
						attachedVolume.AttachedVolume, false /* verifySafeToDetach */, rc.actualStateOfWorld)
					// log
				}
			}
		}
	}
}
```

## 参考

* [**Blog：详解 Kubernetes Volume 的实现原理**](https://draveness.me/kubernetes-volume/)
* [**Blog：kubelet volume manager 源码分析**](http://bingerambo.com/posts/2021/02/kubelet-volume-manager%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90/)
