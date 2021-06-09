# [WIP] CRD 学习


## 1 概述
Kubernetes 不仅仅是一个编排框架，更是提供了极大的扩展性，可以看做一个资源框架。你可以基于 k8s 提供的种种功能，来满足你应用的需要。

其中 CRD 就是允许你来自定义 Resource，可以不修改 Kubernetes 代码来实现类似于 NetworkPolicy 这样的资源。
{{< admonition note Note >}}
可以理解 Kubernetes 提供了基础的框架，Pod Service Volume 等都是最基础的组件，而我们可以在这些组件之上进行**再次抽象与组合**，将其赋予一个特定的概念 CustomResource
{{< /admonition >}}

以贵司 TiDB Operator 来举个例子，Operator 用于管理 TiDB 的集群，其核心组件包括 PD、TiDB、TiKV 程序。这些核心组件在最底层都是以 Pod 方式运行的，并且是多副本形式。而我们将所有的 Pod + Service 等组合在一起，构成了 TiDBCluster 这个 CustomResource。

所以，当你想部署 TiDB 集群时，只需要修改 TiDBCluster 这个资源的配置，然后提交到 Kubernetes 中。TiDB Operator（CRD Controller）就会根据 TiDBCluster 来操作，使得集群按照期望状态部署。
{{< admonition note "为什么能够做到？">}}
Kubernetes 底层架构被尽可能的进行拆分与解耦，这样使得上层可以有很高的扩展性。
{{< /admonition >}}

对于使用自定义资源，最基本要涉及到：
* **`CustomResourceDefinition`** ：一种 Resource 类型，为了让 k8s 能够认识到你的自定义资源，需要先提交 CRD；
* **`CustomResource Controller`** ：管理 CR 的程序，以 Pod 方式运行，并通过 k8s API 来管理 CR 以及执行操作；
* **`CustomResource`** ：你的自定义资源，就像 Pod Service 这种资源一样使用；

{{< admonition tip 比喻>}}
我习惯以编程的方式来看到这 3 个：
* CRD 是类定义，告诉编译器类型；
* CR 基于类创建的对象；
* CR Controller 是代码逻辑；
{{< /admonition >}}

## 2 CRD 
`CRD` 是使用的 CustomResource 的基础，能够**让 k8s 认识到你的自定义的资源**。

基本的 CRD 定义如下（来自官网）：
```yaml
apiVersion: apiextensions.k8s.io/v1beta1
kind: CustomResourceDefinition
metadata:
  name: crontabs.stable.example.com
spec:
  group: stable.example.com
  versions:
  - name: v1beta1
    # Each version can be enabled/disabled by Served flag.
    served: true
    # One and only one version must be marked as the storage version.
    storage: true
  - name: v1
    served: true
    storage: false
  scope: Namespaced
  names:
    # plural name to be used in the URL: /apis/<group>/<version>/<plural>
    plural: crontabs
    singular: crontab
    kind: CronTab
    shortNames:
    - ct
  additionalPrinterColumns:
  - name: Spec
     type: string
     description: The cron spec defining the interval a CronJob is run
     jsonPath: .spec.cronSpec
```
* `metadata.name` ：命名，必须是 `<plural>.<group>`；
* spec
    * `group` ：组织，用以 REST API 注册（`/apis/<group>/<version>`），基本以 URL 方式；
    * `versions` ：版本号，用以 REST API 注册（`/apis/<group>/<version>`）；
    * `scope` ：资源的范围，Namespaced 或者 Cluster；
    * names
        * `plural` ：CRD 的复数命名；
        * `singular` ：CRD 的单数命名；
        * `kind` ：CR 的名字，定义 CR 时使用；
        * `shortNames` ：简短的名称，用以 kubectl 时简写；
* `additionalPrinterColumns` ：kubectl 打印出的额外信息，从 CR 的 spec 中获取相关的信息；

### 2.1 URL
创建 CRD 后，会自动在 API Server 建立其对应的 Endpoint URL，使得可以通过 HTTP 方式来查询与操作该 CR。

其 URL 为 `/apis/<group>/<version>/namespaces/<ns>/<plural>/…`。

例如，示例中的 API endpoint 为 `/apis/stable.example.com/v1/namespaces/*/crontabs/…`

### 2.2 Validation
`spec.validation` 用以用户在提交 CR 时，**对其定义进行合法性检查**。
> 该功能需要配置 kube-apiserver 的 --feature-gates=CustomResourceValidation=true。

我们对上面的示例对其增加两个检查：
* spec.cronSpec 必须是匹配正则表达式的字符串
* spec.replicas 必须是从 1 到 10 的整数

