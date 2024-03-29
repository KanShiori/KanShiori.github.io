# Crossplane 入门


与 [**Pulumi**](../use-pulumi/) 一样，Crossplane 也是一个 Iac 的平台。Crossplane 与 Kubernetes 的集成度更高，允许用户通过创建 Kubernetes 资源来创建云资源。

## 1 基本概念

Crossplane 的一个资源的部署模型如下：

{{< image src="img1.png" height="400" >}}

其中有一些关键的资源（都是 Kubernetes 中的资源）：

* CompositeResource（XR）- 类似于 CR，也就是具体 XRD 的实例化；
* CompositeResource（XRC）- 业务层使用的 XRC，进而生成 XR；
* CompositeResourceDefinition（XRD）- 类似于 CRD 描述了该资源的定义；
* Composition - 描述了 XR 由哪些 MR 组成，以及信息如何传递；
* Managed Resource（MR） - 也就是 CR，各个 Provider 原生支持的资源，对应物理云资源；
* ProviderConfig - 提供给 Provider 访问 Cloud 相关的凭证信息；

{{< admonition note Note>}}
可以看到，Crossplane 资源的部署模型完全参考了 CRD 的方式。

不同的是，CR 一般由自行编写的 Operator 程序来进行管理或部署，而在 Crossplane 中使用了声明式的 Composition 来描述如何部署 XR，从而由 Provider 来部署对应所需的资源。

可以这样认为，Composition + Provider 等同于 Operator。
{{< /admonition >}}

在 XR 的基础上，可以通过 CompositeResourceClaims 来简化字段，简称为 XRC。

## 2 Managed Resource

Managed Resource（简称 MR）指的是各个 Cloud Provider 支持的原生的云资源，一个 MR 对应与一个云厂商的资源。例如，创建一个 RDSInstance 资源实例对应了 AWS RDS 实例。

MR 通常作为构建单元用于 Composition 定义中，一般不需要直接创建 MR。

各个 MR 的定义基于云资源基本不同，可以看一下一些通用的字段：

```yaml
apiVersion: database.aws.crossplane.io/v1beta1
kind: RDSInstance
metadata:
  name: rdspostgresql
spec:
  forProvider:
    region: us-east-1
    dbInstanceClass: db.t2.small
    masterUsername: masteruser
    allocatedStorage: 20
    engine: postgres
    engineVersion: "12"
    skipFinalSnapshotBeforeDeletion: true
  writeConnectionSecretToRef:
    namespace: crossplane-system
    name: aws-rdspostgresql-conn
```

* writeConnectionSecretToRef - Secret 资源的引用，其中包含了如何连接该云资源；
  
* providerConfigRef - ProviderConfig 资源的应用，其包含了访问云服务的凭证信息，默认会使用 `default`；

* deletionPolicy - 删除策略，指定删除 Kubernetes 资源时，是否需要删除对应的云资源，支持：Delete（默认值）和 Orphan；

* forProvider - 提供给对应 Cloud Provider 的额外信息，可以认为是该云资源支持的配置信息；

### 2.1 Reconcile

不同的 Cloud Provider 会 Watch 对应的 MR，进而不断将物理云资源向着 MR 定义的期望状态 Reconcile。因此，如果手动该了云资源配置，可能会被 Provider 改回去。

一些 Iac 平台（例如 Terraform）可能会使用 Delete + ReCreate 来应用资源的修改，但是 Crossplane 不会使用这种方式。Crossplane 只会在删除 MR 并且其 `deletetionPolicy` 为 `Delete` 的情况下，才会去删除对应的云资源。

### 2.2 Name

默认下，云资源的名称与 MR 的名称相同。当然，你可以使用特殊的 annotation `crossplane.io/external-name` 来指定云资源命名：

```yaml
apiVersion: database.gcp.crossplane.io/v1beta1
kind: CloudSQLInstance
metadata:
  name: foodb
  annotations:
    crossplane.io/external-name: my-special-db
spec:
# ...
```

对于哪些不允许指定名称的资源（例如 AWS VPC），在创建云资源后，Crossplane 会将生成的云资源的名称写入该 annotation `crossplane.io/external-name`。

### 2.3 字段默认值

如果 MR 中的一些字段没有配置，那么 Crossplane 默认会配置 Cloud Proviver 提供的默认值。

例如，如果 MR 没有配置 `region` 字段，那么默认会使用 Provider 配置的默认 region。

