# K8s 编程 - 8 - Custom Scheduler


kube-scheduler 负责集群资源的调度，原生的 Kubernetes Scheduler 可以满足大多数情况的需求。不过针对于不同业务的特殊的调度需求，Kubernetes 提供了多种扩展调度器的方式。

## 1 Scheduler Framework 扩展点

Scheduler Framework 是源码中的调度器的框架，其实现了调度过程中一系列的扩展点，支持用户以 Plugin 的方式进行扩展。

整个的调度流程中的扩展点如下图：

{{< image src="img1.png" height=300 >}}

所有的扩展点都是由 Plugin 实现的，允许配置开启或关闭，也可以添加自定义的 Plugin。这在后续的 **`KubeSchedulerConfiguration`** 中深入说明。

### 1.1 调度阶段的扩展点

* **QueueSort**

  ```go
  type QueueSortPlugin interface {
    Plugin
    Less(*QueuedPodInfo, *QueuedPodInfo) bool
  }
  ```

  该插件用于对 scheduling queue 中的 Pod 进行排序，一次只能存在一个 QueueSort Plugin。

  一个 QueueSort Plugin 本质上就是提供了一个 Less(Pod1, Pod2) 函数。

* **PreFilter**
  
  ```go
  type PreFilterPlugin interface {
    Plugin
    PreFilter(ctx context.Context, state *CycleState, p *v1.Pod) *Status
    PreFilterExtensions() PreFilterExtensions
  }
  ```

  该插件用于对 Pod 的信息进行预处理，或者检查一些集群或 Pod 必须满足的前提条件。
  
  如果 PreFilter Plugin 返回一个 error，那么调度流程直接终止。

* **Filter**
  
  ```go
  type FilterPlugin interface {
    Plugin
    Filter(ctx context.Context, state *CycleState, pod *v1.Pod, nodeInfo *NodeInfo) *Status
  }
  ```

  该插件用于过滤不能运行 Pod 的 Node。
  
  对于每一个 Node，调度器将按照顺序执行 Filter Plugin。如果任意一个 Filter Plugin 将 Node 标记为不可行，那么剩下的 Filter Plugin 就不会再运行。
  
  所有 Node 会并行进行 Filter。

* **PostFilter**
  
  ```go
  type PostFilterPlugin interface {
    Plugin
    PostFilter(ctx context.Context, state *CycleState, pod *v1.Pod, filteredNodeStatusMap NodeToStatusMap) (*PostFilterResult, *Status)
  }
  ```

  这些插件在 Filter 阶段之后被调用，只有在没有为 Pod 找到可运行的 Node 时才会运行。所有插件按照顺序运行。
  
  如果任意一个 PostFilter Plugin 将 Node 标记为 Schedulable，剩下的 PostFilter Plugin 就不会再运行。
  
  一个典型的 PostFilter Plugin 是抢占，它通过抢占其他 Pod 来调度 Pod。

* **PreScore**
  
  ```go
  type PreScorePlugin interface {
    Plugin
    PreScore(ctx context.Context, state *CycleState, pod *v1.Pod, nodes []*v1.Node) *Status
  }
  ```

  这些插件用于执行 "预打分" 工作，为 Score Plugin 生成一个可共享的状态。
  
  如果 PreScore 插件返回 error，调度流程终止。

* **Score**
  
  ```go
  type ScorePlugin interface {
    Plugin
    Score(ctx context.Context, state *CycleState, p *v1.Pod, nodeName string) (int64, *Status)
    ScoreExtensions() ScoreExtensions
  }
  ```

  这些插件用于为所有可选 Node 打分，调度器会为每个 Node 调用每个 Score Plugin。
  
  打分有一个明确定义的整数范围，代表最低和最高分数。
  
  在 NormalizeScore 阶段后，Scheduler 将根据配置的插件权重，合并所有插件的打分。

