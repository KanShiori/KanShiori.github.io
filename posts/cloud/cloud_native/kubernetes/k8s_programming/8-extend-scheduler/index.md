# K8s 编程 - 8 - Custom Scheduler


kube-scheduler 负责集群资源的调度，原生的 Kubernetes Scheduler 可以满足大多数情况的需求。不过针对于不同业务的特殊的调度需求，Kubernetes 提供了多种扩展调度器的方式。

## 1 Scheduler 扩展点

Scheduler Framework 在整个调度过程中有着一系列的扩展点，支持用户以 Plugin 的方式进行扩展。

整个的调度流程中的扩展点如下图：
{{< find_img "img1.png" >}}

所有的扩展点都是由 Plugin 实现的，允许配置开启或关闭，也可以添加自定义的 Plugin。这在后续的 KubeSchedulerConfiguration 中深入说明。

### 1.1 QueueSort

该插件用于对 scheduling queue 中的 Pod 进行排序，一次只能存在一个 QueueSort Plugin。

一个 QueueSort Plugin 本质上就是提供了一个 Less(Pod1, Pod2) 函数。

### 1.2 PreFilter

该插件用于对 Pod 的信息进行预处理，或者检查一些集群或 Pod 必须满足的前提条件。

如果 PreFilter Plugin 返回一个 error，那么调度流程直接终止。

### 1.3 Filter

该插件用于过滤不能运行 Pod 的 Node。

对于每一个 Node，调度器将按照顺序执行 Filter Plugin。如果任意一个 Filter Plugin 将 Node 标记为不可行，那么剩下的 Filter Plugin 就不会再运行。

所有 Node 会并行进行 Filter。

### 1.4 PostFilter

这些插件在 Filter 阶段之后被调用，只有在没有为 Pod 找到可运行的 Node 时才会运行。所有插件按照顺序运行。

如果任意一个 PostFilter Plugin 将 Node 标记为 Schedulable，剩下的 PostFilter Plugin 就不会再运行。

一个典型的 PostFilter Plugin 是抢占，它通过抢占其他 Pod 来调度 Pod。

### 1.5 PreScore

这些插件用于执行 "预打分" 工作，为 Score Plugin 生成一个可共享的状态。

如果 PreScore 插件返回 error，调度流程终止。

### 1.6 Score

这些插件用于为所有可选 Node 打分，调度器会为每个 Node 调用每个 Score Plugin。

打分有一个明确定义的整数范围，代表最低和最高分数。

在 NormalizeScore 阶段后，Scheduler 将根据配置的插件权重，合并所有插件的打分。

### 1.7 NormalizeScore

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

### 1.8 Reserve

Reserve Plugin 有两个方法：`Reserve` 和 `Unreserve`，会带有 Reserve 和 Unreserve 调度阶段的信息。有状态的插件可以使用该扩展点来获得 Node 上为 Pod 预留的资源。

该事件发生在调度器将 Pod 绑定到 Node 之前，目的是避免调度器在等 Pod 与 Node 绑定过程中，有新的 Pod 调度到该 Node 上，发生实际使用资源超过可用资源的情况。（因为绑定 Pod 是异步的）

这是调度过程的最后一个步骤，Pod 进入 reserved 状态后，要么绑定失败触发 Unreserve，要么由 Postbind 扩展结束绑定过程。

### 1.9 Permit

Permit Plugin 在调度周期的最后调用，为了组织或延迟绑定后候选的 Node。

Permit Plugin 做以下的事情：
1. approve
   
   一旦所有 Permit Plugin approve 一个 Pod，那么就会进行绑定操作。
2. deny
   
   如果任一 Permit Plugin deny 一个 POod，那么将其重新放入调度队列。这也会触发 Unreserve 阶段。
3. wait（with a timeout）
   
   如果一个 Permit Plugin 返回 wait，那么该 Pod 会在 waiting Pod list 中等待，直到该 Pod 被 approve。

   如果 wait timeout 触发，那么变为 deny Pod。

