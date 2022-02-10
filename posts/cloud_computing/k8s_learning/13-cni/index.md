# K8s 学习 - 11 - CNI 概念与实现


## 1 CNI 基本概念

CNI 全称 Container Network Interface，是 Kubernetes 中的网络接口。Kubectl 会依据 CNI 标准的 API 来调用不同的网络插件接口。

CNI 插件实际上是一个主机上的可执行文件，Kubernetes 会在操作 Pod 网络时根据 Network Configuration 去调用各个插件的可执行文件，同时通过 Stdin 与 Env 传入所必须的信息。执行结束后，通过 Stdout 读取执行的结果。

其整体的架构如下：
{{< find_img "img1.png" >}}

## 2 CNI Spec

[**CNI Spec**](https://github.com/containernetworking/cni/blob/master/SPEC.md) 定义了 CNI 插件的可执行文件需要实现哪些操作。其主要涉及到几个部分：

* Network Configuration - CNI 网络配置，包含需要使用的插件。
* Execution Protocol - 表明一个 CNI 插件需要实现的功能，包括 ADD、CHECK、DEL、VERSION 操作，以及如何返回结果
* CNI 的调用 - CNI 插件的调用方式，以及允许传递参数。

### 2.1 Network Configuration

Network Configuration 代表了 CNI 的 Network 配置，每次调用 CNI 插件执行操作时，都会传入该信息。

以 calico 的配置文件为例，其中包含 "calico" "bandwidth" "portmap" 三个插件：
```json
{
  "name": "k8s-pod-network",
  "cniVersion": "0.3.1",
  "plugins": [
    {
      "type": "calico",
      "datastore_type": "kubernetes",
      "mtu": 0,
      "nodename_file_optional": false,
      "log_level": "Info",
      "log_file_path": "/var/log/calico/cni/cni.log",
      "ipam": { "type": "calico-ipam", "assign_ipv4" : "true", "assign_ipv6" : "false"},
      "container_settings": {
          "allow_ip_forwarding": false
      },
      "policy": {
          "type": "k8s"
      },
      "kubernetes": {
          "k8s_api_root":"https://10.0.0.1:443",
          "kubeconfig": "/etc/cni/net.d/calico-kubeconfig"
      }
    },
    {
      "type": "bandwidth",
      "capabilities": {"bandwidth": true}
    },
    {"type": "portmap", "snat": true, "capabilities": {"portMappings": true}}
  ]
}
```
* cniVersion - CNI 的版本
* name - 必须唯一的网络名称
* disableCheck - 是否需要周期性调用 CHECK 操作检查网络
* plugins - ADD/DEL/CHECK 操作时需要链式调用的插件

`plugins` 字段包含一些标准的字段，也可以包含一些动态信息（用于拓传给插件）。
* 必须的字段
  * type - 插件可执行文件的名字，会根据 type 去调用对应的可执行文件
* 可选的字段
  * capabilities（可选） - 插件负责的功能
* well-known 的字段
  * ipMasq - 插件支持 IP masquerade
  * ipam - IPAM 相关信息
  * dns - 需要插件设置的 DNS 配置
* 其他动态的字段

### 2.2 Execution Protocol

Execution Protocol 表明一个插件需要实现的功能，插件根据功能分为两大类：

* Interface 插件 - 主要的插件，用于构建容器网络
* Chained 插件 - 辅助的通用插件，用于调整网络。例如 bandwidth 插件用于设置限速。

#### 2.2.1 参数传递

当调用 CNI 插件可执行文件时，会通过两种方式传递信息：
* Stdin - CNI 整体信息，即 JSON 格式的 Network Configuration
* Env - 容器相关信息，设置特定命名的 Env

不同的操作会设置不同的 Env，整体上会使用以下的环境变量：
* CNI_COMMAND - 表示插件需要执行的操作：ADD、DEL、CHECK 或 VERSION。
* CNI_CONTAINERID - 操作的容器的 ID
* CNI_NETNS - 容器的 network namespace，通常为 namespace 的文件（例如 /run/netns/\[ns]）
* CNI_IFNAME - 在容器中创建的 interface 的名字
* CNI_ARGS - 额外传递的参数，通过键值对传递，例如 "FOO=BAR;ABC=123"
* CNI_PATH - 包含所有 CNI 插件可执行文件的路径

#### 2.2.2 执行结果

所有的操作会返回三类结果：
* Success - 操作执行成功
* Error - 操作执行失败
* _Version - 针对于 VERSION 操作的特殊的返回

关于各个结果的具体字段，见 [**Spec: Result Types**](https://github.com/containernetworking/cni/blob/master/SPEC.md#section-5-result-types)。

#### 2.2.3 插件操作

CNI 定义了四种操作：
* ADD - 构建容器的网络，或者修改容器的网络配置
* DEL - 销毁容器的网络，或者取消修改容器网络配置
* CHECK - 检查容器网络是否符合期望
* VERSION - 输出插件版本

具体详细的操作输入与输出见 [**Spec: CNI Operations**](https://github.com/containernetworking/cni/blob/master/SPEC.md#cni-operations)

## 3 CNI API

[**CNI 仓库**](https://github.com/containernetworking/cni/blob/master/libcni/api.go) 中定义了 CNI 的标准 API。代码中管理 Pod 网络时通过调用下面的 API 来进行操作。

```go
type CNI interface {
	AddNetworkList(ctx context.Context, net *NetworkConfigList, rt *RuntimeConf) (types.Result, error)
	CheckNetworkList(ctx context.Context, net *NetworkConfigList, rt *RuntimeConf) error
	DelNetworkList(ctx context.Context, net *NetworkConfigList, rt *RuntimeConf) error
	GetNetworkListCachedResult(net *NetworkConfigList, rt *RuntimeConf) (types.Result, error)
	GetNetworkListCachedConfig(net *NetworkConfigList, rt *RuntimeConf) ([]byte, *RuntimeConf, error)

	AddNetwork(ctx context.Context, net *NetworkConfig, rt *RuntimeConf) (types.Result, error)
	CheckNetwork(ctx context.Context, net *NetworkConfig, rt *RuntimeConf) error
	DelNetwork(ctx context.Context, net *NetworkConfig, rt *RuntimeConf) error
	GetNetworkCachedResult(net *NetworkConfig, rt *RuntimeConf) (types.Result, error)
	GetNetworkCachedConfig(net *NetworkConfig, rt *RuntimeConf) ([]byte, *RuntimeConf, error)

	ValidateNetworkList(ctx context.Context, net *NetworkConfigList) ([]string, error)
	ValidateNetwork(ctx context.Context, net *NetworkConfig) ([]string, error)
}
```

{{< admonition note Note>}}
`NetworkConfigList` 结构就是 Network Configuration，`NetworkConfig` 是旧版本仅仅支持一个插件的结构。
{{< /admonition >}}

CNI 官方提供了 `CNIConfig` 对象来实现 CNI API，其接口实现会去调用对应的 CNI 插件的可执行文件。只要提供了可执行文件的目录，就可以创建 CNIConfig 对象并使用。

```go
type CNIConfig struct {
	Path     []string
	exec     invoke.Exec
	cacheDir string
}
```
* Path - 表明 CNI 插件可执行文件的目录，默认为 `/etc/cni/net.d`
* exec - 命令行调用的封装，类似于标准库 `exec`
* cacheDir - 持久化缓存调用结果的目录

### 3.1 构建 Pod 网络

`AddNetworkList()` 与 `AddNetwork()` 用于构建 Pod 的网络，以 `AddNetworkList()` 为例看一下实现：

```go
// AddNetworkList executes a sequence of plugins with the ADD command
func (c *CNIConfig) AddNetworkList(ctx context.Context, list *NetworkConfigList, rt *RuntimeConf) (types.Result, error) {
	var err error
	var result types.Result

	// 链式执行 CNI 插件
	for _, net := range list.Plugins {
		result, err = c.addNetwork(ctx, list.Name, list.CNIVersion, net, result, rt)
	}

	// 结构加入到 cache
	c.cacheAdd(result, list.Bytes, list.Name, rt)

	return result, nil
}

func (c *CNIConfig) addNetwork(ctx context.Context, name, cniVersion string, net *NetworkConfig, prevResult types.Result, rt *RuntimeConf) (types.Result, error) {
	// 插件可执行文件
	pluginPath, err := c.exec.FindInPath(net.Network.Type, c.Path)

	// 参数检查
	err := utils.ValidateContainerID(rt.ContainerID)
	err := utils.ValidateNetworkName(name)
	err := utils.ValidateInterfaceName(rt.IfName)

	// 构建传给可执行文件的配置
	newConf, err := buildOneConfig(name, cniVersion, net, prevResult, rt)

	// 调用可执行文件 plugin 执行 ADD
	return invoke.ExecPluginWithResult(ctx, pluginPath, newConf.Bytes, c.args("ADD", rt), c.exec)
}
```

### 3.2 销毁 Pod 网络

`DelNetworkList()` 与 `DelNetwork()` 用于销毁 Pod 的网络，以 `DelNetworkList()` 为例看一下实现：

```go
// DelNetworkList executes a sequence of plugins with the DEL command
func (c *CNIConfig) DelNetworkList(ctx context.Context, list *NetworkConfigList, rt *RuntimeConf) error {
	var cachedResult types.Result

	// 尝试读取 cache 结果（之前的失败的结果）
	cachedResult, err = c.getCachedResult(list.Name, list.CNIVersion, rt)

	// 链式调用 plugin 执行
	for i := len(list.Plugins) - 1; i >= 0; i-- {
		net := list.Plugins[i]
		if err := c.delNetwork(ctx, list.Name, list.CNIVersion, net, cachedResult, rt); err != nil {
			return err
		}
	}

	// 所有 plugin 执行成功，删除 cache
	_ = c.cacheDel(list.Name, rt)
	return nil
}

func (c *CNIConfig) delNetwork(ctx context.Context, name, cniVersion string, net *NetworkConfig, prevResult types.Result, rt *RuntimeConf) error {
	// 插件可执行文件
	pluginPath, err := c.exec.FindInPath(net.Network.Type, c.Path)

	// 构建传给可执行文件的配置
	newConf, err := buildOneConfig(name, cniVersion, net, prevResult, rt)

	// 调用可执行文件 plugin 执行 DEL
	return invoke.ExecPluginWithoutResult(ctx, pluginPath, newConf.Bytes, c.args("DEL", rt), c.exec)
}
```

### 3.3 检查 Pod 网络

`CheckNetworkList()` 与 `CheckNetwork()` 用于检查 Pod 的网络是否正常，以 `CheckNetworkList()` 为例看一下实现：

```go
func (c *CNIConfig) CheckNetworkList(ctx context.Context, list *NetworkConfigList, rt *RuntimeConf) error {
	if list.DisableCheck {
		return nil
	}

	// 读取缓存结果（上一次检查结果）
	cachedResult, err := c.getCachedResult(list.Name, list.CNIVersion, rt)

	// 链式调用 plugin 执行
	for _, net := range list.Plugins {
		if err := c.checkNetwork(ctx, list.Name, list.CNIVersion, net, cachedResult, rt); err != nil {
			return err
		}
	}
	return nil
}

func (c *CNIConfig) checkNetwork(ctx context.Context, name, cniVersion string, net *NetworkConfig, prevResult types.Result, rt *RuntimeConf) error {
	// 插件可执行文件
	pluginPath, err := c.exec.FindInPath(net.Network.Type, c.Path)

	// 构建传给可执行文件的配置
	newConf, err := buildOneConfig(name, cniVersion, net, prevResult, rt)

	// 调用可执行文件 plugin 执行 CHECK
	return invoke.ExecPluginWithoutResult(ctx, pluginPath, newConf.Bytes, c.args("CHECK", rt), c.exec)
}
```

## 4 kubelet 如何调用 CNI

### 4.1 Node 网络状态维护

Node 资源使用 [**Condition**](https://kubernetes.io/zh/docs/concepts/architecture/nodes/#condition) 来描述 Node 当前的状态，其中包含 "NetworkUnavailable" 用于描述 Node 网络是否正常。与其他 Condition 不同的是，"NetworkUnavailable" 是由 CNI 插件来负责维护的。

CNI 插件会根据自己网络构建情况，来更新（Patch 机制）Node 对象的 "NetworkUnavailable" Condition，从而让 Kubernetes 知晓 Node 的节点网络是否正常。

{{< admonition note Note>}}
在新添加 Node 时，需要部署 CNI 插件才会让 Condition 变为 Ready，从而能够接受 Pod。
{{< /admonition >}}

### 4.2 调用 CNI

#### 4.2.1 何时调用 CNI 插件

kubelet 并不会直接调用 CNI 的接口，而是调用 CRI 的 `RunPodSandbox()` 接口，来启动 Sandbox（在容器实现中也就是 pause 容器）。在 CRI 的 `RunPodSandbox()` 实现中，在 Sandbox 启动后需要构建 Pod 的网络。

以 dockershim 为例，其 `RunPodSandbox()` 接口实现如下：
```go
func (ds *dockerService) RunPodSandbox(ctx context.Context,
	r *runtimeapi.RunPodSandboxRequest) (*runtimeapi.RunPodSandboxResponse, error) {
	config := r.GetConfig()

	// 1. 拉取 sandbox 的image, 默认为 "k8s.gcr.io/pause:3.1"
	// ...
	if err := ensureSandboxImageExists(ds.client, image); err != nil {
		return nil, err
	}

	// 2. 创建 sandbox container, 即 pause 容器
	// ...
	createResp, err := ds.client.CreateContainer(*createConfig)
	if err != nil {
		createResp, err = recoverFromCreationConflictIfNeeded(ds.client, *createConfig, err)
	}

	if err != nil || createResp == nil {
		return nil, fmt.Errorf("failed to create a sandbox for pod %q: %v", config.Metadata.Name, err)
	}
	resp := &runtimeapi.RunPodSandboxResponse{PodSandboxId: createResp.ID}

	// 3. 创建 sandbox checkpoint
	if err = ds.checkpointManager.CreateCheckpoint(createResp.ID, constructPodSandboxCheckpoint(config)); err != nil {
		return nil, err
	}

	// 4. 启动 Sandbox Container
	err = ds.client.StartContainer(createResp.ID)
	if err != nil {
		return nil, fmt.Errorf("failed to start sandbox container for pod %q: %v", config.Metadata.Name, err)
	}

	// 覆盖 container的 resolve.conf 文件
	if dnsConfig := config.GetDnsConfig(); dnsConfig != nil {
		containerInfo, err := ds.client.InspectContainer(createResp.ID)
		if err != nil {
			return nil, fmt.Errorf("failed to inspect sandbox container for pod %q: %v", config.Metadata.Name, err)
		}

		if err := rewriteResolvFile(containerInfo.ResolvConfPath, dnsConfig.Servers, dnsConfig.Searches, dnsConfig.Options); err != nil {
			return nil, fmt.Errorf("rewrite resolv.conf failed for pod %q: %v", config.Metadata.Name, err)
		}
	}

	// 后续 network 相关操作, 如果是 host network, 可以直接退出了
	if config.GetLinux().GetSecurityContext().GetNamespaceOptions().GetNetwork() == runtimeapi.NamespaceMode_NODE {
		return resp, nil
	}

	// 5. 通过 CNI 插件配置 sandbox 的 network
	cID := kubecontainer.BuildContainerID(runtimeName, createResp.ID)
	networkOptions := make(map[string]string)
	if dnsConfig := config.GetDnsConfig(); dnsConfig != nil {
		// Build DNS options.
		dnsOption, err := json.Marshal(dnsConfig)
		if err != nil {
			return nil, fmt.Errorf("failed to marshal dns config for pod %q: %v", config.Metadata.Name, err)
		}
		networkOptions["dns"] = string(dnsOption)
	}
	// 这里就是使用 network plugin 进行网络配置了
	err = ds.network.SetUpPod(config.GetMetadata().Namespace, config.GetMetadata().Name, cID, config.Annotations, networkOptions)
	if err != nil {
		// 网络构建失败，销毁网络并停止容器
		// Ensure network resources are cleaned up even if the plugin
		// succeeded but an error happened between that success and here.
		err = ds.network.TearDownPod(config.GetMetadata().Namespace, config.GetMetadata().Name, cID)

		err = ds.client.StopContainer(createResp.ID, defaultSandboxGracePeriod)
	}

	return resp, nil
}
```

该 network plugin 是 dockershim 对调用 CNI 插件的封装，其 `SetUpPod()` 接口底层才会真正调用 CNI 插件。
```go
// SetUpPod 构建 Pod 的网络
func (plugin *cniNetworkPlugin) SetUpPod(namespace string, name string, id kubecontainer.ContainerID,
	annotations, options map[string]string) error {
	// 检查是否已经初始化
	if err := plugin.checkInitialized(); err != nil {
		return err
	}

	// 通过 <host>.GetNetNS() 得到对应容器的 namespace路径
	netnsPath, err := plugin.host.GetNetNS(id.ID)

	// Windows doesn't have loNetwork. It comes only with Linux
	if plugin.loNetwork != nil {
		if _, err = plugin.addToNetwork(cniTimeoutCtx, plugin.loNetwork,
			name, namespace, id, netnsPath, annotations, options); err != nil {
			return err
		}
	}

	_, err = plugin.addToNetwork(cniTimeoutCtx, plugin.getDefaultNetwork(),
		name, namespace, id, netnsPath, annotations, options)
}

func (plugin *cniNetworkPlugin) addToNetwork(ctx context.Context, network *cniNetwork, podName string, podNamespace string, podSandboxID kubecontainer.ContainerID, podNetnsPath string, annotations, options map[string]string) (cnitypes.Result, error) {
	// 构建 Pod 对应的配置 libcni.RuntimeConf
	rt, err := plugin.buildCNIRuntimeConf(podName, podNamespace, podSandboxID, podNetnsPath, annotations, options)

	// 调用 CNI.AddNetworkList() 接口
	netConf, cniNet := network.NetworkConfig, network.CNIConfig
	res, err := cniNet.AddNetworkList(ctx, netConf, rt)
	return res, nil
}
```

这里的 `network.CNIConfig` 就是之前所述的 `CNIConfig` 对象。

#### 4.2.2 如何管理 CNI 插件

CNI 本质上实现的是二进制文件，而代码中的 CNI 接口就是 kubelet 中实现的。kubelet 会周期性调用 `syncNetworkConfig()` 接口，用于读取 CNI 插件。

```go
func (plugin *cniNetworkPlugin) syncNetworkConfig() {
	network, err := getDefaultCNINetwork(plugin.confDir, plugin.binDirs)
	if err != nil {
		klog.InfoS("Unable to update cni config", "err", err)
		return
	}
	plugin.setDefaultNetwork(network)
}

func getDefaultCNINetwork(confDir string, binDirs []string) (*cniNetwork, error) {
	// 读取网络配置文件，以 .conf/.conflist/.json 结尾的文件
	files, err := libcni.ConfFiles(confDir, []string{".conf", ".conflist", ".json"})

	cniConfig := &libcni.CNIConfig{Path: binDirs}

	// 遍历所有文件，直到成功解析一个配置文件
	sort.Strings(files)
	for _, confFile := range files {
		var confList *libcni.NetworkConfigList
		// ... 解析为 NetworkConfigList

		// 调用 CNI.ValidateNetworkList 进行一次检查
		caps, err := cniConfig.ValidateNetworkList(context.TODO(), confList)

		// 读取成功返回
		return &cniNetwork{
			name:          confList.Name,
			NetworkConfig: confList,
			CNIConfig:     cniConfig,
			Capabilities:  caps,
		}, nil
	}
	return nil, fmt.Errorf("no valid networks found in %s", confDir)
}
```

可以看到，kubelet 会读取 CNI 的 conf 目录（默认 `/etc/cni/net.d`）来读取 CNI 的配置，通过 CNI 的 bin 目录（默认 /opt/cni/bin）去调用可执行文件。

通过创建 `libcni.CNIConfig` 实现了构建 CNI 的接口，而 `CNIConfig` 就是前面所述的官方提供的实现。

```go
cniConfig := &libcni.CNIConfig{Path: binDirs}
```


## 参考

* [**CNI 官方仓库**](https://github.com/containernetworking/cni)
* [**CNI Spec**](https://github.com/containernetworking/cni/blob/master/SPEC.md)