* **NormalizeScore**

  ```go
  type ScoreExtensions interface {
    NormalizeScore(ctx context.Context, state *CycleState, p *v1.Pod, scores NodeScoreList) *Status
  }
  ```

  这些插件用于在最终排序 Node 前修改每个 Node 的评分结果。
  
  当 NormalizeScore Plugin 被调用时，将获得同一个 Plugin 中打分结果。在每个调度周期后，每个插件会被调用一次。
  
  例如，支持 Plugin `BlinkingLightScorer` 根据每个 Node 有多少个 blinking lights 进行排名。

  ```go
  func ScoreNode(_ *v1.pod, n *v1.Node) (int, error) {
      return getBlinkingLightCount(n)
  }
  ```

  但是，与 NodeScoreMax 相比，blinking lights 数量也可能很少。为解决这个问题，BlinkingLightScorer 也注册了这个扩展点。

  ```go
  func NormalizeScores(scores map[string]int) {
      highest := 0
      for _, score := range scores {
          highest = max(highest, score)
      }
      for node, score := range scores {
          scores[node] = score*NodeScoreMax/highest
      }
  }
  ```

  如果任何 NormalizeScore Plugin 返回 error，调度将终止。

* **Reserve**

  ```go
  type ReservePlugin interface {
    Plugin
    Reserve(ctx context.Context, state *CycleState, p *v1.Pod, nodeName string) *Status
    Unreserve(ctx context.Context, state *CycleState, p *v1.Pod, nodeName string)
  }
  ```

  Reserve Plugin 有两个方法：`Reserve` 和 `Unreserve`，会带有 Reserve 和 Unreserve 调度阶段的信息。有状态的插件可以使用该扩展点来获得 Node 上为 Pod 预留的资源。

  该事件发生在调度器将 Pod 绑定到 Node 之前，目的是避免调度器在等 Pod 与 Node 绑定过程中，有新的 Pod 调度到该 Node 上，发生实际使用资源超过可用资源的情况。（因为绑定 Pod 是异步的）

  这是调度过程的最后一个步骤，Pod 进入 reserved 状态后，要么绑定失败触发 Unreserve，要么由 Postbind 扩展结束绑定过程。

* **Permit**
  
  ```go
  type PermitPlugin interface {
    Plugin
    Permit(ctx context.Context, state *CycleState, p *v1.Pod, nodeName string) (*Status, time.Duration)
  }
  ```

  Permit Plugin 在调度周期的最后调用，为了组织或延迟绑定后候选的 Node。

  Permit Plugin 做以下的事情：
  
  1. approve
   
     一旦所有 Permit Plugin approve 一个 Pod，那么就会进行绑定操作。
  
  2. deny
   
     如果任一 Permit Plugin deny 一个 Pod，那么将其重新放入调度队列。这也会触发 Unreserve 阶段。

  3. wait（with a timeout）
   
     如果一个 Permit Plugin 返回 wait，那么该 Pod 会在 waiting Pod list 中等待，直到该 Pod 被 approve。

     如果 wait timeout 触发，那么变为 deny Pod。

### 1.2 绑定阶段的扩展点

* **PreBind**
  
  ```go
  type PreBindPlugin interface {
    Plugin
    PreBind(ctx context.Context, state *CycleState, p *v1.Pod, nodeName string) *Status
  }
  ```

  这些插件用于在 Pod 被绑定前执行一些工作。
  
  例如，PreBind Plugin 可以创建 network volume，并将其绑定到目标 Node。
  
  如果任一 PreBind Plugin 返回 error，Pod 会被拒绝，然后重新放入调度队列。

* **Bind**
  
  ```go
  type BindPlugin interface {
    Plugin
    Bind(ctx context.Context, state *CycleState, p *v1.Pod, nodeName string) *Status
  }
  ```

  这些插件用于将 Pod 绑定到 Node。

  每个 Bind Plugin 会按照配置顺序调用，每个 Bind Plugin 可以选择是否处理该 Pod。如果某个 Bind Plugin 决定处理该 Pod，后续的 Bind Plugin 会跳过。