### 1.10 PreBind

这些插件用于在 Pod 被绑定前执行一些工作。

例如，PreBind Plugin 可以创建 network volume，并将其绑定到目标 Node。

如果任一 PreBind Plugin 返回 error，Pod 会被拒绝，然后重新放入调度队列。

### 1.11 Bind

这些插件用于将 Pod 绑定到 Node。

每个 Bind Plugin 会按照配置顺序调用，每个 Bind Plugin 可以选择是否处理该 Pod。如果某个 Bind Plugin 决定处理该 Pod，后续的 Bind Plugin 会跳过。

### 1.12 PostBind

Post-bind Plugin 在绑定成功后被调用，是绑定周期的最后一个阶段。常常用于清理相关资源。


## 2 Scheduling Policy

Scheduling Policy 是一种老版本配置调度器的方式，可以定制 Predicates 与 Priorities 使用的插件，也可以通过 Extender 对阶段进行扩展。

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

运行 kube-scheduler 时，我们可以通过 `kube-scheduler --policy-config-file <filename>` 指定一个 Policy 文件，或者通过 `kube-scheduler --policy-configmap <ConfigMap> ----policy-configmap-namespace <Namespace>` 指定一个 ConfigMap 获取 Policy。加上 1.22 之前 kube-scheduler 支持的 `--scheduler-name` 参数，很简单的定义出一个新的调度器。
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

使用上述命令，我们运行了一个名为 my-scheduler 的调度器，通过 Pod 定义的 `spec.schedulerName` 可以指定使用该调度器。

{{< admonition warning "Scheduling Policy 是旧版本的">}}
1.22 后，kube-scheduler 已经不支持 `--scheduler-name` 参数了，因此已经无法使用 Scheduler Policy 来得到一个新的调度器了。