```yaml
apiVersion: apiextensions.k8s.io/v1beta1
kind: CustomResourceDefinition
metadata:
  name: crontabs.stable.example.com
spec:
  group: stable.example.com
  version: v1
  scope: Namespaced
  names:
    plural: crontabs
    singular: crontab
    kind: CronTab
    shortNames:
    - ct
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
更多的 openAPIV3Schema 检查语法见 [**OpenAPI v3 schemas**](https://github.com/OAI/OpenAPI-Specification/blob/main/versions/3.0.0.md#schemaObject)。

### 2.3 Defaulting 与 Nullable 
在 OpenAPI v3 validation schema 下，允许你为某些项指定默认值。
> apiVersion 必须是 apiextensions.k8s.io/v1

下面示例为 spec.cronSpec 指定了一个默认值，而 spec.image 默认为 null 值：
```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: crontabs.stable.example.com
spec:
  group: stable.example.com
  versions:
    - name: v1
      served: true
      storage: true
      schema:
        # openAPIV3Schema is the schema for validating custom objects.
        openAPIV3Schema:
          type: object
          properties:
            spec:
              type: object
              properties:
                cronSpec:
                  type: string
                  pattern: '^(\d+|\*)(/\d+)?(\s+(\d+|\*)(/\d+)?){4}$'
                  default: "5 0 * * *"
                image:
                  type: string
                  nullable: true
                replicas:
                  type: integer
                  minimum: 1
                  maximum: 10
                  default: 1
  scope: Namespaced
  names:
    plural: crontabs
    singular: crontab
    kind: CronTab
    shortNames:
    - ct
```
### 2.4 Subresources
TODO

### 2.5 Categories
`spec.names.categories` 可以将资源分组，通过使用 `kubectl get <categories>` 来获取一组资源。
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

Controller 需要我们自行编写其逻辑，然后注册到 Kubernetes 中。

### 4.1 Controller 模式
不论是内置的 Controller，还是你需要编写的自定义的 Controller，都需要遵循 [控制器模式]^(Controller Pattern) 的方式实现。

我理解的控制器模式是：Controller 不断永久循环执行着 Controll Loop，而每一个 Loop 都是**基于当前实际的资源状态（status），进行系统操作，向期望的资源状态（spec）靠拢**。

所以整个逻辑是基于状态的，而不是基于事件的，这也就是 [声明式 API]^(Declarative API) 与 [命令式 API]^(Imperative API) 的区别。

### 4.2 工作原理
Kubernetes 提供了资源状态的存储功能，而 Controller 需要实现的就是基于对资源的更变的控制逻辑。

一个 Controller 的工作原理，可以使用以下流程图表示：
{{< find_img "img1.png" >}}

整体上看，Controller 实现分为三个组件：
* **`Informer`** ：提供特定对象的存储，事件触发功能，即 List 与 Watch 功能；
* **`WorkQueue`** ：对象变更事件通知队列，通过队列来防止阻塞 Informer；
* **`Control Loop`** ：控制循环，基于事件来执行对应的操作；

#### 4.2.1 Informer
Informer 是 Kubernetes 提供的代码模块，针对于特定的 API 对象，包含几个职责：
1. 同步 etcd 中对应 API 对象，将所有对象缓存到本地；
1. 某个 API 对象的任何变更，都可以触发对象变更事件；
1. 触发/检测到对象变更事件时，触发注册的 ResourceEventHandler 回调，即 AddFunc UpdateFunc DeleteFunc 回调；

针对于 etcd 的 List 与 Watch 机制，Informer 的同步机制也包含两种：
* 使用 Watch 的事件增量同步：当 APIServer 有对象的创建、删除或者更新，Informer.Reflector 模块会收到对应事件，解析后放入 Delta FIFO Queue；
* 使用 List 的定期周期全量同步：经过一定周期，Informer 会通过 List 来进行本地对象的强制更新。

  该更新操作会强制触发“更新事件”，从而调用 UpdateFunc 回调
{{< admonition note "主动判断 ResourceVersion">}}
所以 UpdateFunc 要通过对象的 ResourceVersion 来判断，是否对象有着真正的更新。
{{< /admonition >}}

同时，Informer 中异步的不断读取 Delta FIFO Queue 中事件，并触发其注册的**回调函数**（AddFunc UpdateFunc DeleteFunc）。

## 5 自定义资源示例
### 5.1 使用 kubebuilder
整体过程来自于官方文档 [**QuickStart**](https://book.kubebuilder.io/quick-start.html)
1. 安装 kubebuilder
1. 进行初始化
```bash
$ kubebuilder init --domain shiori.me --repo shiori.me/crd-practice --skip-go-version-check
…
Next: define a resource with:
$ kubebuilder create api
```
3. 创建一个 API
```bash
$ kubebuilder create api --group shiori --version v1 --kind Echo
Create Resource [y/n]
y
Create Controller [y/n]
y
```
* Resource 会让其生成资源的定义文件 api/v1/echo_types.go
* Controller 会让其生成 Controller 逻辑框架 controllers/echo_controller.go

我们看一下基本的目录结构：
TODO

