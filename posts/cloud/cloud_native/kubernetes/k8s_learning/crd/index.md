# Kubernetes - CustomResourceDefinition


## 1 概述

Kubernetes 不仅仅是一个编排框架，更是提供了极大的扩展性，可以看做一个资源框架。你可以基于 k8s 提供的种种功能，来满足你应用的需要。

以贵司 TiDB Operator 来举个例子，Operator 用于管理 TiDB 的集群，其核心组件包括 PD、TiDB、TiKV 程序。这些核心组件在最底层都是以 Pod 方式运行的，并且是多副本形式。而我们将所有的 Pod + Service 等组合在一起，构成了 TiDBCluster 这个 CustomResource。

所以，当你想部署 TiDB 集群时，只需要修改 TiDBCluster 这个资源的配置，然后提交到 Kubernetes 中。TiDB Operator（CRD Controller）就会根据 TiDBCluster 来操作，使得集群按照期望状态部署。

{{< admonition note "为什么能够做到？">}}
Kubernetes 底层架构被尽可能的进行拆分与解耦，这样使得上层可以有很高的扩展性。
{{< /admonition >}}

对于使用自定义资源，最基本要涉及到：

* **`CustomResourceDefinition`** ：一种 Resource 类型，为了让 k8s 能够认识到你的自定义资源，需要先提交 CRD；

* **`CustomResource Controller`** ：管理 CR 的程序，以 Pod 方式运行，并通过 k8s API 来管理 CR 以及执行操作；

* **`CustomResource`** ：你的自定义资源，就像 Pod Service 这种资源一样使用；

{{< admonition tip 类比>}}
* CRD 是类定义，告诉编译器类型；
* CR 基于类创建的对象；
* CR Controller 是代码逻辑；
{{< /admonition >}}

## 2 CRD

`CRD` 是使用的 CustomResource 的基础，能够**让 APIServer 认识到你的自定义的资源**。

基本的 CRD 定义如下（来自官网）：

```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  # 名字必需与下面的 spec 字段匹配，并且格式为 '<名称的复数形式>.<组名>'
  name: crontabs.stable.example.com
spec:
  # 组名称，用于 REST API: /apis/<组>/<版本>
  group: stable.example.com
  # 列举此 CustomResourceDefinition 所支持的版本
  versions:
    - name: v1
      # 每个版本都可以通过 served 标志来独立启用或禁止
      served: true
      # 其中一个且只有一个版本必需被标记为存储版本
      storage: true
      schema:
        openAPIV3Schema:
          type: object
          properties:
            spec:
              type: object
              properties:
                cronSpec:
                  type: string
                image:
                  type: string
                replicas:
                  type: integer
  # 可以是 Namespaced 或 Cluster
  scope: Namespaced
  names:
    # 名称的复数形式，用于 URL：/apis/<组>/<版本>/<名称的复数形式>
    plural: crontabs
    # 名称的单数形式，作为命令行使用时和显示时的别名
    singular: crontab
    # kind 通常是单数形式的驼峰命名（CamelCased）形式。你的资源清单会使用这一形式。
    kind: CronTab
    # shortNames 允许你在命令行使用较短的字符串来匹配资源
    shortNames:
    - ct
```