新版多调度器只能通过 [**Multiple Profiles**](#32-multiple-profiles) 方式实现，而其与 Scheduler Policy 是不兼容的。所以如果你要考虑运行一个新的调度器，应该直接使用 Scheduling Profiles 方式。

当然，Schedule Policy 还是能够实现定制 default scheduler 的作用，不过 Scheduling Profiles 能实现的更多。
{{< /admonition >}}


## 3 Scheduling Profiles

无论哪种方式扩展调度器，都是围绕着通过 Scheduling Profiles 对 kube-scheduler 进行配置实现的。

运行 kube-scheduler 时，通过 `kube-scheduler --config <filename>` 指定 Scheduling Profiles。其文件内容是 v1beta1 或者 v1beta2 版本的 KubeSchedulerConfiguration 结构。

最简单的配置如下：
```yaml
apiVersion: kubescheduler.config.k8s.io/v1beta2
kind: KubeSchedulerConfiguration
clientConnection:
  kubeconfig: /etc/srv/kubernetes/kube-scheduler/kubeconfig
```

### 3.1 配置 Plugin

在 KubeSchedulerConfiguration 定义中的 profiles 字段对 kube-scheduler 的插件进行配置。配置方式按照 Scheduler 扩展点 中所属的扩展点。

对于每个扩展点，你可以关闭某些默认开启的 Plugin，或者打开你自己定义的 Plugin。

例如下面示例中，关闭了 Score 扩展点的 NodeResourcesLeastAllocated Plugin，打开 Score 扩展点的 MyCustomPluginA Plugin 与 MyCustomPluginB Plugin。
```yaml
apiVersion: kubescheduler.config.k8s.io/v1beta2
kind: KubeSchedulerConfiguration
profiles:
  - plugins:
      score:
        disabled:
        - name: NodeResourcesLeastAllocated
        enabled:
        - name: MyCustomPluginA
          weight: 2
        - name: MyCustomPluginB
          weight: 1
```
* profiles[].plugins.score - 表明配置 score 扩展点的插件。
  * disabled - 关闭插件
  * enabled - 打开插件

{{< admonition note "插件是编译到程序中的">}}
在 `profiles.plugins` 字段中配置的插件都是预编译到 scheduler 代码中。也就是说，你需要先通过 scheduler framework 编写自定义 Plugin 代码，编译成 scheduler 镜像，然后才能打开你定义的插件。
{{< /admonition >}}

### 3.2 Multiple Profiles

可以通过 `profiles` 字段配置多个 Profile，每个 Profile 就类似与一个 Scheduler 的 “分身”，可以进行不同扩展点 Plugin 的配置，并且有着独立的 `schedulerName`。

如下配置中，将运行两个 Scheduler：default-scheduler 运行所有默认 Plugin，no-scoring-scheduler 关闭了所有的 Score Plugin。
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

默认下，一个 Profile 的 scheduler name 为 "default-scheduler"，也就是 Kubernetes 中默认的调度器名字（Pod 没有指定 `.spec.schedulerName` 字段时，自动将其设置为 "default-scheduler"）。

### 3.3 Extender

上面都是基于内置的 Plugin 进行配置，Scheduling Profile 也完全能够实现 Scheduling Policy 的 Webhook 扩展功能。
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
    preemptVerb: preempt
    weight: 1
    enableHTTPS: false
    httpTimeout: 30s
```
* `extenders` 字段用于定制扩展的 Webhook，调度器会在对应的阶段调用对应的 URL。这对于所有的 profiles 都是生效的。

按照示例中的定义，我们定义一个新调度器只需要一个配置：
```bash
$ kube-scheduler --config <config-file>
```










## 4 方案

### 4.1 如何配置 Scheduler

从前面所述知识看到，配置一个 Scheduler 有着几种方式：
1. 基于 kube-scheduler 的代码框架，添加自定义的 Plugin，然后编译部署（可能需要在 Scheduling Profiles 配置开启 Plugin）；
   
   如果我们需要很细致的定制调度，可以使用这种方式。不过，我们需要自己对 kube-scheduler 进行版本维护。
2. 通过 Scheduling Policy 定制；
   
   不应该是优先考虑的方式，除非 Kubernetes 版本不支持其他方式。
3. 通过 Scheduling Profiles 定制内置的 Plugin；
   
   如果 kube-scheduler 内置的 Plugin 能够满足需求，仅仅需要配置，那么可以使用这种方式。
4. 通过 Scheduling Porfiles 中的 Extender 功能配置自定义的 Webhook。
   
   需要简单的定制调度，可以使用这种方式。

第一种与第四种都可以让我们实现我们自己的调度，区别在于 Plugin 可以基于所有的扩展点实现，而 Extender 只能基于大的阶段进行定制。

### 4.2 如何运行 Multiple Schedulers

Multiple Schedulers 是基于配置 Scheduler 的基础上实现的，我们可以将一个定制的 Scheduler 命名，可以与 Kubernetes 原生的 Scheduler（名为 default-scheduler）同时运行。所以关键就是如何命名 Scheduler 了。

目前有两种方式来命名 Scheduler：
1. 版本 < 1.22 可以使用 `kube-scheduler --scheduler-name <name>` 给调度器命名；
2. 版本 >= 1.19 可以使用 Scheduling Profiles 配置文件中的 `profiles[].schedulerName` 给调度器命名，并且支持一个 kube-scheduler 程序运行多个调度器。


## 5 实现 Plugin


## 6 实现 Extender


## 参考

* [**Doc: Scheduling Policies**](https://kubernetes.io/docs/reference/scheduling/policies/)
* [**Ooc: Scheduler Configuration**](https://kubernetes.io/docs/reference/scheduling/config/)
* [**Doc: Scheduling Framework**](https://kubernetes.io/docs/concepts/scheduling-eviction/scheduling-framework/)
* [**Blog: 自定义 Kubernetes 调度器**](https://www.qikqiak.com/post/custom-kube-scheduler/)