### 2.4 删除过程

当 MR 被删除时，Provider 会立即开始删除流程。Crossplane 使用 Kubernetes Finalizer 来阻塞 MR 的删除，直到对应的云资源被删除。

如果云资源删除出现任何异常，异常信息会配置到资源的 `status` 字段。

### 2.5 依赖控制

MR 字段中有 3 个字段可用来引用另一个资源。例如下面 Azure MySQL 的配置：

```yaml
spec:
  forProvider:
    resourceGroupName: foo-res-group
    resourceGroupNameRef:
      name: resourcegroup
    resourceGroupNameSelector:
      matchLabels:
        app: prod
```

每个 MR 的字段名可能不同，但是一般都会有类似的三种方式：

* 指定依赖的云资源的名称；
* 指定依赖的云资源的 MR 名称；
* 使用 label selector 来选择依赖的资源；

当依赖另一个云资源时，对应的云资源会等待依赖的云资源 `Ready` 后才会创建。

### 2.6 使用已有的云资源

如果你已经有了一些云资源，可以创建 MR 并建立与其的管理，进而 Crossplane 会管理 MR 与云资源。

你只需要在创建 MR 时，提供 annotation `crossplane.io/external-name`，并且确认配置能够与云资源对应：

```yaml
apiVersion: compute.gcp.crossplane.io/v1beta1
kind: Network
metadata:
  name: foo-network
  annotations:
    crossplane.io/external-name: existing-network
spec:
  forProvider: {}
  providerConfigRef:
    name: default
```

当创建 MR 时，Crossplane 会发现对应的云资源已经存在，并且期望状态已经满足，那么就不会执行任何操作。

## 3 Composite Resources

Crossplane 基本的部署模型是围绕着 Composite Resource 来构建的。

官方的资源之间的联系图如下：

{{< image src="img3.png" height="400" >}}

应用层通过创建 XRC 实例来声明资源，Crossplane 根据其构建出对应的 XR 实例。根据该 XR 的 XRD 与 Composite，继续创建出对应需要的 MR。Provider 监控到 MR 被创建后，去创建对应的云资源。

### 3.1 CompositeResources 与 CompositeResourceClaims

类似于 CR，CompositeResources（简称 XR）是将 Managed Resource 进行组合，从而构建出一个更高层级的资源抽象。

```yaml
apiVersion: database.example.org/v1alpha1
kind: XPostgreSQLInstance
metadata:
  name: my-db
spec:
  parameters:
    storageGB: 20
  compositionRef:
    name: production
  writeConnectionSecretToRef:
    namespace: crossplane-system
    name: my-db-connection-details
```

对于业务层 Crossplane 提供了 CompositeResourceClaims（简称 XRC）的概念。XRC 的定义基本与 XR 相同，一般通常都是使用 XRC 来部署资源。

```yaml
apiVersion: database.example.org/v1alpha1
kind: PostgreSQLInstance
metadata:
  namespace: default
  name: my-db
spec:
  parameters:
    storageGB: 20
  compositionRef:
    name: production
  writeConnectionSecretToRef:
    name: my-db-connection-details
```

XRC 与 XR 的区别在于：

* XRC 是 Namespaced 资源，而 XR 是 Cluster Scoped 资源；
* 按照惯例，XRC 定义通常不包含 `X`；
* XRC 仅仅引用 XR，而 XR 包含对应的 XRC，也包含对应的 MR 数组；

XR 可以认为是比较基础的配置。例如下图中，VPC 资源的 XR `XNetwork` 实例是被 `XPostgreSQLInstance` 引用的。对于业务层，可以指定引用哪个 `XNetwork` 示例，但是不应该自己创建一个 `XNetwork`。因此，用户应该看到的是 `PostgreSQLInstance` 而不应该看到 `XNetwork`。

{{< image src="img2.png" height="400" >}}

可以这样认为，对于业务层 XR 是私有的不应该使用（即使能够创建），XRC 是公开的可以使用。通过这样的拆分使得权限更加清晰。

#### 3.1.1 指定 Composition

创建 XR 或者 XRC 时，可以指定使用哪个 Composition。没有指定的情况下，默认会使用 XRD 中指定 Composition。

```yaml
# ...
spec:
  compositionRef:
    name: production-us-east
  compositionSelector:
    matchLabels:
      environment: production
      region: us-east
      provider: gcp
# ...
```

* compositionSelector - 通过名称明确指定 Composition；
* compositionSelector - 通过 label selector 选择一个 Composition；