* **PostBind**

  ```go
  type PostBindPlugin interface {
    Plugin
    PostBind(ctx context.Context, state *CycleState, p *v1.Pod, nodeName string)
  }
  ```

  PostBind Plugin 在绑定成功后被调用，是绑定周期的最后一个阶段。常常用于清理相关资源。

## 2 定制 Scheduler

Scheduler 内置了一些 Plugin，可以通过配置文件的方式选择开启或关闭哪些 Plugin。

### 2.1 ~~Scheduling Policy~~

**`Scheduling Policy`** 是一种老版本配置调度器的方式，可以定制 Predicates 与 Priorities 使用的插件，也可以通过 Extender 对阶段进行扩展。

{{< admonition warning "Scheduling Policy 是旧版本的">}}
1.22 后，kube-scheduler 已经不支持 `--scheduler-name` 参数了，因此已经无法使用 Scheduler Policy 来得到一个新的调度器了。

新版多调度器只能通过 [**Multiple Profiles**](#32-multiple-profiles) 方式实现，而其与 Scheduler Policy 是不兼容的。所以如果你要考虑运行一个新的调度器，应该直接使用 Scheduling Profiles 方式。

当然，Schedule Policy 还是能够实现定制 default scheduler 的作用，不过 Scheduling Profiles 能实现的更多。
{{< /admonition >}}

Scheduling Policy 可以为 yaml 或者 json 格式，一个示例如下：

```json
{
  "kind" : "Policy",
  "apiVersion" : "v1",
  "predicates": [
    {"name": "NoVolumeZoneConflict"},
    {"name": "MaxEBSVolumeCount"},
    {"name": "MaxAzureDiskVolumeCount"},
    {"name": "NoDiskConflict"},
    {"name": "GeneralPredicates"},
    {"name": "PodToleratesNodeTaints"},
    {"name": "CheckVolumeBinding"},
    {"name": "MaxGCEPDVolumeCount"},
    {"name": "MatchInterPodAffinity"},
  ],
  "priorities": [
    {"name": "SelectorSpreadPriority", "weight": 1},
    {"name": "InterPodAffinityPriority", "weight": 1},
    {"name": "LeastRequestedPriority", "weight": 1},
    {"name": "BalancedResourceAllocation", "weight": 1},
    {"name": "NodePreferAvoidPodsPriority", "weight": 1},
    {"name": "NodeAffinityPriority", "weight": 1},
    {"name": "TaintTolerationPriority", "weight": 1}
  ],
  "extenders": [
    {
      "urlPrefix": "http://127.0.0.1:10262/scheduler",
      "filterVerb": "filter",
      "preemptVerb": "preempt",
      "weight": 1,
      "httpTimeout": 30000000000,
      "enableHttps": false
    }
  ]
}
```
* `predicates` 定制 Predicate 阶段使用的插件；
* `priorities` 定制 Priority 阶段使用的插件；
* `extenders` 定制扩展的 Webhook，调度器会在对应的阶段调用对应的 URL；

运行 kube-scheduler 时，可以通过两个参数配置 Policy 文件：

* `--policy-config-file $filename` 指定 Policy 文件路径；
* `--policy-configmap $configmap --policy-configmap-namespace $ns` 指定存储 Policy 的 ConfigMap；

加上 `--scheduler-name` 参数指定 scheduler name，很简单的定义出一个新的调度器。

```bash
$ kube-scheduler
    --port=10261
    --leader-elect=true
    --lock-object-namespace=pingcap
    --lock-object-name=my-scheduler
    --scheduler-name=my-scheduler
    --policy-configmap=my-scheduler-policy
    --policy-configmap-namespace=pingcap
```

使用上述命令，我们运行了一个名为 `my-scheduler` 的调度器，通过 Pod 定义的 `spec.schedulerName` 可以指定使用该调度器。

### 2.2 Scheduling Profiles

无论哪种方式扩展调度器，都是围绕着通过 **`Scheduling Profiles`** 对 kube-scheduler 进行配置实现的。

运行 kube-scheduler 时，通过 `kube-scheduler --config $filename` 指定 Scheduling Profiles。其文件内容是 v1beta1 或者 v1beta2 版本的 KubeSchedulerConfiguration 结构。

最简单的配置如下：

```yaml
apiVersion: kubescheduler.config.k8s.io/v1beta2
kind: KubeSchedulerConfiguration
clientConnection:
  kubeconfig: /etc/srv/kubernetes/kube-scheduler/kubeconfig
```

#### 2.2.1 配置 Plugin

在 KubeSchedulerConfiguration 定义中的 **`profiles`** 字段对 kube-scheduler 的插件进行配置。

支持配置 [**Scheduler Framework 扩展点**](#1-scheduler-framework-扩展点) 中的扩展点。对于每个扩展点，你可以关闭某些默认开启的 Plugin，或者打开你自己定义的 Plugin。

例如下面示例：

```yaml
apiVersion: kubescheduler.config.k8s.io/v1beta2
kind: KubeSchedulerConfiguration
profiles:
  - plugins:
      score:  # score plugin
        disabled:
        - name: NodeResourcesLeastAllocated # disable NodeResourcesLeastAllocated
        enabled:
        - name: MyCustomPluginA # enable my plugin
          weight: 2
        - name: MyCustomPluginB
          weight: 1
    pluginConfig:
    - name: NodeResourcesAllocatable
      args:
        mode: Least
        resources:
        - name: cpu
          weight: 1000000
        - name: memory
          weight: 1
```

* `profiles.plugins`- 配置各个扩展点的插件
  * disabled - 关闭插件
  * enabled - 打开插件
* `profiles.pluginConfig` - 给特定插件的配置

{{< admonition note "插件是编译到程序中的">}}
在 `profiles.plugins` 字段中配置的插件都是预编译到 scheduler 代码中。

也就是说，你需要先通过 scheduler framework 编写自定义 Plugin 代码，编译成 scheduler 镜像，然后才能打开你定义的插件。
{{< /admonition >}}

#### 2.2.2 Multiple Profiles

可以通过 `profiles` 字段配置多个 Profile，每个 Profile 就类似与一个 Scheduler 的 “分身”，可以进行不同扩展点 Plugin 的配置，并且有着独立的 `schedulerName`。

如下配置中，将运行两个调度器：

* default-scheduler - 运行所有默认 Plugin
* no-scoring-scheduler - 关闭了所有的 Score Plugin。

```yaml
apiVersion: kubescheduler.config.k8s.io/v1beta2
kind: KubeSchedulerConfiguration
profiles:
  - schedulerName: default-scheduler
  - schedulerName: no-scoring-scheduler
    plugins:
      preScore:
        disabled:
        - name: '*'
      score:
        disabled:
        - name: '*'
```

多个调度器运行时，通过 Pod 定义中的 `.spec.schedulerName` 字段来指定期望的调度器。

如果配置中只有一个 Profile，那么默认名为 "default-scheduler"，也就是 Kubernetes 中默认的调度器名字。

### 2.3 Scheduler Extender

Policy 与 Profiles 都支持 Scheduler Extender 方式定义，与定制 Plugin 方式不同，Extender 允许 Scheduler 在调度时候调用 Webhook。

下面以 Profiles 为例，看下 Extender 是如何配置的：

```yaml
apiVersion: kubescheduler.config.k8s.io/v1beta1
kind: KubeSchedulerConfiguration
leaderElection:
  leaderElect: true
  resourceNamespace: namespace
  resourceName: my-scheduler
profiles:
  - schedulerName: my-scheduler
extenders:
  - urlPrefix: http://127.0.0.1:10262/scheduler
    filterVerb: filter
    prioritizeVerb: prioritize
    preemptVerb: preempt
    weight: 1
    enableHTTPS: false
    tlsConfig: // ...
    httpTimeout: 30s
    // ...
```

* `extenders`
  
  * urlPrefix - Webhook 的基本的 URL 地址
  * filterVerb - 设置后表示 Webhook 支持 filter 功能
  * prioritizeVerb - 设置后表示 Webhook 支持 prioritize 功能
  * preemptVerb - 设置后表示 Webhook 支持 preempt 功能
  * bindVerb - 设置后表示 Webhook 支持 bind 功能
  * weight - Webhook 对 Node 打分后的乘数
  * enableHTTPS / tlsConfig - TLS 配置

Extender 支持多个位置的 Webhook：

* filter
  
  在 Filter 阶段影响 Pod 筛选 Node，在所有 Plugin 运行后调用。

* prioritize
  
  在 Prioritize 阶段影响 Node 的打分，在所有 Plugin 运行后被调用。

* preempt
  
  在 Preempt 阶段影响如何选择 Pod 抢占，在所有 Plugin 运行后调用。

* bind (optional)
  
  在 Bind 阶段影响 Node 的绑定，在所有 Plugin 运行前调用。

#### 2.3.1 filter

filter webhook 在 Filter 阶段所有 Plugin 运行后调用。

* Request 提供了 Pod 以及筛选出的 Node 列表
  
  ```go
  // ExtenderArgs represents the arguments needed by the extender to filter/prioritize
  // nodes for a pod.
  type ExtenderArgs struct {
    // Pod being scheduled
    Pod *v1.Pod
    // List of candidate nodes where the pod can be scheduled; to be populated
    // only if Extender.NodeCacheCapable == false
    Nodes *v1.NodeList
    // List of candidate node names where the pod can be scheduled; to be
    // populated only if Extender.NodeCacheCapable == true
    NodeNames *[]string
  }
  ```

* Response 返回修改后的 Node 列表，以及给出过滤的原因
  
  ```go
  // ExtenderFilterResult represents the results of a filter call to an extender
  type ExtenderFilterResult struct {
    // Filtered set of nodes where the pod can be scheduled; to be populated
    // only if Extender.NodeCacheCapable == false
    Nodes *v1.NodeList
    // Filtered set of nodes where the pod can be scheduled; to be populated
    // only if Extender.NodeCacheCapable == true
    NodeNames *[]string
    // Filtered out nodes where the pod can't be scheduled and the failure messages
    FailedNodes FailedNodesMap
    // Filtered out nodes where the pod can't be scheduled and preemption would
    // not change anything. The value is the failure message same as FailedNodes.
    // Nodes specified here takes precedence over FailedNodes.
    FailedAndUnresolvableNodes FailedNodesMap
    // Error message indicating failure
    Error string
  }
  ```

#### 2.3.2 prioritize

prioritize webhook 在 Prioritize 阶段影响 Node 的打分，在所有 Plugin 运行后被调用。

* Request 提供了 Pod 以及筛选出的 Node 列表，与 filter 一致
  
  ```go
  // ExtenderArgs represents the arguments needed by the extender to filter/prioritize
  // nodes for a pod.
  type ExtenderArgs struct {
    // Pod being scheduled
    Pod *v1.Pod
    // List of candidate nodes where the pod can be scheduled; to be populated
    // only if Extender.NodeCacheCapable == false
    Nodes *v1.NodeList
    // List of candidate node names where the pod can be scheduled; to be
    // populated only if Extender.NodeCacheCapable == true
    NodeNames *[]string
  }
  ```

* Response 返回一组 Node 的分数
  
  ```go
  // HostPriorityList declares a []HostPriority type.
  type HostPriorityList []HostPriority

  // HostPriority represents the priority of scheduling to a particular host, higher priority is better.
  type HostPriority struct {
    // Name of the host
    Host string
    // Score associated with the host
    Score int64
  }
  ```

#### 2.3.4 preempt

preempt webhook 在 Preempt 阶段影响如何选择 Pod 抢占，在所有 Plugin 运行后调用。

* Request 提供了 Pod 以及已经筛选出的可以被抢占的 Pod 的列表
  
  ```go
  // Victims represents:
  //   pods:  a group of pods expected to be preempted.
  //   numPDBViolations: the count of violations of PodDisruptionBudget
  type Victims struct {
    Pods             []*v1.Pod
    NumPDBViolations int64
  }

  // ExtenderPreemptionArgs represents the arguments needed by the extender to preempt pods on nodes.
  type ExtenderPreemptionArgs struct {
    // Pod being scheduled
    Pod *v1.Pod
    // Victims map generated by scheduler preemption phase
    // Only set NodeNameToMetaVictims if Extender.NodeCacheCapable == true. Otherwise, only set NodeNameToVictims.
    NodeNameToVictims     map[string]*Victims
    NodeNameToMetaVictims map[string]*MetaVictims
  }
  ```

* Response 经过处理后的可以被抢占的 Pod 列表
  
  ```go
  // ExtenderPreemptionResult represents the result returned by preemption phase of extender.
  type ExtenderPreemptionResult struct {
    NodeNameToMetaVictims map[string]*MetaVictims
  }
  ```

#### 2.3.4 bind

bind webhook 在 Bind 阶段影响 Node 的绑定，在所有 Plugin 运行前调用。

* Request 提供了 Pod 以及最后选择的 Node
  
  ```go
  // ExtenderBindingArgs represents the arguments to an extender for binding a pod to a node.
  type ExtenderBindingArgs struct {
    // PodName is the name of the pod being bound
    PodName string
    // PodNamespace is the namespace of the pod being bound
    PodNamespace string
    // PodUID is the UID of the pod being bound
    PodUID types.UID
    // Node selected by the scheduler
    Node string
  }
  ```

* Response 返回一个 Error，表明拒绝绑定 Node
  
  ```go
  // ExtenderBindingResult represents the result of binding of a pod to a node from an extender.
  type ExtenderBindingResult struct {
    // Error message indicating failure
    Error string
  }
  ```


## 3 扩展调度器的方案

### 3.1 如何配置 Scheduler

从前面所述知识看到，配置一个 Scheduler 有着几种方式：

* **自定义 Plugin**
  
  基于 kube-scheduler 的代码框架，添加自定义的 Plugin，然后编译部署（可能需要在 Scheduling Profiles 配置开启 Plugin）；
   
  如果我们需要很细致的定制调度，可以使用这种方式。不过，我们需要自己对 kube-scheduler 进行版本维护。

* **Scheduling Policy** <important>(deprecated)</important>
   
  不应该是优先考虑的方式，除非 Kubernetes 版本不支持其他方式。
  
* **Scheduling Profiles**
   
  如果 kube-scheduler 内置的 Plugin 能够满足需求，仅仅需要配置，那么可以使用这种方式。

* **Scheduling Porfiles Extender Webhook**
   
  需要简单的定制调度，可以使用这种方式。

第一种与第四种都可以让我们实现我们自己的调度，区别在于 Plugin 可以基于所有的扩展点实现，而 Extender 只能基于大的阶段进行定制。

### 3.2 如何运行 Multiple Schedulers

Multiple Schedulers 是基于配置 Scheduler 的基础上实现的，我们可以将一个定制的 Scheduler 命名，可以与 Kubernetes 原生的 Scheduler（名为 default-scheduler）同时运行。所以关键就是如何命名 Scheduler 了。

目前有两种方式来命名 Scheduler：

* (版本 >= 1.19) 可以使用 Scheduling Profiles 配置文件中的 `profiles[].schedulerName` 给调度器命名，并且支持一个 kube-scheduler 程序运行多个调度器。
* (版本 < 1.22) 可以使用 `kube-scheduler --scheduler-name <name>` 给调度器命名；

## 5 自定义 Plugin

以官方示例 [**scheduler-plugins**](https://github.com/kubernetes-sigs/scheduler-plugins) 来看下如何通过自定义 Plugin 的方式来扩展调度器。

### 5.1 实现自定义的 Plugin

所有的 Plugin 都实现了 `Plugin` 接口，各个扩展点都有着对应的接口。如果要实现自定义的 Plugin，那么实现对应 Plugin interface 即可。

```go
type Plugin interface {
    Name() string
}

type QueueSortPlugin interface {
    Plugin
    Less(*v1.pod, *v1.pod) bool
}

type PreFilterPlugin interface {
    Plugin
    PreFilter(context.Context, *framework.CycleState, *v1.pod) error
}

// ...
```

### 5.2 注册到 Scheduler Framework

定制 Scheduler 时，可以复用 Kubernetes 中的启动命令：

```go
import (
	"os"

	"k8s.io/component-base/cli"
	"k8s.io/kubernetes/cmd/kube-scheduler/app"

	"sigs.k8s.io/scheduler-plugins/pkg/capacityscheduling"
	// ...

	// Ensure scheme package is initialized.
	_ "sigs.k8s.io/scheduler-plugins/apis/config/scheme"
)

func main() {
	// Register custom plugins to the scheduler framework.
	// Later they can consist of scheduler profile(s) and hence
	// used by various kinds of workloads.
	command := app.NewSchedulerCommand(
		app.WithPlugin(capacityscheduling.Name, capacityscheduling.New),
		app.WithPlugin(coscheduling.Name, coscheduling.New),
	)

	code := cli.Run(command)
	os.Exit(code)
}
```

可以看到通过 `WithPlugin` 接口来注册 Plugin，其需要提供 Plugin 的构造函数：

```go
func New(obj runtime.Object, handle framework.Handle) (framework.Plugin, error) {
	args, ok := obj.(*config.CoschedulingArgs)
	if !ok {
		return nil, fmt.Errorf("want args to be of type CoschedulingArgs, got %T", obj)
	}

	pgClient := pgclientset.NewForConfigOrDie(handle.KubeConfig())
	pgInformerFactory := pgformers.NewSharedInformerFactory(pgClient, 0)
	pgInformer := pgInformerFactory.Scheduling().V1alpha1().PodGroups()
	podInformer := handle.SharedInformerFactory().Core().V1().Pods()

	// ...
	return plugin, nil
}
```

注册函数中的两个参数：

* obj - Scheduler Profiles 中提供给插件的特定参数，也就是 `profiles.pluginConfig` 字段；
  
* handle - 提供一些公共共享的结构，例如 `KubeConfig()` `SharedInformerFactory()`；
  
  不过 SharedInformerFactory 仅仅包含部分常用资源，如果需要访问 CRD 还是需要自己创建 Informer。

### 5.3 配置 Scheduler Profile

最后，将自定义的 Plugin 在 Scheduler Profile 中打开：

```yaml
apiVersion: kubescheduler.config.k8s.io/v1beta2
kind: KubeSchedulerConfiguration
leaderElection:
  leaderElect: false
clientConnection:
  kubeconfig: "REPLACE_ME_WITH_KUBE_CONFIG_PATH"
profiles:
- schedulerName: default-scheduler
  plugins:
    queueSort:
      enabled:
      - name: Coscheduling
      disabled:
      - name: "*"
    preFilter:
      enabled:
      - name: Coscheduling
    postFilter:
      enabled:
      - name: Coscheduling
    permit:
      enabled:
      - name: Coscheduling
    reserve:
      enabled:
      - name: Coscheduling
    postBind:
      enabled:
      - name: Coscheduling
  pluginConfig:
  - name: Coscheduling
    args:
      permitWaitingTimeSeconds: 10
      deniedPGExpirationTimeSeconds: 3
```

启动时，通过 `kube-scheduler --config=$file` 配置对应的 Profiles 文件即可。

## 参考

* [**Doc: Scheduling Policies**](https://kubernetes.io/docs/reference/scheduling/policies/)
* [**Doc: Scheduler Configuration**](https://kubernetes.io/docs/reference/scheduling/config/)
* [**Doc: Scheduling Framework**](https://kubernetes.io/docs/concepts/scheduling-eviction/scheduling-framework/)
* [**Blog: 自定义 Kubernetes 调度器**](https://www.qikqiak.com/post/custom-kube-scheduler/)
* [**GitHub：schedule-plugins**](https://github.com/kubernetes-sigs/scheduler-plugins)
* [**GitHub：Scheduler extender**](https://github.com/kubernetes/design-proposals-archive/blob/main/scheduling/scheduler_extender.md)