{{< admonition note "生成 CRD">}}
自己编写与维护 CRD 是令人烦躁的，因为总是要随着代码中结构体的修改而修改。目前，主流都是使用 [**controller-gen**](https://github.com/kubernetes-sigs/controller-tools) 来自动从结构体生成 CRD。
{{< /admonition >}}

### 2.1 URL

创建 CRD 后，会自动在 API Server 建立其对应的 Resource Endpoint URL，使得可以通过 HTTP 方式来查询与操作该 CR。

其 URL 为 `/apis/<Group>/<Version>/namespaces/<ns>/<Resources>/…`。例如，示例中的 API endpoint 为 `/apis/stable.example.com/v1/namespaces/*/crontabs/…`。

### 2.2 Validation

`spec.validation` 用以用户在提交 CR 时，**对其定义进行合法性检查**。

我们对上面的示例对其增加两个检查：

```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: crontabs.stable.example.com
spec:
  # ...
  validation:
   # openAPIV3Schema is the schema for validating custom objects.
    openAPIV3Schema:
      properties:
        spec:
          properties:
            cronSpec:
              type: string
              pattern: '^(\d+|\*)(/\d+)?(\s+(\d+|\*)(/\d+)?){4}$'
            replicas:
              type: integer
              minimum: 1
              maximum: 10
```

* spec.cronSpec 必须是匹配正则表达式的字符串
* spec.replicas 必须是从 1 到 10 的整数

更多的 OpenAPIV3Schema 检查语法见 [**OpenAPI v3 schemas**](https://github.com/OAI/OpenAPI-Specification/blob/main/versions/3.0.0.md#schemaObject)。

1.25 开始，Validate 还支持使用 [**CEL**](../cel-in-k8s/) 语法，支持更高级的验证。通过 `x-kubernetes-validations` 扩展验证语法。

```yaml
  openAPIV3Schema:
    type: object
    properties:
      spec:
        type: object
        x-kubernetes-validations:
          - rule: "self.minReplicas <= self.replicas"
            message: "replicas should be greater than or equal to minReplicas."
          - rule: "self.replicas <= self.maxReplicas"
            message: "replicas should be smaller than or equal to maxReplicas."
        properties:
        # ...
```

### 2.3 字段剪裁

CRD 中定义了支持的数据结构，如果你指定了 CRD 中未定义的字段，默认情况下 APIServer 拒绝请求。

不过，如果你使用了 `--validate=false` 跳过 APIServer 检查，那么这些未定义的字段会被剪裁掉。

一些情况下，我们可能某些字段想预留为动态字段（类似于 Map），那么需要在对应字段 Schema 定义中指定 `x-kubernetes-preserve-unknown-fields: true`，告知 APIServer 不要剪裁该属性（仅仅针对该属性，不会对子属性生效）。

```yaml
type: object
properties:
  json:  # json 可以保存任何值
    x-kubernetes-preserve-unknown-fields: true
```

### 2.4 IntOrString

如果 CRD 中某个字段是 Int 类型，但是能够接受输入是 String 类型，可以将字段指定 `x-kubernetes-int-or-string: true`。

```yaml
type: object
properties:
  foo:  # foo 的输入可以是 int 或 string
    x-kubernetes-int-or-string: true
```

### 2.5 RawExtension

如果想在 CRD 定义中内嵌入动态的子资源（包含 `apiVersion` 与 `kind` 字段），可以通过 `x-kubernetes-embedded-resource: true` 来指定：

```yaml
type: object
properties:
  foo:  # foo 可以包含一个完整的子资源，例如 Pod
    x-kubernetes-embedded-resource: true
    x-kubernetes-preserve-unknown-fields: true
```

设置了 RawExtension 后，子资源的 `apiVersion`、`kind` 和 `metadata` 都会隐式的完成合法性验证。

### 2.6 默认值

在 OpenAPI V3 Validation Schema 下，允许你为某些项指定默认值。

下面示例为 `spec.replica` 指定了一个默认值：

```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: crontabs.stable.example.com
spec:
  group: stable.example.com
  versions:
    - name: v1
      schema:
        openAPIV3Schema:
          type: object
          properties:
            spec:
              type: object
              properties:
                replicas:
                  type: integer
                  minimum: 1
                  maximum: 10
                  default: 1  # 设置默认值
```

设置默认值的操作会发生以下情况：

* 发送给 APIServer 请求后，APIServer 会根据 External Version 设置默认值；

* 从 ETCD 中读取对象时，会根据 Storage Version 填充默认值；

* 在 Mutate Webhook 更新资源时，会根据 External Version 填充默认值；

{{< admonition note Note>}}
从 ETCD 读取对象时设置的默认值不会写回 ETCD，需要人为通过 API 执行写入才能将默认值写入 ETCD 中。
{{< /admonition >}}

### 2.7 Nullable

默认下，将赋值为 `null` 的字段会被自动剪裁掉，不会保存。如果该字段设置了默认值，那么会赋值默认值。

如果字段设置 `nullable: true`，那么 `null` 值也会保留。如果设置了默认值，不会赋值默认值而是保留 `null` 值。

```yaml
type: object
properties:
  spec:
    type: object
    properties:
      foo:  # 设置为 null 时，最后赋值是 default
        type: string
        nullable: false
        default: "default"
      bar:  # 设置为 null 时，最后赋值是 null
        type: string
        nullable: true
      baz:  # 设置为 null时，最后该字段会被剪裁
        type: string
      foa:  # 设置为 null 时，最后赋值是 null
        type: string
        nullable: true
        default: "default"
```

### 2.8 打印 Columns

通过 `additionalPrinterColumns` 配置当 `kubectl get` 执行时，打印出的列。

```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: crontabs.stable.example.com
spec:
  # ...
    additionalPrinterColumns:  # 打印出 NAME SPEC REPLICAS AGE 列
    - name: Spec
      type: string
      description: The cron spec defining the interval a CronJob is run
      jsonPath: .spec.cronSpec
    - name: Replicas
      type: integer
      description: The number of jobs launched by the CronJob
      jsonPath: .spec.replicas
    - name: Age
      type: date
      jsonPath: .metadata.creationTimestamp
```

### 2.9 Subresources

Subresources 指定是资源请求 URL 的两个子路径，目前支持设置 `/status` 和 `/scale`。使用 Subresource 最大的好处是将一些操作请求路径抽离，因此可以更加精细化的控制访问权限。

#### 2.9.1 Status

如果定义了 status 子资源，会在资源的 API Group 中添加 `/status` API 路径。

```yaml
```yaml
# ...
spec:
  group: stable.example.com
  versions:
    - name: v1
      subresources:
        status: {}
```

* 对 `/status` API 的更新会忽略 `status` 字段以外的所有字段更新。

* 对 `/status` API 的更新仅仅检查 `status` 内容的合法性。

* 无法通过其他 API 来更新 `status` 字段，全部会被忽略。

* 所有资源更新，除了更新 `status` 或 `metadata` 字段，都会增加 `metadata.generation` 的值。

#### 2.9.2 Scale

启动 scale 子资源后，会在资源的 API Group 中添加 `/scale` API 路径。开启了 `/scale` API 后，`kubectl scale` 等功能就可以操作 CR。

```yaml
# ...
spec:
  group: stable.example.com
  versions:
    - name: v1
      subresources:
        scale:
          # 指定 spec.replicas 路径
          specReplicasPath: .spec.replicas
          # 指定 status.replicas 路径
          statusReplicasPath: .status.replicas
          # （可选）设置后才能使用 HPA
          labelSelectorPath: .status.labelSelector
```


### 2.5 Categories

`categories` 可以将资源分组，通过使用 `kubectl get <categories>` 来获取一组资源。

```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: crontabs.stable.example.com
spec:
  # … 
  names:
    # categories 是定制资源所归属的分类资源列表
    categories:
    - all
```

上面例子将 CR 分类到 "all" 类别中，使得 `kubectl get all` 可以获取到该类别下的所有已经创建的资源。

### 2.6 Finalizer

Finalizer 会在删除 CR 时自动被调用，可以实现对 CR 的清理工作。

```yaml
apiVersion: "stable.example.com/v1"
kind: CronTab
metadata:
  finalizers:
  - stable.example.com/finalizer
```

当用户请求删除资源对象时，Kubernetes 会先将 `metadata.deletionTimestamp` 的值，然后将其标记为开始删除。设置该值后，不断的轮询检查是否所有的 `finalizers` 被移除。

在 Controller 实现中，发现 `metadata.deletionTimestamp` 被设置，知道了资源对象正在删除流程，于是开始执行其负责的的清理工作。执行完成后，从 `finalizers` 中移除对应的一项。

所有 finalizers 移除后，Kubernetes 才将其真正的资源对象删除。

{{< admonition note Note>}}
Controler 可以读取 `metadata.deletionGracePeriodSeconds` 的值，用以得到期望的优雅删除时间是多久。
{{< /admonition >}}

### 2.7 删除 CRD

当你删除一个 CRD 时，APIServer 就会卸载对应的 Resource Endpoint，<important>并且会删除所有的 CR</important>。

## 3 CR

在提交 CRD 后，我们就可以创建 CR 来提交了。当然，CR 中的 spec 相关的属性 k8s 是不会使用的，而是在 Controller 中解析并使用。

基于上面示例，创建一个 CronTab 资源：

```yaml
apiVersion: "stable.example.com/v1"
kind: CronTab
metadata:
  name: my-new-cron-object
spec:
  cronSpec: "* * * * /5"
  image: my-awesome-cron-image
```

当我们提交资源后，其资源就保存在了 Kubernetes 中，可以查看与删除。但是，我们并没有注册 CronTab 的 Controller，所以没有基于该资源进行系统上的管理。

## 4 CR Controller

上面看到，CR 与 CRD 都是静态的资源，而 CR Controller 就是基于已经创建的 CR 来进行实际的系统操作，使得整个系统是向期望的状态发展。

Controller 本质上是一个服务，监听着集群的资源，并执行一些操作。

关于 Kubernetes 编程，可以参考：[**Kubernetes 编程**](../../k8s_programming/1-basic/) 相关文章。

### 4.1 Controller 模式

不论是内置的 Controller，还是你需要编写的自定义的 Controller，都需要遵循 [控制器模式]^(Controller Pattern) 的方式实现。

我理解的控制器模式是：Controller 不断永久循环执行着 Controll Loop，而每一个 Loop 都是**基于当前实际的资源状态（status），进行系统操作，向期望的资源状态（spec）靠拢**。

所以整个逻辑是基于状态的，而不是基于事件的，这也就是 [声明式 API]^(Declarative API) 与 [命令式 API]^(Imperative API) 的区别。

### 4.2 工作原理

Kubernetes 提供了资源状态的存储功能，而 Controller 需要实现的就是基于对资源的更变的控制逻辑。

一个 Controller 的工作原理，可以使用以下流程图表示：

{{< image src="img1.png" height=200 >}}

整体上看，Controller 实现分为三个组件：

* **`Informer`** ：提供特定对象的存储，事件触发功能，即 List 与 Watch 功能；

* **`WorkQueue`** ：对象变更事件通知队列，通过队列来防止阻塞 Informer；

* **`Control Loop`** ：控制循环，基于事件来执行对应的操作；

#### 4.2.1 Informer

Informer 是 Kubernetes 提供的代码模块，针对于特定的 API 对象，包含几个职责：

1. 同步 ETCD 中对应 API 对象，将所有对象缓存到本地；
   
2. 某个 API 对象的任何变更，都可以触发对象变更事件；
   
3. 触发/检测到对象变更事件时，触发注册的 `ResourceEventHandler` 回调，即 `AddFunc` `UpdateFunc` `DeleteFunc` 回调；

针对于 etcd 的 List 与 Watch 机制，Informer 的同步机制也包含两种：

* 使用 Watch 的事件增量同步：当 APIServer 有对象的创建、删除或者更新，Informer.Reflector 模块会收到对应事件，解析后放入 Delta FIFO Queue；

* 使用 List 的定期周期全量同步：经过一定周期，Informer 会通过 List 来进行本地对象的强制更新。

  定期全量同步在即使对象没有发生变化情况下，也会调用 Update 回调。因此，代码中需要通过 `resoureVersion` 判断资源是否有真正的更新。

更多实现细节见 [**Informer 实现**](../../k8s_programming/informer-implement/)。