### 3.2 CompositeResourceDefinition

类似于 CRD，CompositeResourceDefinition（简称 XRD）定义了 XR 的字段并像注册该资源。

```yaml
apiVersion: apiextensions.crossplane.io/v1
kind: CompositeResourceDefinition
metadata:
  name: xpostgresqlinstances.database.example.org
spec:
  group: database.example.org
  names:
    kind: XPostgreSQLInstance
    plural: xpostgresqlinstances
  claimNames:
    kind: PostgreSQLInstance
    plural: postgresqlinstances
  versions:
  - name: v1alpha1
    served: true
    referenceable: true
    schema:
      openAPIV3Schema:
        type: object
        properties:
          spec:
            type: object
            properties:
              parameters:
                type: object
                properties:
                  storageGB:
                    type: integer
                required:
                - storageGB
            required:
            - parameters
```

一些标准的字段（例如 `writeConnectionSecretToRef` 和 `compositionRef`）不需要在 XRD 中定义，Crossplane 会自动为所有的 XR 注册这些标准字段，也称为 XRM。

完整的字段介绍见官方文档：[**CompositeResourceDefinitions**](https://docs.crossplane.io/v1.11/concepts/composition/#compositeresourcedefinitions)。

### 3.3 Composition

XRD 描述了面向使用者的 XR 的结构，而 XR 实际由哪些 MR 或 XR 组成由 Composition 定义。

以 `XPostgreSQLInstance` 为例，其 Composition 定义中指定了由 MR `CloudSQLInstance` 组成。

Composition 定义中 `resouces` 字段指定了 MR，并且其 `resouces.base` 指定了 MR 的基本模板，并且通过 patch 的方式来配置 MR 的相关变量。

```yaml
apiVersion: apiextensions.crossplane.io/v1
kind: Composition
metadata:
  name: example
  labels:
    crossplane.io/xrd: xpostgresqlinstances.database.example.org
    provider: gcp
spec:
  writeConnectionSecretsToNamespace: crossplane-system
  compositeTypeRef:
    apiVersion: database.example.org/v1alpha1
    kind: XPostgreSQLInstance
  resources:
  - name: cloudsqlinstance
    base:
      apiVersion: database.gcp.crossplane.io/v1beta1
      kind: CloudSQLInstance
      spec:
        forProvider:
          databaseVersion: POSTGRES_12
          region: us-central1
          settings:
            tier: db-custom-1-3840
            dataDiskType: PD_SSD
            ipConfiguration:
              ipv4Enabled: true
              authorizedNetworks:
                - value: "0.0.0.0/0"
    patches:
    - type: FromCompositeFieldPath
      fromFieldPath: spec.parameters.storageGB
      toFieldPath: spec.forProvider.settings.dataDiskSizeGb
```

#### Patch

可以看到，Composition 中使用 patch 的方式来将 XRC 字段值传递给 MR。Patch 支持多种方式。

* 默认使用 `FromCompositeFieldPath` 方式，从 XRC 中读取字段并覆盖 MR 中字段。

  ```yaml
  - type: FromCompositeFieldPath
    fromFieldPath: spec.parameters.size
    toFieldPath: spec.forProvider.settings.tier
  ```

* `ToCompositeFieldPath` 用于将 MR 字段暴露到 XR/XRC 中。一般用于同步 `status` 相关字段。

  ```yaml
  # Patch from the composed resource's status.atProvider.zone field to the XR's status.zone field.
  - type: ToCompositeFieldPath
    fromFieldPath: status.atProvider.zone
    toFieldPath: status.zone
  ```

* `CombineFromComposite` 读取 XR 中多个字段，进一步处理后覆盖 MR 的字段。

  ```yaml
  - type: CombineFromComposite
    combine:
      # The patch will only be applied when all variables have non-zero values.
      variables:
      - fromFieldPath: spec.parameters.location
      - fromFieldPath: metadata.annotations[crossplane.io/claim-name]
      strategy: string
      string:
        fmt: "%s-%s"
    toFieldPath: spec.forProvider.administratorLogin
    # By default Crossplane will skip the patch until all of the variables to be
    # combined have values. Set the fromFieldPath policy to 'Required' to instead
    # abort composition and return an error if a variable has no value.
    policy:
      fromFieldPath: Required
  ```

* `CombineToComposite` 将 MR 字段组合，从而暴露到 XR 中。

  ```yaml
  - type: CombineToComposite
    combine:
      variables:
        - fromFieldPath: spec.parameters.administratorLogin
        - fromFieldPath: status.atProvider.fullyQualifiedDomainName
      strategy: string
      # Here, our administratorLogin parameter and fullyQualifiedDomainName
      # status are formatted to a single output string representing a DSN.
      string:
        fmt: "mysql://%s@%s:3306/my-database-name"
    toFieldPath: status.adminDSN
  ```

* `PatchSet` 用于抽离出一个 patch 定义，以可以在多个 MR 中复用。

  ```yaml
  apiVersion: apiextensions.crossplane.io/v1
  kind: Composition
  spec:
    resources:
    - name: cloudsqlinstance
      base:
        # ...
      patches:
      - type: PatchSet
        patchSetName: metadata
    patchSets:
    - name: metadata
      patches:
      - type: FromCompositeFieldPath
        fromFieldPath: metadata.labels[some-important-label]
  ```

#### Transform

patch 下可以定义一些文本转换处理的逻辑，用于将 “from” 的值进行转换，例如字段替换等等。

例如下面示例，将 `medium` 文本值都替换为 `db-custom-1-3840`。

```yaml
apiVersion: apiextensions.crossplane.io/v1
kind: Composition
spec:
  resources:
  - name: cloudsqlinstance
    base:
      # ...
    patches:
    - type: PatchSet
      patchSetName: metadata
      transforms:
      - type: map
        map:
          medium: db-custom-1-3840
```

更多的转换方式见：[**Transform Types**](https://docs.crossplane.io/v1.11/concepts/composition/#transform-types)。

#### 传递 Connect 信息

在 Composition 定义中，通过字段 `connectionDetails` 来表明如何传递 XR 提供的 Connect Secret 信息。

可以在 XRD 定义中的 `connectionSecretKeys` 表明 Secret 中的哪些 key 可以被 Composition 定义中使用。如果 `connectionSecretKeys` 为空，那么表明所有的 key 都可以被使用。

与 patch 类似，传递 Connect 信息也支持多种方式。

* 默认为 `FromConnectionSecretKey` 方式，从 Secret 读取 key。

  例如，下面示例读取 Secret 中的 `username` 的值作为 `user` 的值：

  ```yaml
  apiVersion: apiextensions.crossplane.io/v1
  kind: Composition
  spec:
    resources:
    - name: cloudsqlinstance
    - type: MatchString
      # ...
      - type: FromConnectionSecretKey
        name: user
        fromConnectionSecretKey: username
  ```

* `FromFieldPath` 从 MR 中字段中读取信息。
  
  例如，下面示例读取 MR 字段的 `status.atProvider.adminUser` 值作为 `user` 的值：

  ```yaml
  apiVersion: apiextensions.crossplane.io/v1
  kind: Composition
  spec:
    resources:
    - name: cloudsqlinstance
      # ...
      connectionDetails:
      - type: FromFieldPath
        name: user
        fromFieldPath: status.atProvider.adminUser
  ```

* `FromValue` 设置一个固定值。
  
  ```yaml
  apiVersion: apiextensions.crossplane.io/v1
  kind: Composition
  spec:
    resources:
    - name: cloudsqlinstance
      # ...
      connectionDetails:
      - type: FromFieldPath
        name: user
        fromValue: admin
  ```

#### Readiness Check

Composition 定义中支持指定检查 MR 资源是否 Ready 的方式。

* `MatchString` 判断某个字段是否等于固定字符串
  
  ```yaml
  apiVersion: apiextensions.crossplane.io/v1
  kind: Composition
  spec:
    resources:
    - name: cloudsqlinstance
      # ...
      readinessChecks:
      - type: MatchString
        fieldPath: status.atProvider.state
        matchString: "Online"
  ```

* `MatchInteger` 判断某个字段是否等于固定数值
  
  ```yaml
  apiVersion: apiextensions.crossplane.io/v1
  kind: Composition
  spec:
    resources:
    - name: cloudsqlinstance
      # ...
      readinessChecks:
      - type: MatchInteger
        fieldPath: status.atProvider.state
        matchInteger: 4
  ```

* `NonEmpty` 判断某个字段是否等于不为空

  ```yaml
  apiVersion: apiextensions.crossplane.io/v1
  kind: Composition
  spec:
    resources:
    - name: cloudsqlinstance
      # ...
      readinessChecks:
      - type: NonEmpty
        fieldPath: status.atProvider.online
  ```

* `None` 认为资源存在就表明就绪

  ```yaml
  apiVersion: apiextensions.crossplane.io/v1
  kind: Composition
  spec:
    resources:
    - name: cloudsqlinstance
      # ...
      readinessChecks:
      - type: None
  ```

### 3.4 Connection Secret

为了能够填充 MR 的 `writeConnectionSecretToRef`，需要从 XR 或 XRC 开始将连接信息传递下来。

1. XRD 定义中包含 `connectionSecretKeys` 字段，以表明 Secret 中哪些 key 会被读取；

   ```yaml
   apiVersion: apiextensions.crossplane.io/v1
   kind: CompositeResourceDefinition
   metadata:
     name: xpostgresqlinstances.example.org
   spec:
      connectionSecretKeys:
      - hostname
      # ...
   ```

2. 创建 XR 与 XRC 时，指定 `writeConnectionSecretToRef` 字段以提供连接信息的来源。

   ```yaml
   apiVersion: database.example.org/v1alpha1
   kind: PostgreSQLInstance
   spec:
     writeConnectionSecretToRef:
       namespace: crossplane-system
       name: my-db-connection-details
     # ...
   ```

   不同的是，因为 XRC 是属于 namespace 的，所以创建的 XRC 不需要指定 namespace，而 Crossplane 会自动复制 Secret 到特定的 namespace 提供给 XR。

3. Composition 定义中必须指明如何将 XR/XRC 提供的 `writeConnectionSecretToRef` 转换为各个 MR 的 `writeConnectionSecretToRef` 字段。
   
   通过 `writeConnectionSecretsToNamespace` 字段将 XRC 的 Secret 复制一份，提供给 XR 使用。在 MR 数组中定义读取 Secret 的哪个 Key 提供给 MR。

   ```yaml
   apiVersion: apiextensions.crossplane.io/v1
   kind: Composition
   spec:
     writeConnectionSecretsToNamespace: crossplane-system
     resources:
     - name: cloudsqlinstance
       connectionDetails:
       - name: hostname
         fromConnectionSecretKey: hostname
   ```

## 4 Configuration

Configuration 是将 XRD 与 Composition 定义打包，进而将自己定义的资源以 Configuration 方式进行分发。

Configuration 包含三个文件：

* `crossplane.yaml` - 包含 Configuration 的元信息；
* `definition.yaml` - XRD 定义文件；
* `composition.yaml` - Composition 定义文件；

关于如果构建 Configuration 参考官方文档：[**Create a Configuration**](https://docs.crossplane.io/v1.11/concepts/packages/#configuration-packages)。

## 5 Provider

当创建了 Managed Resource 资源后，由各个云服务对应的 Provider 负责解析并调用 Cloud API 去创建物理的云资源。

一个 Provider 定义如下：

```yaml
apiVersion: pkg.crossplane.io/v1
kind: Provider
metadata:
  name: provider-aws
spec:
  package: "crossplane/provider-aws:master"
```

### 5.1 部署 Provider

一个 Provider 部署后就是一个 Pod，运行着指定 MR Controller。Provider 定义中的 `spec.package` 字段其实就是指定了 Provider 的镜像，也称为 Package。

与平常的镜像不同的是，Package 中包含了 Pod 部署所属的信息（例如 RBAC 和 CRD 等）。

有两种方式来部署 Provider：

* Helm 安装 Chart（本质上就是安装了 Provider 定义）。
* Crossplane CLI 安装 - 例如 `kubectl crossplane install provider crossplane/provider-aws:master`。

### 5.2 ProviderConfig

为了能够让 Provider 访问 Cloud API，需要提供给 Provider 云相关的访问凭证信息，访问凭证通过 ProviderConfig 传递。

例如，访问 AWS 所需的 `ProviderConfig` 如下，其引用了包含凭证信息的 Secret。

```yaml
apiVersion: aws.crossplane.io/v1beta1
kind: ProviderConfig
metadata:
  name: aws-provider
spec:
  credentials:
    source: Secret
    secretRef:
      namespace: crossplane-system
      name: aws-creds
      key: key
```

可以看到，ProviderConfig 可以代表着访问云服务的账号。当然，可以指定 MR 使用哪个账号来创建云资源。

```yaml
apiVersion: database.aws.crossplane.io/v1beta1
kind: RDSInstance
metadata:
  name: prod-sql
spec:
  providerConfigRef:
    name: aws-provider
  # ...
```

{{< admonition note Note>}}
通过 `providerConfigRef` 可以实现管理多账户云资源的功能。
{{< /admonition >}}

## 6 实现

### 6.1 Provider 实现

Provider 是管理各个 MR 的，也就是 CR。因此各个 Provider 就是 Controller，管理着 Crossplane 定义的 CR。

当你部署了 Provider 之后，就可以看到集群中出现的 Controller Pod：

```bash
$ k get pods -n crossplane-system
NAME                                                            READY   STATUS    RESTARTS   AGE
crossplane-contrib-provider-aws-e0b33d016bb8-74754647cf-kltps   1/1     Running   0          5m31s
```

每个 Provider 实现是使用 controller-runtime 的库实现的。但是与一般的编写方式不同，因为大多数资源都是基于相同的管理逻辑，所以 Crossplane 在 Reconciler 的基础上封装了一层。

可以看到，Provider 中每个 Manager 都基于以下接口实现：

```go
type ExternalConnectDisconnecter interface {
	ExternalConnecter
	ExternalDisconnecter
}

// ExternalConnecter 创建出一个 ExternalClient，通常是创建 cloud client
type ExternalConnecter interface {
	Connect(ctx context.Context, mg resource.Managed) (ExternalClient, error)
}

// ExternalConnecter 关闭一个 ExternalClient
type ExternalDisconnecter interface {
	Disconnect(ctx context.Context) error
}

// ExternalClient 管理某个 Managed Resource 的声明周期
type ExternalClient interface {
  // Observe 查询 Managed Resource 相关的外部资源是否存在，以及更新 Status
	Observe(ctx context.Context, mg resource.Managed) (ExternalObservation, error)

  // Create 创建 Managed Resource 资源相关的外部资源
  // Create 会在 Observe 查询外部资源不存在之后调用。Create 实现中只允许更新
  // Managed Resource 的 annotation，其他的更新会被丢弃
	Create(ctx context.Context, mg resource.Managed) (ExternalCreation, error)

  // Update 更新 Managed Resource 相关的外部资源
  // Update 会在 Observe 返回需要更新之后调用
	Update(ctx context.Context, mg resource.Managed) (ExternalUpdate, error)

  // Delete 删除 Managed Resource 相关外部资源
  // Delete 在 Managed Resource 标记为删除时调用
	Delete(ctx context.Context, mg resource.Managed) error
}
```

基本所有的 Resource Manager 都实现 `ExternalConnecter` 与 `ExternalClient` 接口，然后注册到 [**Controller Manager**](../kubebuilder/#12-manager) 中。

```go
func SetupBucket(mgr ctrl.Manager, l logging.Logger, rl workqueue.RateLimiter) error {
	name := managed.ControllerName(v1alpha3.BucketGroupKind)

  // 注册到 Controller Manager 中
	return ctrl.NewControllerManagedBy(mgr).
		Named(name).
		WithOptions(controller.Options{
			RateLimiter: ratelimiter.NewDefaultManagedRateLimiter(rl),
		}).
		For(&v1alpha3.Bucket{}).
		Complete(
      // 通过通用函数将 ExternalConnectDisconnecter 扩展为 Reconciler
			managed.NewReconciler(mgr,
				resource.ManagedKind(v1alpha3.BucketGroupVersionKind),
				managed.WithExternalConnecter(&connecter{client: mgr.GetClient()}),
				managed.WithLogger(l.WithValues("controller", name)),
				managed.WithRecorder(event.NewAPIRecorder(mgr.GetEventRecorderFor(name))),
			),
		)
}
```

看一下 Crossplane 实现的通用的 Reconciler 大致逻辑：

```go
type Reconciler struct {
	client     client.Client            // kubernetes client
	newManaged func() resource.Managed  // factory to create a managed resource

  external mrExternal                 // ExternalConnectDisconnecter
	managed  mrManaged
  // ...
}

// Reconcile a managed resource with an external resource.
func (r *Reconciler) Reconcile(ctx context.Context, req reconcile.Request) (reconcile.Result, error) {
	// 读取 MR CR
  managed := r.newManaged()
	if err := r.client.Get(ctx, req.NamespacedName, managed); err != nil {
		return requeue
	}

  // MR 标记为删除，并且删除策略是 Orphan，那么仅仅移除 CR Finalizer
	if meta.WasDeleted(managed) && managed.GetDeletionPolicy() == xpv1.DeletionOrphan {
		// ...
		if err := r.managed.RemoveFinalizer(ctx, managed); err != nil {
			return requeue
		}
		return norequeue
	}

  // 对 MR 做一些初始的信息填充
  // 例如设置 external name
	if err := r.managed.Initialize(ctx, managed); err != nil {
		return requeue
	}

  // 通过 MR Anno 发现创建操作处于中间态，那么不进行任何操作了
	if meta.ExternalCreateIncomplete(managed) {
		return requeue
	}

	// 解析 MR 相关依赖（填充一些字段）
  // 例如使用 spec.networkRef.name 填充 spec.network
	if !meta.WasDeleted(managed) {
		if err := r.managed.ResolveReferences(ctx, managed); err != nil {
			return requeue
		}
	}

  // 构建 ExternalClient
	external, err := r.external.Connect(externalCtx, managed)
	if err != nil {
		return requeue
	}
	defer func() {
		if err := r.external.Disconnect(ctx); err != nil {
		}
	}()

  // 执行 Observe 查询 MR 相关资源情况
	observation, err := external.Observe(externalCtx, managed)
	if err != nil {
		return requeue
	}

  // 查询结果为相关资源不存在，但是 MR Anno 表明之前创建成功，那么检查是否满足 Create 调用周期
  // 这样是因为防止一些外部资源正在创建中，但是查询可能会报告资源不存在
	if !observation.ResourceExists && meta.ExternalCreateSucceededDuring(managed, r.creationGracePeriod) {
		return requeue
	}

  // MR 标记为删除，那么执行 Delete 操作
	if meta.WasDeleted(managed) {
		if observation.ResourceExists {
			if err := external.Delete(externalCtx, managed); err != nil {
				return requeue
			}

			return requeue
		}

    // 外部资源已经删除，那么移除 MR Finalizer
		if err := r.managed.RemoveFinalizer(ctx, managed); err != nil {
			managed.SetConditions(xpv1.Deleting(), xpv1.ReconcileError(err))
			return requeue
		}

    // 完全删除，不再需要 requeue
		return norequeue
	}


  // MR 添加 Finalizer
	if err := r.managed.AddFinalizer(ctx, managed); err != nil {
		return requeue
	}

  // 外部资源不存在，走 Create/Update 逻辑
	if !observation.ResourceExists {
    // MR 打上 Pending Anno，用于判断 Create 操作是否执行完
		meta.SetExternalCreatePending(managed, time.Now())

    // 执行 Create 操作
		creation, err := external.Create(externalCtx, managed)
		if err != nil {
      // Create 失败，打上 CreateFailed Anno
			meta.SetExternalCreateFailed(managed, time.Now())
			return requeue
		}

    // Create 成功，MR 打上 CreateSucceeded Anno
		meta.SetExternalCreateSucceeded(managed, time.Now())
		return requeue
	}

  // 外部资源不需要更新，直接返回
	if observation.ResourceUpToDate {
		return requeue
	}

  // 外部资源需要更新，执行 Update 操作
	update, err := external.Update(externalCtx, managed)
	if err != nil {
		return requeue
	}
	return requeue
```

可以看到，其抽象了一个 Reconciler 的最基本的逻辑，基于目前 MR 的期望状态，执行 Create/Update 操作。

下面以 GCP CloudSQL Manager 为例，看一下 Resource Manager 是怎么实现的。

`cloudsqlConnector` 实现了 `ExternalConnecter`，读取凭证信息构建好 GCP Client，返回一个 `ExternalClient`：

```go
type cloudsqlConnector struct {
	kube client.Client
}

func (c *cloudsqlConnector) Connect(ctx context.Context, mg resource.Managed) (managed.ExternalClient, error) {
  // 读取 ProviderConfig 以及相关的凭证信息
	projectID, opts, err := gcp.GetAuthInfo(ctx, c.kube, mg)
	if err != nil {
		return nil, err
	}
  // 构建 GCP Client
	s, err := sqladmin.NewService(ctx, opts)
	if err != nil {
		return nil, errors.Wrap(err, errNewClient)
	}
	return &cloudsqlExternal{kube: c.kube, db: s.Instances, projectID: projectID}, nil
}
```

`cloudsqlExternal` 实现了 `ExternalClient`，通过 GCP Client 管理 CloudSQL：

```go
type cloudsqlExternal struct {
	kube      client.Client
	db        *sqladmin.InstancesService
	projectID string
}

func (c *cloudsqlExternal) Observe(ctx context.Context, mg resource.Managed) (managed.ExternalObservation, error) {
	cr, ok := mg.(*v1beta1.CloudSQLInstance)
	if !ok {
		return managed.ExternalObservation{}, errors.New(errNotCloudSQL)
	}

  // 查看 DB 是否存在
	instance, err := c.db.Get(c.projectID, meta.GetExternalName(cr)).Context(ctx).Do()
	if err != nil {
		return managed.ExternalObservation{}, errors.Wrap(resource.Ignore(gcp.IsErrorNotFound, err), errGetFailed)
	}
	
  // 更新 MR 中的状态
	cr.Status.AtProvider = cloudsql.GenerateObservation(*instance)
	switch cr.Status.AtProvider.State {
	case v1beta1.StateRunnable:
		cr.Status.SetConditions(xpv1.Available())
	case v1beta1.StateCreating:
		cr.Status.SetConditions(xpv1.Creating())
	case v1beta1.StateCreationFailed, v1beta1.StateSuspended, v1beta1.StateMaintenance, v1beta1.StateUnknownState:
		cr.Status.SetConditions(xpv1.Unavailable())
	}

  // 对于 MR Spec 与实际的 DB 状态，判断是否需要执行 Update
	upToDate, err := cloudsql.IsUpToDate(meta.GetExternalName(cr), &cr.Spec.ForProvider, instance)
	if err != nil {
		return managed.ExternalObservation{}, errors.Wrap(err, errCheckUpToDate)
	}
	return managed.ExternalObservation{
		ResourceExists:    true,
		ResourceUpToDate:  upToDate,
		ConnectionDetails: getConnectionDetails(cr, instance),
	}, nil
}

func (c *cloudsqlExternal) Create(ctx context.Context, mg resource.Managed) (managed.ExternalCreation, error) {
	cr, ok := mg.(*v1beta1.CloudSQLInstance)
	if !ok {
		return managed.ExternalCreation{}, errors.New(errNotCloudSQL)
	}
	
  // 根据 MR Spec 执行 Create
	instance := &sqladmin.DatabaseInstance{}
	cloudsql.GenerateDatabaseInstance(meta.GetExternalName(cr), cr.Spec.ForProvider, instance)

  // GCP Client 执行创建
	if _, err := c.db.Insert(c.projectID, instance).Context(ctx).Do(); err != nil {
		if gcp.IsErrorAlreadyExists(err) {
			return managed.ExternalCreation{}, errors.Wrap(err, errNameInUse)
		}
		return managed.ExternalCreation{}, errors.Wrap(err, errCreateFailed)
	}

	cd := managed.ConnectionDetails{
		xpv1.ResourceCredentialsSecretPasswordKey: []byte(pw),
	}
	return managed.ExternalCreation{ConnectionDetails: cd}, nil
}

func (c *cloudsqlExternal) Update(ctx context.Context, mg resource.Managed) (managed.ExternalUpdate, error) {
	cr, ok := mg.(*v1beta1.CloudSQLInstance)
	if !ok {
		return managed.ExternalUpdate{}, errors.New(errNotCloudSQL)
	}
	if cr.Status.AtProvider.State == v1beta1.StateCreating {
		return managed.ExternalUpdate{}, nil
	}
  // 根据 MR Spec 执行 Patch 请求
	instance := &sqladmin.DatabaseInstance{}
	cloudsql.GenerateDatabaseInstance(meta.GetExternalName(cr), cr.Spec.ForProvider, instance)
	_, err := c.db.Patch(c.projectID, meta.GetExternalName(cr), instance).Context(ctx).Do()
	return managed.ExternalUpdate{}, errors.Wrap(err, errUpdateFailed)
}

func (c *cloudsqlExternal) Delete(ctx context.Context, mg resource.Managed) error {
	cr, ok := mg.(*v1beta1.CloudSQLInstance)
	if !ok {
		return errors.New(errNotCloudSQL)
	}
	cr.SetConditions(xpv1.Deleting())
	_, err := c.db.Delete(c.projectID, meta.GetExternalName(cr)).Context(ctx).Do()
	if gcp.IsErrorNotFound(err) {
		return nil
	}
	return errors.Wrap(err, errDeleteFailed)
}
```

## 参考

* 官方文档：[**Document**](https://crossplane.io/docs/master/)
