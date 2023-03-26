# K8s 编程 - 6 - Custom APIServer


> **Kubernetes 编程系列** 主要记录一些开发 Controller 所相关的知识，大部分内容来自于[《Programming Kubernetes》](https://www.oreilly.com/library/view/programming-kubernetes/9781492047094/)（推荐直接阅读）。

Custom APIServer 可以作为 CRD 的替代品，它可以像 Kubernetes 的原生的 APIServer 一样为 API Group 提供服务。

> 下面代码都来自于 [*kubernetes/sample-apiserver*](https://github.com/kubernetes/sample-apiserver)，这是官方的 Custom APIServer 的示例

## 1 Custom APISever 的适用场景

自定义 APIServer 以一定开发与使用复杂性的代价，而做到十分灵活的功能。我们通过对比 CRD 与 APIServer 来看一下其特有的好处：

CRD 的缺陷：

* 只能使用 Kubernetes 使用的存储方式（默认 etcd）
* 不支持 protobuf，只支持 JSON
* 仅仅支持 `/status` 和 `/scale` 两种子资源
* 不支持平滑删除，你需要靠 Finalizer 来模拟这个行为
* 显著增加了 Kubernetes APIServer 的 CPU 负载，因为所有算法都用一个通用方式实现
* 对 API HTTP Endpoint 仅仅实现了标准的 CRUD 语义，不能扩展
* 不支持资源共栖（即不同 APIGroup 的资源或不同名字的资源在底层共栖存储）

APIServer 可以实现：

* **可以使用任何存储作为后端存储**
* **可以实现 Protobuf 支持**
* **可以提供任意自定义的子资源**，例如 `/exec`、`/logs`、`/port-forward` 等
* **可以实现平滑删除逻辑**
* **可以用 Go 高效的实现所有操作，包括验证、准入和转换**
* **可以自定义语义**，即定义 CRUD 之外的 API
* **可以为相同存储机制的不同 APIGroup 或不同名称资源提供服务**，例如 Deployment 最初存储在 `extension/v1` 组，后来挪到了 `apps/v1`

## 2 架构

自定义 APIServer 是为 APIGroup 提供服务的进程，通常会使用 [**k8s.io/apiserver**](https://github.com/kubernetes/apiserver) 这个库来实现。

自定义 APIServer 可以在 Kubernetes 集群内运行，也可以在集群外运行。**通常它们运行在 Pod 中，并通过 Service 提供服务**。

Kubernetes 原生的 APIServer 称为 **`kube-apiserver`**。**当 client 请求自定义 APIServer 时，请求首先会访问 kube-apiserver，然后由其转发给自定义 APIServer**。这也代表了，kube-apiserver 知道所有的自定义 APIServer。

在 kube-apiserver 实现中，这个转发代理的组件称为 **`kube-aggregator`**，代理 API 请求的过程称为 **`API Aggregate`**。

向自定义 APIServer 发起请求的过程如下（下图中最左侧的流程）：

1. kube-apiserver 收到请求。
2. 请求经过处理链处理，包括身份认证、审计日志、切换用户、限流、授权等流程。
3. kube-apiserver 对相关 `/apis/<aggregate-API-group-name>` HTTP Path 下的请求进行拦截。
4. 将拦截下来的请求转发给自定义 APIServer。

{{< image src="img1.png" height=300 >}}

总结一下，kube-aggregator 提供两种功能：

* **Proxy** - 可以为某个 HTTP Path 下的一个特定版本提供代理服务，例如 **`/apis/group-name/version`**

* **Discovery** - kube-aggregator 可以为所有聚合的 APIServer 提供 Discovery Endpoint，也是就说，你可以通过 **`/apis`** 和 **`/apis/group-name`** 找到所有的自定义 APIServer。

### 2.1 APIService

如同 CRD 一样，为了让 Kubernetes APIServer 知道一个自定义 APIServer 的存在，必须创建一个 **`APIService`** 对象。

```yaml
apiVersion: apiregistration.k8s.io/v1
kind: APIService 
metadata:  
  name: name 
spec: 
  group: API-group-name 
  version: API-group-version 
  service: 
    namespace: custom-API-server-service-namespace 
    name: API-server-service
    port: 1234
  caBundle: base64-caBundle
  groupPriorityMinimum: 2000 
  versionPriority: 20 
```

* `spec.service` - 指定自定义 APIServer 的 service，可以是集群内普通的 ClusterIP Service，也可以是外部的 ExternalName Service。

* `spec.caBundle` - 用于设置访问自定义 APIServer 所需要的证书。

* `spec.groupPriorityMinimum` - Group 的优先级，如果多个不同版本 APIService 设置了不同的值，值最大的会实际生效。
  
  如果存在冲突的资源名称或者短名称，则拥有最大 groupPriorityMinimum 值的资源会生效。

* `spec.versionPriority` - 给 Group 下各个版本排序，用于决定客户端优先使用哪个版本。

下面是 Kubernetes API 组的 groupPriorityMinimum 值：

```go
var apiVersionPriorities = map[schema.GroupVersion]priority{
	{Group: "", Version: "v1"}: {group: 18000, version: 1},
	// to my knowledge, nothing below here collides
	{Group: "apps", Version: "v1"}:                               {group: 17800, version: 15},
	{Group: "events.k8s.io", Version: "v1"}:                      {group: 17750, version: 15},
	{Group: "events.k8s.io", Version: "v1beta1"}:                 {group: 17750, version: 5},
	{Group: "authentication.k8s.io", Version: "v1"}:              {group: 17700, version: 15},
	{Group: "authorization.k8s.io", Version: "v1"}:               {group: 17600, version: 15},
	{Group: "autoscaling", Version: "v1"}:                        {group: 17500, version: 15},
	{Group: "autoscaling", Version: "v2beta1"}:                   {group: 17500, version: 9},
	{Group: "autoscaling", Version: "v2beta2"}:                   {group: 17500, version: 1},
	{Group: "batch", Version: "v1"}:                              {group: 17400, version: 15},
	{Group: "batch", Version: "v1beta1"}:                         {group: 17400, version: 9},
	{Group: "batch", Version: "v2alpha1"}:                        {group: 17400, version: 9},
	{Group: "certificates.k8s.io", Version: "v1"}:                {group: 17300, version: 15},
	{Group: "networking.k8s.io", Version: "v1"}:                  {group: 17200, version: 15},
	{Group: "policy", Version: "v1"}:                             {group: 17100, version: 15},
	{Group: "policy", Version: "v1beta1"}:                        {group: 17100, version: 9},
	{Group: "rbac.authorization.k8s.io", Version: "v1"}:          {group: 17000, version: 15},
	{Group: "storage.k8s.io", Version: "v1"}:                     {group: 16800, version: 15},
	{Group: "storage.k8s.io", Version: "v1beta1"}:                {group: 16800, version: 9},
	{Group: "storage.k8s.io", Version: "v1alpha1"}:               {group: 16800, version: 1},
	{Group: "apiextensions.k8s.io", Version: "v1"}:               {group: 16700, version: 15},
	{Group: "admissionregistration.k8s.io", Version: "v1"}:       {group: 16700, version: 15},
	{Group: "scheduling.k8s.io", Version: "v1"}:                  {group: 16600, version: 15},
	{Group: "coordination.k8s.io", Version: "v1"}:                {group: 16500, version: 15},
	{Group: "node.k8s.io", Version: "v1"}:                        {group: 16300, version: 15},
	{Group: "node.k8s.io", Version: "v1alpha1"}:                  {group: 16300, version: 1},
	{Group: "node.k8s.io", Version: "v1beta1"}:                   {group: 16300, version: 9},
	{Group: "discovery.k8s.io", Version: "v1"}:                   {group: 16200, version: 15},
	{Group: "discovery.k8s.io", Version: "v1beta1"}:              {group: 16200, version: 12},
	{Group: "flowcontrol.apiserver.k8s.io", Version: "v1beta1"}:  {group: 16100, version: 12},
	{Group: "flowcontrol.apiserver.k8s.io", Version: "v1alpha1"}: {group: 16100, version: 9},
	{Group: "internal.apiserver.k8s.io", Version: "v1alpha1"}:    {group: 16000, version: 9},
	// Append a new group to the end of the list if unsure.
	// You can use min(existing group)-100 as the initial value for a group.
	// Version can be set to 9 (to have space around) for a new group.
}
```

{{< admonition tip "替换原生的 API">}}
**如果你替换原生的 API Group，就可以对自定义 APIService 指定为比上表中更低的优先级实现。**
{{< /admonition >}}

### 2.2 Custom APIServer 的内部架构

Custom APIServer 大体上与 Kubernetes APIServer 相同，不过没有嵌入 kube-aggregator 和 apiextension-apiserver。

{{< image src="img2.png" height=400 >}}

总结一下自定义 APIServer 的特点：

* **与 Kubernetes APIServer 内部结构一致**。
* **拥有自己的处理链，包括身份认证、审计、切换用户等**
* **拥有自己的资源处理流水线**，包括解码、转换、准入、REST 映射和编码。
* **会调用 Webhook**。
* **可以写 etcd，或者其他的后端存储**。
* **拥有自己的 Scheme 并实现了自定义 API 组的 Registry**。
* **再次进行身份认证**。通常会发送一个 TokenAccessReview 请求对 Kubernetes APIServer 回调，实现基于 client 的证书认证和 token 认证。
* **自己进行审计**。
* 使用 SubjectAccessReview 请求 Kubernetes APIServer 完成授权。

### 2.3 身份认证机制

Custom APIServer 的处理在于 Kubernetes APIServer 之后，所以 **Kubernetes APIServer 会先对请求进行认证，通过后再转发给自定义 APIServer**。

Kubernetes APIServer 会将身份认证的结果保存在 HTTP 请求头里，通常是 **`X-Remote-User`** 和 **`X-Remote-Group Head`**。

{{< admonition note Note>}}
通过 APIServer 的启动命令参数 **`--requestheader-username-headers`** 和 **`--requestheader-group-headers`** 可以配置。
{{< /admonition >}}

为了让 Custom APIServer 信任这两个 HTTP Head，Custom APIServer 会验证发送请求者的 CA。因此，需要**通过一个 ConfigMap 配置 Kubernetes APIServer 的证书**，证书要与 Custom APIServer 相同的根证书签名：

```yaml
apiVersion: v1 
kind: ConfigMap 
metadata: 
  name: extension-apiserver-authentication 
  namespace: kube-system 
data: 
  client-ca-file: | 
    -----BEGIN CERTIFICATE----- 
    ... 
    -----END CERTIFICATE----- 
  requestheader-allowed-names: '["aggregator"]' 
  requestheader-client-ca-file: | 
    -----BEGIN CERTIFICATE----- 
    ... 
    -----END CERTIFICATE----- 
  requestheader-extra-headers-prefix: '["X-Remote-Extra-"]' 
  requestheader-group-headers: '["X-Remote-Group"]' 
  requestheader-username-headers: '["X-Remote-User"]'
```

这样认证的过程如下：

1. Kubernetes APIServer 就会使用 `data.client-ca-file` 指定的客户端证书发起请求，进行 “预认证”
2. 预认证后需要通过 `data.requestheader-client-ca-file` 证书发起转发请求，同时设置好相关的 Head。
3. 最后，Custom APIServer 会通过 **`TokenAccessReview`** 的机制发送 Bearer Token（通过 HTTP Head：Authorization: bearer token）给 Kubernetes APIServer 来验证是否合法。

{{< admonition note Note>}}
上面这些过程主要是由 `k8s.io/apiserver` 库自动完成，我们只需要配置好证书。
{{< /admonition >}}

### 2.4 授权

完成身份认证后，每个请求都需要授权，Kubernetes 的授权机制是基于 RBAC 来完成的。

RBAC 将身份映射到角色，再将角色映射到授权规则，授权规则最终决定接受或者拒绝请求。我们需要理解，自定义 APIServer 是通过 **`SubjectAccessReview`** 代理授权来对请求授权的，**它自身不会处理 RBAC 规则，而是委托给 Kubernetes APIServer 完成的**。

{{< admonition note "为了什么需要 APIServer 授权">}}
由于可能发送给 Custom APIServer 的请求是没有经过授权的，所以需要通过 Kubernetes APIServer 来进行授权。
{{< /admonition >}}

自定义 APIServer 会发送一个 SubjectAccessReview 请求到 Kubernetes APIServer。

```yaml
apiVersion: authorization.k8s.io/v1 
kind: SubjectAccessReview 
spec: 
  resourceAttributes: 
    group: apps 
    resource: deployments 
    verb: create 
    namespace: default 
    version: v1 
    name: example 
  user: michael 
  groups: 
  - system:authenticated 
  - admins 
  - authors
```

Kubernetes APIServer 收到请求后，将基于 RBAC 规则进行判定，结果还是返回一个 SubjectAccessReview 对象。其中，会设置关键的 status 字段。

```yaml
apiVersion: authorization.k8s.io/v1 
kind: SubjectAccessReview 
status: 
  allowed: true 
  denied: false 
  reason: "rule foo allowed this request"
```

* status.allowed - 表明是否允许
* status.denied - 表明是否拒绝请求 

{{< admonition note Note>}}
allowed 与 denied 可能同时为 false，这表明 Kubernetes APIServer 无法对请求做出判断。这应该依靠自定义 APIServer 里的权限逻辑进行判断。
{{< /admonition >}}

注意，处于性能方面考虑，**委托授权机制在每个 Custom APIServer 中都维护了一个本地缓存**：

* 默认缓存 1024 个授权条目。
* 所有通过的授权请求缓存过期时间为 5min。
* 所有拒绝的授权请求缓存过期时间为 30s。

{{< admonition note Note>}}
可以通过 **`--authorization-webhook-cache-authorized-ttl`** 和 **`--authorization-webhook-cache-unauthorized-ttl`** 来进行配置。
{{< /admonition >}}


## 3 多版本类型实现

### 3.1 处理流程

每个 APIServer 都可以提供多个资源和版本的接口。为了让一个资源的多版本共存称为可能，**APIServer 需要将资源在多个版本之间进行转换**。

{{< image src="img3.png" height=300 >}}

APIServer 在真正实现 API 逻辑时，会使用一个 [内部版本]^(internal version)，也称为 [中枢版本]^(hub version)。它可以用作每个版本都可以与之转换的中间版本，所以内部 API 逻辑都是根据中枢版本实现的。

所以，APIServer 中存在三个版本的概念：

* **`External Version`** - 能够处理的版本，例如 v1beta1 v1 等；
* **`Internal Version`** - 用于 External Version 之间转换的中间版本；
* **`Storage Version`** - 保存到后端存储（etcd）的版本，属于 External Version 其中一个（通过 CRD 定义设置）；

下图展示了 APIServer 在一个 API 请求的生命周期中，三个版本之间的转换情况：

{{< image src="img4.png" height=400 >}}

1. 用户发送一个特定版本的请求（例如 v1，但是可以是任何支持的版本）。**[External Version]**
2. APIServer 将请求解码，并转换为内部版本。**[External Version -> Internal Version]**
3. APIServer 对内部版本进行准入检测与验证。**[Internal Version]**
4. 注册表中的 API 逻辑都是使用内部版本的。**[Internal Version]**
5. etcd 读写或写入特定版本对象（例如使用 v2 作为存储版本），这个过程中，它需要与内部版本进行互转。**[Internal Version -> Storage Version]**
6. 结果转换为请求中的版本，例如 v1。**[Internal Version -> External Version]**

从上图可以看到，一次写请求操作中，至少要做三次转换（步骤 2 5 6）。如果部署了 admission webhook，还会发生更多次的转换。

除了转换，下图中展示了默认值会何时处理，Default 填充未设置值字段的过程。默认值处理总是和转换一起出现，并且总是在用户请求、etcd 或者 admission webhook 时的 External Version 进行（也就是只在**External Version 转换到 Internal Version 时进行**），不会发生在中枢版本向外部版本转换过程中。

{{< image src="img5.png" height=400 >}}


{{< admonition note Note>}}
可以看到，转换是至关重要的，所有的两个方向的转换都必须是正确的，即是 [双程的]^(roundtrippable)。roundtrippable 意味着我们可以对任意的值在整个版本中来回转换，而不丢弃任何信息。
{{< /admonition >}}

### 3.2 Internal Version 类型实现

我们看下 Internal Version 实现，位于 **`pkg/apis/<group>/types.go`** 文件中：

```go
// +genclient
// +k8s:deepcopy-gen:interfaces=k8s.io/apimachinery/pkg/runtime.Object

type Flunder struct {
	metav1.TypeMeta
	metav1.ObjectMeta

	Spec   FlunderSpec
	Status FlunderStatus
}

// +genclient
// +genclient:nonNamespaced
// +k8s:deepcopy-gen:interfaces=k8s.io/apimachinery/pkg/runtime.Object

// Fischer is an example type with a list of disallowed Flunder.Names
type Fischer struct {
	metav1.TypeMeta
	metav1.ObjectMeta

	// DisallowedFlunders holds a list of Flunder.Names that are disallowed.
	DisallowedFlunders []string
}

// ...
```

{{< admonition note 没有标签>}}
注意，**Internal Version 类型是没有 JSON 和 protobuf 标签的**。因为 JSON 标签会被一些生成器用于探测一个 types.go 文件是对应内部版本还是外部版本，所以不能包含标签。
{{< /admonition >}}

在将 Internal Version 类型注册到 Scheme 时，我们可以看到其注册的 GroupVersion 是一个特殊的 `runtime.APIVersionInternal`：

```go
var SchemeGroupVersion = schema.GroupVersion{Group: GroupName, Version: runtime.APIVersionInternal}

// Adds the list of known types to the given scheme.
func addKnownTypes(scheme *runtime.Scheme) error {
	scheme.AddKnownTypes(SchemeGroupVersion,
		&Flunder{},
		&FlunderList{},
		&Fischer{},
		&FischerList{},
	)
	return nil
}
```

除了基本的类型与对应的 DeepCopy 相关的实现，Internal Version 类型不需要更多的代码实现了。Convert 与 Default 相关都是针对特定的 External Version 类型需要实现的。

### 3.3 External Version 类型实现

在 sample-apiserver 中实现了 `v1alpha1` `v1beta1` 两个 External Version。我们以 v1alpha1 版本主要说明。

一个 External Version 类型放置于 **`pkg/apis/<group>/<version>/types.go`** 文件中。当然，其是必须包含 JSON 标签的。

```go
// +genclient
// +k8s:deepcopy-gen:interfaces=k8s.io/apimachinery/pkg/runtime.Object
type Flunder struct {
	metav1.TypeMeta   `json:",inline"`
	metav1.ObjectMeta `json:"metadata,omitempty" protobuf:"bytes,1,opt,name=metadata"`

	Spec   FlunderSpec   `json:"spec,omitempty" protobuf:"bytes,2,opt,name=spec"`
	Status FlunderStatus `json:"status,omitempty" protobuf:"bytes,3,opt,name=status"`
}
// +genclient
// +genclient:nonNamespaced
// +k8s:deepcopy-gen:interfaces=k8s.io/apimachinery/pkg/runtime.Object

type Fischer struct {
	metav1.TypeMeta   `json:",inline"`
	metav1.ObjectMeta `json:"metadata,omitempty" protobuf:"bytes,1,opt,name=metadata"`

	// DisallowedFlunders holds a list of Flunder.Names that are disallowed.
	DisallowedFlunders []string `json:"disallowedFlunders,omitempty" protobuf:"bytes,2,rep,name=disallowedFlunders"`
}
// ...
```

与 Internal Version 不同了，其注册到 Scheme 的 GroupVersion 就是对应版本的 string 了。

```go
var SchemeGroupVersion = schema.GroupVersion{Group: GroupName, Version: "v1alpha1"}

func addKnownTypes(scheme *runtime.Scheme) error {
	scheme.AddKnownTypes(SchemeGroupVersion,
		&Flunder{},
		&FlunderList{},
		&Fischer{},
		&FischerList{},
	)
	metav1.AddToGroupVersion(scheme, SchemeGroupVersion)
	return nil
}
```

#### 3.3.1 Convert

在 [**3.1 处理流程**](#31-处理流程) 中看到，APIServer 处理请求过程中有着多次 External Version 与 Internal Version 双向的转换，因此其需要注册对应的转换函数，其注册函数会自动生成位于 **`pkg/apis/\<group>/\<version>/zz_generated.conversion.go`**。

可以看到，其类型转换函数通过 `Scheme.AddGeneratedConversionFunc()` 接口注册到 Scheme 中。而在 APIServer 抽象实现中，会按照 [**3.1 处理流程**](#31-处理流程) 中的请求处理流程调用 `Scheme.Convert()` 接口进行转换。

```go
func init() {
	// 注册转换函数
	localSchemeBuilder.Register(RegisterConversions)
}

// RegisterConversions adds conversion functions to the given scheme.
// Public to allow building arbitrary schemes.
func RegisterConversions(s *runtime.Scheme) error {
	// Scheme.AddGeneratedConversionFunc(a, b interface{}, fn conversion.ConversionFunc) 用于注册一个转换函数
	// 表明 fn 会用于将 a -> b
	//
	// fn 为 type ConversionFunc func(a, b interface{}, scope Scope) error
	// 实现实际的将 a -> b，scope 表示范围（大部分情况不使用）

	if err := s.AddGeneratedConversionFunc((*Fischer)(nil), (*wardle.Fischer)(nil), func(a, b interface{}, scope conversion.Scope) error {
		return Convert_v1alpha1_Fischer_To_wardle_Fischer(a.(*Fischer), b.(*wardle.Fischer), scope)
	}); err != nil {
		return err
	}
	if err := s.AddGeneratedConversionFunc((*wardle.Fischer)(nil), (*Fischer)(nil), func(a, b interface{}, scope conversion.Scope) error {
		return Convert_wardle_Fischer_To_v1alpha1_Fischer(a.(*wardle.Fischer), b.(*Fischer), scope)
	}); err != nil {
		return err
	}
	if err := s.AddGeneratedConversionFunc((*FischerList)(nil), (*wardle.FischerList)(nil), func(a, b interface{}, scope conversion.Scope) error {
		return Convert_v1alpha1_FischerList_To_wardle_FischerList(a.(*FischerList), b.(*wardle.FischerList), scope)
	}); err != nil {
		return err
	}
	if err := s.AddGeneratedConversionFunc((*wardle.FischerList)(nil), (*FischerList)(nil), func(a, b interface{}, scope conversion.Scope) error {
		return Convert_wardle_FischerList_To_v1alpha1_FischerList(a.(*wardle.FischerList), b.(*FischerList), scope)
	}); err != nil {
		return err
	}
	if err := s.AddGeneratedConversionFunc((*Flunder)(nil), (*wardle.Flunder)(nil), func(a, b interface{}, scope conversion.Scope) error {
		return Convert_v1alpha1_Flunder_To_wardle_Flunder(a.(*Flunder), b.(*wardle.Flunder), scope)
	}); err != nil {
		return err
	}
	if err := s.AddGeneratedConversionFunc((*wardle.Flunder)(nil), (*Flunder)(nil), func(a, b interface{}, scope conversion.Scope) error {
		return Convert_wardle_Flunder_To_v1alpha1_Flunder(a.(*wardle.Flunder), b.(*Flunder), scope)
	}); err != nil {
		return err
	}
	if err := s.AddGeneratedConversionFunc((*FlunderList)(nil), (*wardle.FlunderList)(nil), func(a, b interface{}, scope conversion.Scope) error {
		return Convert_v1alpha1_FlunderList_To_wardle_FlunderList(a.(*FlunderList), b.(*wardle.FlunderList), scope)
	}); err != nil {
		return err
	}
	if err := s.AddGeneratedConversionFunc((*wardle.FlunderList)(nil), (*FlunderList)(nil), func(a, b interface{}, scope conversion.Scope) error {
		return Convert_wardle_FlunderList_To_v1alpha1_FlunderList(a.(*wardle.FlunderList), b.(*FlunderList), scope)
	}); err != nil {
		return err
	}
	if err := s.AddGeneratedConversionFunc((*FlunderStatus)(nil), (*wardle.FlunderStatus)(nil), func(a, b interface{}, scope conversion.Scope) error {
		return Convert_v1alpha1_FlunderStatus_To_wardle_FlunderStatus(a.(*FlunderStatus), b.(*wardle.FlunderStatus), scope)
	}); err != nil {
		return err
	}
	if err := s.AddGeneratedConversionFunc((*wardle.FlunderStatus)(nil), (*FlunderStatus)(nil), func(a, b interface{}, scope conversion.Scope) error {
		return Convert_wardle_FlunderStatus_To_v1alpha1_FlunderStatus(a.(*wardle.FlunderStatus), b.(*FlunderStatus), scope)
	}); err != nil {
		return err
	}
	if err := s.AddConversionFunc((*FlunderSpec)(nil), (*wardle.FlunderSpec)(nil), func(a, b interface{}, scope conversion.Scope) error {
		return Convert_v1alpha1_FlunderSpec_To_wardle_FlunderSpec(a.(*FlunderSpec), b.(*wardle.FlunderSpec), scope)
	}); err != nil {
		return err
	}
	if err := s.AddConversionFunc((*wardle.FlunderSpec)(nil), (*FlunderSpec)(nil), func(a, b interface{}, scope conversion.Scope) error {
		return Convert_wardle_FlunderSpec_To_v1alpha1_FlunderSpec(a.(*wardle.FlunderSpec), b.(*FlunderSpec), scope)
	}); err != nil {
		return err
	}
	return nil
}
```

生成器会自动生成 `Convert_<group>_<Internal-Type>_To_<External-Version>_<<External-Type>()` 与 `Convert_<External-Version>_<External-Type>_To_<group>_<Internal-Type>` 两个双向的 Internal Version 与 External Version 转换的函数。

当然，自动生成的函数只是简单的结构体转换，即转换前后的结构体都是使用同一块内存。例如：

```go
func autoConvert_v1alpha1_FischerList_To_wardle_FischerList(in *FischerList, out *wardle.FischerList, s conversion.Scope) error {
	out.ListMeta = in.ListMeta
	out.Items = *(*[]wardle.Fischer)(unsafe.Pointer(&in.Items))
	return nil
}
```

{{< admonition note Note>}}
上面这种转换方式也说明了，在转换后，源对象应该不要再修改了，否则会影响到转换后的对象。
{{< /admonition >}}

**对于版本之间不同的结构体，就需要我们自己实现对应的转换函数了**。在 **`pkg/apis/<group>/<version>/conversion.go`** 文件中，我们按照对应的格式实现特殊的转换函数。而生成器判断到对应的转换函数已经存在，就会跳过自动生成。

```go
// 自定义的版本转换函数
func Convert_v1alpha1_FlunderSpec_To_wardle_FlunderSpec(in *FlunderSpec, out *wardle.FlunderSpec, s conversion.Scope) error {
	if in.ReferenceType != nil {
		// assume that ReferenceType is defaulted
		switch *in.ReferenceType {
		case FlunderReferenceType:
			out.ReferenceType = wardle.FlunderReferenceType
			out.FlunderReference = in.Reference
		case FischerReferenceType:
			out.ReferenceType = wardle.FischerReferenceType
			out.FischerReference = in.Reference
		}
	}

	return nil
}

// 自定义的版本转换函数
func Convert_wardle_FlunderSpec_To_v1alpha1_FlunderSpec(in *wardle.FlunderSpec, out *FlunderSpec, s conversion.Scope) error {
	switch in.ReferenceType {
	case wardle.FlunderReferenceType:
		t := FlunderReferenceType
		out.ReferenceType = &t
		out.Reference = in.FlunderReference
	case wardle.FischerReferenceType:
		t := FischerReferenceType
		out.ReferenceType = &t
		out.Reference = in.FischerReference
	}

	return nil
}
```

{{< admonition warning WARN>}}
转换的过程中，源对象一定不能被修改。
{{< /admonition >}}

#### 3.3.1 Default

除了 Convert 函数之外，External Version 类型也可以设置 Default 函数。在 [**3.1.1 处理流程**](#31-处理流程) 中提到过，APIServer 会在 External Version 转换到 Internal Version 时进行默认值处理。

同样，生成器也会自动生成 Default 相关函数，位于文件 **`pkg/apis/\<group>/\<version>/zz_generated.defaults.go`**，但是底层设置默认值的函数还是要我们自己编写。

```go
// RegisterDefaults 用于向 Scheme 注册默认值处理函数
func RegisterDefaults(scheme *runtime.Scheme) error {
	scheme.AddTypeDefaultingFunc(&Flunder{}, func(obj interface{}) { SetObjectDefaults_Flunder(obj.(*Flunder)) })
	scheme.AddTypeDefaultingFunc(&FlunderList{}, func(obj interface{}) { SetObjectDefaults_FlunderList(obj.(*FlunderList)) })
	return nil
}

func SetObjectDefaults_Flunder(in *Flunder) {
	SetDefaults_FlunderSpec(&in.Spec) // 调用真正的设置默认值的函数
}

func SetObjectDefaults_FlunderList(in *FlunderList) {
	for i := range in.Items {
		a := &in.Items[i]
		SetObjectDefaults_Flunder(a)
	}
}
```

自己编写的设置默认值的函数位于文件 **`pkg/apis/<group>/<version>/defaults.go`**:

```go
// SetDefaults_FlunderSpec sets defaults for Flunder spec
func SetDefaults_FlunderSpec(obj *FlunderSpec) {
	if (obj.ReferenceType == nil || len(*obj.ReferenceType) == 0) && len(obj.Reference) != 0 {
		t := FlunderReferenceType
		obj.ReferenceType = &t
	}
}
```

### 3.4 向 Scheme 注册

最后在说明一下类型、Convert 函数、Default 函数是如何注册到 Scheme 中的。这三类都会注册到代码生成的一个 `SchemeBuilder` 对象，其作用就是用于调用注册的 `funcs (scheme *Scheme) error` 去设置 Scheme。

而前面看到的各个 Register 函数都是注册到 `SchemeBuilder` 对象中。

```go
var (
	SchemeBuilder      runtime.SchemeBuilder
	localSchemeBuilder = &SchemeBuilder
	// AddToScheme is a common registration function for mapping packaged scoped group & version keys to a scheme
	AddToScheme = localSchemeBuilder.AddToScheme
)

func init() {
	// We only register manually written functions here. The registration of the
	// generated functions takes place in the generated files. The separation
	// makes the code compile even when the generated files are missing.
	localSchemeBuilder.Register(addKnownTypes, addDefaultingFuncs)
}

// addKnownTypes 注册类型
func addKnownTypes(scheme *runtime.Scheme) error {
	scheme.AddKnownTypes(SchemeGroupVersion,
		&Flunder{},
		&FlunderList{},
		&Fischer{},
		&FischerList{},
	)
	metav1.AddToGroupVersion(scheme, SchemeGroupVersion)
	return nil
}

// addDefaultingFuncs 注册 Default 函数
func addDefaultingFuncs(scheme *runtime.Scheme) error {
	return RegisterDefaults(scheme)
}

func init() {
	localSchemeBuilder.Register(RegisterConversions) // 注册 Convert 函数
}
```

所有版本的 AddToScheme 都是在 **`pkg/apis/<group>/install/install.go`** 中一个辅助函数 Install 中注册：

```go
// Install registers the API group and adds types to a scheme
func Install(scheme *runtime.Scheme) {
	// 注册 Internal Version
	utilruntime.Must(wardle.AddToScheme(scheme))
	// 注册 v1beta1
	utilruntime.Must(v1beta1.AddToScheme(scheme))
	// 注册 v1alpha1
	utilruntime.Must(v1alpha1.AddToScheme(scheme))
	// 设置优先级（优先级用于客户端选择版本）
	utilruntime.Must(scheme.SetVersionPriority(v1beta1.SchemeGroupVersion, v1alpha1.SchemeGroupVersion))
}
```

Install 函数在 APIServer 包中会自动调用，也就是说，程序运行一开始就将所有 Scheme 相关的注册处理好了。

```go
var (
	// Scheme defines methods for serializing and deserializing API objects.
	Scheme = runtime.NewScheme()
)

func init() {
	install.Install(Scheme)
  // ...
}
```


## 4 Registry 与 Strategy 

APIServer 本质上就是一个 HTTP Server，client 通过访问以 Resource 为单位的 RESTful 接口来进行交互。在 Custom APIServer 中，为了让 Custom APIServer 能够处理 CR 相关的接口，我们需要注册相关接口的 HTTP Handle。

`k8s.io/apiserver` 库提供了该功能的高度封装，其核心就是 Registry。

### 4.1 APIServer 对象

在 Custom APIServer 实现中，不需要自己定义 HTTP Server，可以直接内嵌提供的 `GenericAPIServer` 类。

```go
import genericapiserver "k8s.io/apiserver/pkg/server"

// WardleServer contains state for a Kubernetes cluster master/api server.
type WardleServer struct {
	GenericAPIServer *genericapiserver.GenericAPIServer
}
```

在整个 Custom APIServer 程序启动流程中，会调用 `GenericAPIServer.PrepareRun().Run()` 运行一个 HTTP Server。（整个启动流程很长，后面单独说明）

在创建 WardleServer 过程中，我们会创建 `APIGroupInfo` 对象，而将其注册到 `GenericAPIServer` 对象中，可以理解为完成了某个 Group 下所有 Resource 对应的 HTTP Handle 注册。

```go
func (c completedConfig) New() (*WardleServer, error) {
	// CompletedConfig 得到一个 GenericAPIServer
	genericServer, err := c.GenericConfig.New("sample-apiserver", genericapiserver.NewEmptyDelegate())
	if err != nil {
		return nil, err
	}

	// Custom APIServer 对象
	s := &WardleServer{
		GenericAPIServer: genericServer,
	}

	// 构建 APIGroupInfo
	apiGroupInfo := genericapiserver.NewDefaultAPIGroupInfo(wardle.GroupName, Scheme, metav1.ParameterCodec, Codecs)

	// 构建 APIGroup
	v1alpha1storage := map[string]rest.Storage{}
	v1alpha1storage["flunders"] = wardleregistry.RESTInPeace(flunderstorage.NewREST(Scheme, c.GenericConfig.RESTOptionsGetter))
	v1alpha1storage["fischers"] = wardleregistry.RESTInPeace(fischerstorage.NewREST(Scheme, c.GenericConfig.RESTOptionsGetter))
	apiGroupInfo.VersionedResourcesStorageMap["v1alpha1"] = v1alpha1storage

	v1beta1storage := map[string]rest.Storage{}
	v1beta1storage["flunders"] = wardleregistry.RESTInPeace(flunderstorage.NewREST(Scheme, c.GenericConfig.RESTOptionsGetter))
	apiGroupInfo.VersionedResourcesStorageMap["v1beta1"] = v1beta1storage

	// 注册 APIGroup 到 Kubernetes APIServer
	if err := s.GenericAPIServer.InstallAPIGroup(&apiGroupInfo); err != nil {
		return nil, err
	}

	return s, nil
}
```

{{< admonition note Note>}}
这里暂时不去深入了解 APIGroupInfo 是如何映射到 HTTP Handle 的，因为这是 `k8s.io/apiserver` 库提供的封装。（其实我也不懂）
{{< /admonition >}}

### 4.2 APIGroupInfo

**`APIGroupInfo`** 能够变为一个 Group 下所有 Version 所有 Resource 的的 HTTP Handle。因此，我们可以想到，我们提供给 APIGroupInfo 什么信息：

* 该 Group 支持的所有 Version，以及每个 Version 支持的所有 Resource；
  
  由调用 `genericapiserver.NewDefaultAPIGroupInfo()` 传入的 Group + Scheme + Codecs 提供。这样，处理 HTTP 请求时能够将数据序列化为特定的对象。
  
* GroupVersionResource 维度的 RESTful HTTP Handle；

  通过将 `rest.Storage` 注册到 `APIGroupInfo.VersionedResourcesStorageMap` 中。rest.Storage 为一个 RESTful HTTP Handle 的实现。

```go
	// 构建 APIGroupInfo
	apiGroupInfo := genericapiserver.NewDefaultAPIGroupInfo(wardle.GroupName, Scheme, metav1.ParameterCodec, Codecs)

	// 构建 APIGroup
	// 一个 rest.Storage 对应了一个 HTTP API Endpoint
	// 即 /apis/<group>/<version>/<resource>

	// 下面注册了两个 Endpoint
	//	+ /apis/wardle.example.com/v1alpha1/flunders
	//	+ /apis/wardle.example.com/v1alpha1/fischers
	v1alpha1storage := map[string]rest.Storage{}
	v1alpha1storage["flunders"] = wardleregistry.RESTInPeace(flunderstorage.NewREST(Scheme, c.GenericConfig.RESTOptionsGetter))
	v1alpha1storage["fischers"] = wardleregistry.RESTInPeace(fischerstorage.NewREST(Scheme, c.GenericConfig.RESTOptionsGetter))
	apiGroupInfo.VersionedResourcesStorageMap["v1alpha1"] = v1alpha1storage // 注册到 APIGroupInfo v1alpha1 版本

	// 下面注册了一个 Endpoint
	//	+ /apis/wardle.example.com/v1beta1/flunders
	v1beta1storage := map[string]rest.Storage{}
	v1beta1storage["flunders"] = wardleregistry.RESTInPeace(flunderstorage.NewREST(Scheme, c.GenericConfig.RESTOptionsGetter))
	apiGroupInfo.VersionedResourcesStorageMap["v1beta1"] = v1beta1storage // 注册到 APIGroupInfo v1beta1 版本
```

### 4.3 rest.Storage

终于，我们看似来到了这个封装的最底层 `rest.Storage`，提供 REST 风格各个 Handle 的类。`rest.Storage` 本身的 interface 实现很简单，仅仅只需要提供 `New()` 方法就好了，也就是说，一个 Resource HTTP Endpoint 至少要提供一个 Create/Update 接口。

```go
// Storage is a generic interface for RESTful storage services.
// Resources which are exported to the RESTful API of apiserver need to implement this interface. It is expected
// that objects may implement any of the below interfaces.
type Storage interface {
	// New returns an empty object that can be used with Create and Update after request data has been put into it.
	// This object must be a pointer type for use with Codec.DecodeInto([]byte, runtime.Object)
	New() runtime.Object
}
```

实现 Custom APIServer 时，为了能够支持多个 HTTP Endpoint，使用的是 `rest.StandardStorage` interface。

```go
// StandardStorage is an interface covering the common verbs. Provided for testing whether a
// resource satisfies the normal storage methods. Use Storage when passing opaque storage objects.
type StandardStorage interface {
	Getter
	Lister
	CreaterUpdater
	GracefulDeleter
	CollectionDeleter
	Watcher
}

// Getter is an object that can retrieve a named RESTful resource.
type Getter interface {
	// Get finds a resource in the storage by name and returns it.
	// Although it can return an arbitrary error value, IsNotFound(err) is true for the
	// returned error value err when the specified resource is not found.
	Get(ctx context.Context, name string, options *metav1.GetOptions) (runtime.Object, error)
}

// Lister is an object that can retrieve resources that match the provided field and label criteria.
type Lister interface {
	// NewList returns an empty object that can be used with the List call.
	// This object must be a pointer type for use with Codec.DecodeInto([]byte, runtime.Object)
	NewList() runtime.Object
	// List selects resources in the storage which match to the selector. 'options' can be nil.
	List(ctx context.Context, options *metainternalversion.ListOptions) (runtime.Object, error)
	// TableConvertor ensures all list implementers also implement table conversion
	TableConvertor
}

// CreaterUpdater is a storage object that must support both create and update.
// Go prevents embedded interfaces that implement the same method.
type CreaterUpdater interface {
	Creater
	Update(ctx context.Context, name string, objInfo UpdatedObjectInfo, createValidation ValidateObjectFunc, updateValidation ValidateObjectUpdateFunc, forceAllowCreate bool, options *metav1.UpdateOptions) (runtime.Object, bool, error)
}

// Creater is an object that can create an instance of a RESTful object.
type Creater interface {
	// New returns an empty object that can be used with Create after request data has been put into it.
	// This object must be a pointer type for use with Codec.DecodeInto([]byte, runtime.Object)
	New() runtime.Object

	// Create creates a new version of a resource.
	Create(ctx context.Context, obj runtime.Object, createValidation ValidateObjectFunc, options *metav1.CreateOptions) (runtime.Object, error)
}

// GracefulDeleter knows how to pass deletion options to allow delayed deletion of a
// RESTful object.
type GracefulDeleter interface {
	// Delete finds a resource in the storage and deletes it.
	// The delete attempt is validated by the deleteValidation first.
	// If options are provided, the resource will attempt to honor them or return an invalid
	// request error.
	// Although it can return an arbitrary error value, IsNotFound(err) is true for the
	// returned error value err when the specified resource is not found.
	// Delete *may* return the object that was deleted, or a status object indicating additional
	// information about deletion.
	// It also returns a boolean which is set to true if the resource was instantly
	// deleted or false if it will be deleted asynchronously.
	Delete(ctx context.Context, name string, deleteValidation ValidateObjectFunc, options *metav1.DeleteOptions) (runtime.Object, bool, error)
}

// CollectionDeleter is an object that can delete a collection
// of RESTful resources.
type CollectionDeleter interface {
	// DeleteCollection selects all resources in the storage matching given 'listOptions'
	// and deletes them. The delete attempt is validated by the deleteValidation first.
	// If 'options' are provided, the resource will attempt to honor them or return an
	// invalid request error.
	// DeleteCollection may not be atomic - i.e. it may delete some objects and still
	// return an error after it. On success, returns a list of deleted objects.
	DeleteCollection(ctx context.Context, deleteValidation ValidateObjectFunc, options *metav1.DeleteOptions, listOptions *metainternalversion.ListOptions) (runtime.Object, error)
}

// Watcher should be implemented by all Storage objects that
// want to offer the ability to watch for changes through the watch api.
type Watcher interface {
	// 'label' selects on labels; 'field' selects on the object's fields. Not all fields
	// are supported; an error should be returned if 'field' tries to select on a field that
	// isn't supported. 'resourceVersion' allows for continuing/starting a watch at a
	// particular version.
	Watch(ctx context.Context, options *metainternalversion.ListOptions) (watch.Interface, error)
}
```

所有嵌套的 Handle interface 都是包含名字对应的 HTTP Handle 实现，从名字上就可以知道对应的功能。之外还有许多个 Handle interface：

| Interface         | HTTP Method      | Endpoint              | 作用                                          |
| ----------------- | ---------------- | --------------------- | --------------------------------------------- |
| Lister            | List             | \<ResourcePath>        | 可以匹配指定 Field 和 Label 获取 ResourceList |
| Creater           | POST             | \<ResourcePath>        | 创建一个 Resource                             |
| CollectionDeleter | DELETECOLLECTION | \<ResourcePath>         删除一组 Resource                             |
| Getter            | GET              | \<ResourcePath>/{name} | 按 name 获取一个 Resource                     |
| Updater           | PUT              | \<ResourcePath>/{name} | 更新一个 Resource                             |
| Patcher           | PATCH            | \<ResourcePath>/{name} | Patch 方式更新一个 Resource Resource          |
| GracefulDeleter   | DELETE           | \<ResourcePath>/{name} | 删除一个 Resource                             |
| Watcher           | HTTP Method      | \<ResourcePath>        | 删除一组 Resource                             |
| Connecter     | Connect           | -       | Creater + Updater                             |
| CreateUpdater     | GET/PUT          | -       | Creater + Updater                             |
| Scoper            | -                | -                     | （必须实现）用于代码中判断是否是 Namespaced   |

{{< admonition tip ResourcePath>}}
ResourcePath 就是路径 /api/apiVersion/resource 或者 /api/apiVersion/{namespace}/resource。
{{< /admonition >}}

所以，我们需要将实现了 `rest.Storage` interface 的对象注册到 `APIGroupInfo` 中。

### 4.4 registry.Store 与 Strategy

要实现 `rest.Storage` interface？Kubernetes 依旧提供了实现了 `rest.Storage` 的类 `registry.Store`。我们真正要做的就是将一个回调函数设置到 Store 对象中。

```go
// Store implements k8s.io/apiserver/pkg/registry/rest.StandardStorage. It's
// intended to be embeddable and allows the consumer to implement any
// non-generic functions that are required. This object is intended to be
// copyable so that it can be used in different ways but share the same
// underlying behavior.
//
// All fields are required unless specified.
//
// The intended use of this type is embedding within a Kind specific
// RESTStorage implementation. This type provides CRUD semantics on a Kubelike
// resource, handling details like conflict detection with ResourceVersion and
// semantics. The RESTCreateStrategy, RESTUpdateStrategy, and
// RESTDeleteStrategy are generic across all backends, and encapsulate logic
// specific to the API.
//
// TODO: make the default exposed methods exactly match a generic RESTStorage
type Store struct {
	// ...
}
```

我们仅仅关注在如何使用 `rest.Store`，以 Flunder 的 REST 实现为例:

```go
// REST implements a RESTStorage for API services against etcd
type REST struct {
	*genericregistry.Store
}

func NewREST(scheme *runtime.Scheme, optsGetter generic.RESTOptionsGetter) (*registry.REST, error) {
	strategy := NewStrategy(scheme)

	store := &genericregistry.Store{
		NewFunc:                  func() runtime.Object { return &wardle.Flunder{} },
		NewListFunc:              func() runtime.Object { return &wardle.FlunderList{} },
		PredicateFunc:            MatchFlunder,
		DefaultQualifiedResource: wardle.Resource("flunders"),

		CreateStrategy: strategy,
		UpdateStrategy: strategy,
		DeleteStrategy: strategy,

		// TODO: define table converter that exposes more than name/creation timestamp
		TableConvertor: rest.NewDefaultTableConvertor(wardle.Resource("flunders")),
	}
	options := &generic.StoreOptions{RESTOptions: optsGetter, AttrFunc: GetAttrs}
	if err := store.CompleteWithOptions(options); err != nil {
		return nil, err
	}
	return &registry.REST{store}, nil
}
```

Store 已经实现了 Creater、Updater 等 interface，确切的说是 `rest.StandardStorage`。也就是说，Store 将最基本的 CRUD 逻辑实现了，我们需要实现的就是扩展点的函数。

每个操作的扩展点就称为 Strategy（总算看到了标题）。看下我们自己实现的 Strategy:

```go
type flunderStrategy struct {
	runtime.ObjectTyper
	names.NameGenerator
}

func (flunderStrategy) NamespaceScoped() bool {
	return true
}

func (flunderStrategy) PrepareForCreate(ctx context.Context, obj runtime.Object) {
}

func (flunderStrategy) PrepareForUpdate(ctx context.Context, obj, old runtime.Object) {
}

// Validate 用于对对象进行验证
func (flunderStrategy) Validate(ctx context.Context, obj runtime.Object) field.ErrorList {
	flunder := obj.(*wardle.Flunder)
	return validation.ValidateFlunder(flunder)
}

// WarningsOnCreate returns warnings for the creation of the given object.
func (flunderStrategy) WarningsOnCreate(ctx context.Context, obj runtime.Object) []string { return nil }

func (flunderStrategy) AllowCreateOnUpdate() bool {
	return false
}

func (flunderStrategy) AllowUnconditionalUpdate() bool {
	return false
}

func (flunderStrategy) Canonicalize(obj runtime.Object) {
}

func (flunderStrategy) ValidateUpdate(ctx context.Context, obj, old runtime.Object) field.ErrorList {
	return field.ErrorList{}
}

// WarningsOnUpdate returns warnings for the given update.
func (flunderStrategy) WarningsOnUpdate(ctx context.Context, obj, old runtime.Object) []string {
	return nil
}
```

### 4.5 总结

兜兜绕绕了一圈，我们总结一下这个复杂的结构：

1. 整个 APIServer 就是一个 HTTP Server；

2. HTTPServer 由多个 APIGroupInfo，每个 APIGroupInfo 代表了一组 HTTP 接口，即 **`/apis/\<group>`**；

3. 每个 APIGroupInfo 包含多个版本，即 **`/apis/\<group>/\<version>`**；

4. 每个版本可以包含多个 Resource 对应的 rest.Storage 实现，一个 rest.Storage 就是该 Resource 的 REST HTTP Handle，即 **`/apis/\<group>/\<version>/\<resource>`**。

5. registry.Store 是 Kubernetes 库提供的实现了 rest.Storage 的类，其提供多个扩展点，扩展点就称为 Strategy。

我们再回头看，其实这样复杂的结构实现就是为能够实现了 **`/apis/<group>/<version>/<resource>`** 以及处理函数的抽象。

## 5 Validate

**准入过程中的 Validate 能够实现针对字段动态验证**，比 CRD 定义中的静态验证更加灵活。由下图可见，Validate 过程是在 Mutate Plugin 与 Validate Plugin 之间完成的。

{{< image src="img1.png" height=400 >}}

因此 Validate 只需要为 Internal Version 实现一次，不许为各个 External Version 分别实现。Registry 与 Strategy 为我们封装了上面的真个 Resource Handler，因此 Validate 的入口其实就是 Strategy 中对于 Create/Update 的检查。

```go
type flunderStrategy struct {
	runtime.ObjectTyper
	names.NameGenerator
}

// ...

// Validate 用于对 Create Object 时进行验证
func (flunderStrategy) Validate(ctx context.Context, obj runtime.Object) field.ErrorList {
	flunder := obj.(*wardle.Flunder)
	return validation.ValidateFlunder(flunder)
}
```

后续就是我们自己实现的 Validate 逻辑，这里使用了 field.ErrorList 这样的方式来方便的展示出错误的字段。

```go
// ValidateFlunder validates a Flunder.
func ValidateFlunder(f *wardle.Flunder) field.ErrorList {
	allErrs := field.ErrorList{}

	// 嵌套的属性认证
	allErrs = append(allErrs, ValidateFlunderSpec(&f.Spec, field.NewPath("spec"))...) // field.NewPath(记录属性路径)

	return allErrs
}
```

由于 Validate 逻辑都是在代码中编写的，所以你可以对比多个字段来进行 Validate，灵活性页更高。

## 6 Admission

还是下图，每个请求在经过反序列化、默认值处理、转换为 Internal Version 后，都会经过 **`Admission Plugin Chain`** 处理（图中的 Admission 一块）。

{{< image src="img1.png" height=400 >}}

Admission Plugin 因为两个阶段，也就是可以分为两类插件：

* **Mutate Plugin**
* **Validate Plugin**

一个 Admission Plugin 可以同时属于这两类，那么在 Admission Controll 过程中，该插件会被调用两次（要么同时启动、要么同时禁用）。

* Mutate 阶段，所有 Mutate Plugin 会依次调用；
* Validate 阶段，所有 Validate Plugin 会被调用（可能并发地调用）；

{{< admonition note "Webhook Admission">}}
这里所说的都是继承到 APIServer 代码内部的 Admission Plugin，Kubernetes 也提供了外部基于 Webhook 的 Admission Control，也就是图中调用外部服务的地方。
{{< /admonition >}}

下面从入口开始看 Admission Plugin 如何实现的。

### 6.1 Admission Plugin 选项

在 APIServer 中，所有的 Admission Plugin 以及相关选项都记录在 APIServer Option `RecommendedOptions.Admission` 字段中，其是 `AdmissionOptions` 类型。

```go
// AdmissionOptions holds the admission options
type AdmissionOptions struct {
	// RecommendedPluginOrder holds an ordered list of plugin names we recommend to use by default
	RecommendedPluginOrder []string
	// DefaultOffPlugins is a set of plugin names that is disabled by default
	DefaultOffPlugins sets.String

	// EnablePlugins indicates plugins to be enabled passed through `--enable-admission-plugins`.
	EnablePlugins []string
	// DisablePlugins indicates plugins to be disabled passed through `--disable-admission-plugins`.
	DisablePlugins []string
	// ConfigFile is the file path with admission control configuration.
	ConfigFile string
	// Plugins contains all registered plugins.
	Plugins *admission.Plugins
	// Decorators is a list of admission decorator to wrap around the admission plugins
	Decorators admission.Decorators
}
```

* RecommendedPluginOrder - Plugin 执行顺序
* EnablePlugins - 通过命令行参数 `--enable-admission-plugins` 打开的 Plugin
* DisablePlugins - 通过命令行参数 `--disable-admission-plugins` 关闭的 Plugin
* Plugins - 所有注册了的 Plugin

与 Kubernetes APIServer 包含的 Plugin 不同，`NewAdmissionOptions()` 仅仅包含三个默认的 Plugin：

* NamespaceLifecycle - 根据 namespace 阶段来执行一些约束，包括：禁止在删除中的 namespace 中创建新对象，对不存在的 namespace 请求拒绝，禁止删除 default、kube-system、kube-public 三个 namespace。
* ValidatingAdmissionWebhook - 允许注册与使用 Validating Admission Webhook。
* MutatingAdmissionWebhook - 允许注册与使用 Mutating Admission Webhook。

{{< admonition note Note>}}
Kubernetes APIServer 提供了许多的 Admission Plugin，具体见 [**使用准入控制器**](https://kubernetes.io/zh/docs/reference/access-authn-authz/admission-controllers/#%E6%AF%8F%E4%B8%AA%E5%87%86%E5%85%A5%E6%8E%A7%E5%88%B6%E5%99%A8%E7%9A%84%E4%BD%9C%E7%94%A8%E6%98%AF%E4%BB%80%E4%B9%88)
{{< /admonition >}}

代码中，在 APIServer 启动过程中的 `Complete()` 时会进行自定义 Plugin 的注册，调用 `*admission.Plugins.Register()` 即可:

```go
// Complete fills in fields required to have valid data
func (o *WardleServerOptions) Complete() error {
	// 注册 BanFlunder 的 Admission Plugin
	// register admission plugins
	banflunder.Register(o.RecommendedOptions.Admission.Plugins)

	// 记录到 Plugins 中
	// add admission plugins to the RecommendedPluginOrder
	o.RecommendedOptions.Admission.RecommendedPluginOrder = append(o.RecommendedOptions.Admission.RecommendedPluginOrder, "BanFlunder")

	return nil
}

// banflunder.Register
// Register registers a plugin
func Register(plugins *admission.Plugins) {
	plugins.Register("BanFlunder", func(config io.Reader) (admission.Interface, error) {
		return New()
	})
}
```

### 6.2 Admission Plugin Interface

要实现一个 Admission Plugin 需要实现以下接口：

* `addmission.Interface` - 用于注册到 Plugin Chain 中
* `addmission.MutatingInterface`（可选）- 能够在 Mutate 阶段被调用
* `addmission.ValidationInterface`（可选）- 能够在 Validate 阶段被调用

这三个接口都在 **`k8s.io/apiserver/pkg/admission`** 包中：

```go
// Interface is an abstract, pluggable interface for Admission Control decisions.
type Interface interface {
	// Handles returns true if this admission controller can handle the given operation
	// where operation can be one of CREATE, UPDATE, DELETE, or CONNECT
	Handles(operation Operation) bool
}

type MutationInterface interface {
	Interface

	// Admit makes an admission decision based on the request attributes.
	// Context is used only for timeout/deadline/cancellation and tracing information.
	Admit(ctx context.Context, a Attributes, o ObjectInterfaces) (err error)
}

// ValidationInterface is an abstract, pluggable interface for Admission Control decisions.
type ValidationInterface interface {
	Interface

	// Validate makes an admission decision based on the request attributes.  It is NOT allowed to mutate
	// Context is used only for timeout/deadline/cancellation and tracing information.
	Validate(ctx context.Context, a Attributes, o ObjectInterfaces) (err error)
}
```

* Interface.Handles() 根据操作对请求进行过滤，因此可以针对特定的请求才进行 Mutate 与 Validate 阶段；
* MutationInterface.Admit() 验证请求，并允许变更对象；
* ValidationInterface.Validate() 仅仅验证请求；

### 6.3 实现 Admission Plugin

好了，我们已经知道在哪里注册 Plugin，以及 Plugin 应该实现哪些接口了，我们开始看一个 Plugin 的实现：

```go
// New creates a new ban flunder admission plugin
func New() (*DisallowFlunder, error) {
	return &DisallowFlunder{
		Handler: admission.NewHandler(admission.Create),
	}, nil
}

// DisallowFlunder is a ban flunder admission plugin
type DisallowFlunder struct {
	*admission.Handler
	lister listers.FischerLister
}

// SetInternalWardleInformerFactory gets Lister from SharedInformerFactory.
// The lister knows how to lists Fischers.
func (d *DisallowFlunder) SetInternalWardleInformerFactory(f informers.SharedInformerFactory) {
	d.lister = f.Wardle().V1alpha1().Fischers().Lister()
	d.SetReadyFunc(f.Wardle().V1alpha1().Fischers().Informer().HasSynced)
}

// ValidateInitialization checks whether the plugin was correctly initialized.
func (d *DisallowFlunder) ValidateInitialization() error {
	if d.lister == nil {
		return fmt.Errorf("missing fischer lister")
	}
	return nil
}
```

* `admission.Handler` - 内嵌 admission.Handler 的对象实现了 Interface.Handles() 方法，我们只需要注册我们感兴趣的 Operation（Create）；

* `SetInternalWardleInformerFactory` - 传入需要的 Lister

`Admit()` 实现就很简单了，进行一些检查：

```go
func (d *DisallowFlunder) Admit(ctx context.Context, a admission.Attributes, o admission.ObjectInterfaces) error {
	// 检查 GVK
	// we are only interested in flunders
	if a.GetKind().GroupKind() != wardle.Kind("Flunder") {
		return nil
	}

	// 等待 Lister 准备好
	if !d.WaitForReady() {
		return admission.NewForbidden(a, fmt.Errorf("not yet ready to handle request"))
	}

	// 进行业务上的 Validate
	metaAccessor, err := meta.Accessor(a.GetObject())
	if err != nil {
		return err
	}
	flunderName := metaAccessor.GetName()

	fischers, err := d.lister.List(labels.Everything())
	if err != nil {
		return err
	}

	// 黑名单检查
	for _, fischer := range fischers {
		for _, disallowedFlunder := range fischer.DisallowedFlunders {
			if flunderName == disallowedFlunder {
				return errors.NewForbidden(
					a.GetResource().GroupResource(),
					a.GetName(),
					fmt.Errorf("this name may not be used, please change the resource name"),
				)
			}
		}
	}
	return nil
}
```

### 6.4 基础组件

Admission Plugin 通常需要 client 和 Informer 去访问其他资源，可以在插件的初始化逻辑中准备好这些基础组件。

库中提供了 `admission.PluginInitializer` interface 抽象来进行 Plugin 的初始化操作，初始化过程中会对所有的 Plugin 调用每个 `PluginInitializer.Initialize()`。

```go
// PluginInitializer is used for initialization of shareable resources between admission plugins.
// After initialization the resources have to be set separately
type PluginInitializer interface {
	Initialize(plugin Interface)
}
```

根据前面的 Plugin 实现中的 `SetInternalWardleInformerFactory()` 接口，我们只需要实现了自己的 Initializer 来传递 Informer 即可：

```go
type pluginInitializer struct {
	informers informers.SharedInformerFactory
}

// New creates an instance of wardle admission plugins initializer.
func New(informers informers.SharedInformerFactory) pluginInitializer {
	return pluginInitializer{
		informers: informers,
	}
}

// Initialize checks the initialization interfaces implemented by a plugin
// and provide the appropriate initialization data
func (i pluginInitializer) Initialize(plugin admission.Interface) {
	if wants, ok := plugin.(WantsInternalWardleInformerFactory); ok {
		wants.SetInternalWardleInformerFactory(i.informers) // 将 Informer 传递给 Plugin
	}
}

// WantsInternalWardleInformerFactory defines a function which sets InformerFactory for admission plugins that need it
type WantsInternalWardleInformerFactory interface {
	SetInternalWardleInformerFactory(informers.SharedInformerFactory)
	admission.InitializationValidator
}
```

* WantsInternalWardleInformerFactory - 仅仅是特针对于 Informer 的 interface 实现，也是为了单向的抽象，这样如果多个 Plugin 依赖于同一个 Informer，那么至需要创建一个 Initializer

最后，所有 `admission.PluginInitializer` 记录到 `RecommendedOptions.ExtraAdmissionInitializers` 字段，这样就会自动调用进行初始化。

```go
	o.RecommendedOptions.ExtraAdmissionInitializers = func(c *genericapiserver.RecommendedConfig) ([]admission.PluginInitializer, error) {
		// 创建 ClientSet
		client, err := clientset.NewForConfig(c.LoopbackClientConfig)
		if err != nil {
			return nil, err
		}
		// 创建 CustomResource Informer，并记录到 Option 中
		informerFactory := informers.NewSharedInformerFactory(client, c.LoopbackClientConfig.Timeout)
		o.SharedInformerFactory = informerFactory

		// 构建 Admission Plugin Initializer，传递了 Informer
		// Initializer.Initialize 将 Informer 传递给了 Admission Plugin
		return []admission.PluginInitializer{wardleinitializer.New(informerFactory)}, nil
	}
```

### 6.5 总结

又是一个复杂的框架搬的实现，看下重要的几个点：

1. `RecommendedOptions.Admission` 记录着所有需要执行的 Admission Plugin，自己编写的也是注册到其中。

2. `RecommendedOptions.ExtraAdmissionInitializers` 记录着初始化 Plugin 的实现，用于在代码中向各个 Plugin 传递 Informer 等基础组件。

3. 我们需要编写类实现 admission.Interface + admission.MutationInterface + admission.ValidationInterface。在 Admit() 函数中进行 Mutate，在 Validate() 函数中进行 Validate。

4. 如果我们的 Plugin 需要使用 Informer 这样的组件，还需要编写一个类实现 admission.PluginInitializer，用于传递 Informer。

## 7 初始化与启动

看 main 函数，一切的入口是一个 Option:

```go
func main() {
	// 初始化日志
	logs.InitLogs()
	defer logs.FlushLogs()

	// 注册 SIGTERM 与 SIGINT 信号
	stopCh := genericapiserver.SetupSignalHandler()

	// 构建 Option
	options := server.NewWardleServerOptions(os.Stdout, os.Stderr)

	// 得到 cmd 命令行
	cmd := server.NewCommandStartWardleServer(options, stopCh)
	cmd.Flags().AddGoFlagSet(flag.CommandLine)
	if err := cmd.Execute(); err != nil {
		klog.Fatal(err)
	}
}
```

`NewCommandStartWardleServer` 得到一个 cobra.Command，所以其真正的运行入口在这里面：

```go
// NewCommandStartWardleServer provides a CLI handler for 'start master' command
// with a default WardleServerOptions.
func NewCommandStartWardleServer(defaults *WardleServerOptions, stopCh <-chan struct{}) *cobra.Command {
	o := *defaults
	cmd := &cobra.Command{
		Short: "Launch a wardle API server",
		Long:  "Launch a wardle API server",
		RunE: func(c *cobra.Command, args []string) error {
			if err := o.Complete(); err != nil {
				return err
			}

			if err := o.Validate(args); err != nil {
				return err
			}

			if err := o.RunWardleServer(stopCh); err != nil {
				return err
			}
			return nil
		},
	}

	// 注册命令行参数，Option 中所有字段都可以使用命令行参数配置
	flags := cmd.Flags()
	o.RecommendedOptions.AddFlags(flags)
	utilfeature.DefaultMutableFeatureGate.AddFlag(flags)

	return cmd
}
```

### 7.1 Options

启动 Server 的第一个阶段是得到 Option，代码中创建的是 `WardleServerOptions`，其最重要的就是包含了 `RecommendedOptions`，这是 k8s.io/apiserver 库提供的包含所有最基本的 APIServer 配置。

```go
// WardleServerOptions contains state for master/api server
type WardleServerOptions struct {
	// RecommendedOptions 记录官方的配置，包含大多数 APIServer 指定的配置
	RecommendedOptions *genericoptions.RecommendedOptions

	SharedInformerFactory informers.SharedInformerFactory
	StdOut                io.Writer
	StdErr                io.Writer
}

// RecommendedOptions contains the recommended options for running an API server.
// If you add something to this list, it should be in a logical grouping.
// Each of them can be nil to leave the feature unconfigured on ApplyTo.
type RecommendedOptions struct {
	Etcd           *EtcdOptions  // 后端存储相关
	SecureServing  *SecureServingOptionsWithLoopback // HTTPS 相关配置
	Authentication *DelegatingAuthenticationOptions 
	Authorization  *DelegatingAuthorizationOptions
	Audit          *AuditOptions   // 审计相关，默认关闭，开启后可以输出审计日志或者发送审计事件到外部后端系统
	Features       *FeatureOptions // 开启或禁用某些 Alpha 或 Beta 功能
	CoreAPI        *CoreAPIOptions // 访问 Kubernetes APIServer 的 kubeconfig 文件路径

	// FeatureGate is a way to plumb feature gate through if you have them.
	FeatureGate featuregate.FeatureGate
	// ExtraAdmissionInitializers is called once after all ApplyTo from the options above, to pass the returned
	// admission plugin initializers to Admission.ApplyTo.
	ExtraAdmissionInitializers func(c *server.RecommendedConfig) ([]admission.PluginInitializer, error)
	Admission                  *AdmissionOptions
	// API Server Egress Selector is used to control outbound traffic from the API Server
	EgressSelector *EgressSelectorOptions
	// Traces contains options to control distributed request tracing.
	Traces *TracingOptions
}
```

我们重点关注使用 `RecommendedOptions`，使用提供的 `genericoptions.NewRecommendedOptions()` 来创建：

```go
const defaultEtcdPathPrefix = "/registry/wardle.example.com"

// NewWardleServerOptions returns a new WardleServerOptions
func NewWardleServerOptions(out, errOut io.Writer) *WardleServerOptions {
	o := &WardleServerOptions{
		// NewRecommendedOptions 创建 APIServer 推荐的配置，大多数都包含了默认的配置
		RecommendedOptions: genericoptions.NewRecommendedOptions(
			defaultEtcdPathPrefix, // 存在 Etcd 中的 path 前缀
			apiserver.Codecs.LegacyCodec(v1alpha1.SchemeGroupVersion), // 注册解码器
		),

		StdOut: out,
		StdErr: errOut,
	}
	o.RecommendedOptions.Etcd.StorageConfig.EncodeVersioner = runtime.NewMultiGroupVersioner(v1alpha1.SchemeGroupVersion, schema.GroupKind{Group: v1alpha1.GroupName})
	return o
}
```

* L13 - 定义 ETCD 存储的版本

#### 7.1.1 Complete Option

启动 Server 的第一步，就是调用 `WardleServerOptions.Complete()`，其中主要作用就是注册自定义的 Admission Plugin：

```go
// Complete fills in fields required to have valid data
func (o *WardleServerOptions) Complete() error {
	// 注册 BanFlunder 的 Admission Plugin
	// register admission plugins
	banflunder.Register(o.RecommendedOptions.Admission.Plugins)

	// 记录到 Plugins 中
	// add admission plugins to the RecommendedPluginOrder
	o.RecommendedOptions.Admission.RecommendedPluginOrder = append(o.RecommendedOptions.Admission.RecommendedPluginOrder, "BanFlunder")

	return nil
}
```

#### 7.1.2 Validate Option

启动 Server 的第二步，调用 `WardleServerOptions.Validate()` 进行参数验证。因为我们没有使用额外的参数，所以直接调用 `RecommendedOptions.Validate()`:

```go
// Validate validates WardleServerOptions
func (o WardleServerOptions) Validate(args []string) error {
	errors := []error{}
	// 验证 Option 的合法性，执行各个 XXXOptions.Validate()
	errors = append(errors, o.RecommendedOptions.Validate()...)
	return utilerrors.NewAggregate(errors)
}
```

#### 7.1.3 Run Server

启动 Server 的第三步，调用 `WardleServerOptions.RunWardleServer` 运行一个 Server。其流程为：Options -> Config -> APIServer，最后真正运行：

```go
// RunWardleServer starts a new WardleServer given WardleServerOptions
func (o WardleServerOptions) RunWardleServer(stopCh <-chan struct{}) error {
	// Option 转化为 APIServer Config
	config, err := o.Config()
	if err != nil {
		return err
	}

	// config.Complete() 填充一些默认的参数
	// New() 得到一个 Server 对象，并注册 APIGroup 到 Kubernetes APIServer
	// NOTE: Custom APIServer 的关键就在这里，注册了一个 APIGroup，Kubernetes APIServer
	// 会根据注册的 APIGroup 来转发请求
	server, err := config.Complete().New()
	if err != nil {
		return err
	}

	// 注册一个 PostStart Hook
	// Hook 会在 HTTPs Server 启动并监听后被调用
	server.GenericAPIServer.AddPostStartHookOrDie("start-sample-server-informers", func(context genericapiserver.PostStartHookContext) error {
		// 启动原生对象的 SharedInformer
		config.GenericConfig.SharedInformerFactory.Start(context.StopCh)
		// 启动 Custom Resource Informer
		o.SharedInformerFactory.Start(context.StopCh)
		return nil
	})

	// PrepareRun 执行启动前的准备工作
	// Run 运行 HTTPs Server
	return server.GenericAPIServer.PrepareRun().Run(stopCh)
}
```

可以看到，Option 对视为针对于命令行参数的数据结构，Config 是针对于 APIServer 配置的数据结构。其通过 `WardleServerOptions.Config()` 进行转化。

```go
// Config returns config for the api server given WardleServerOptions
func (o *WardleServerOptions) Config() (*apiserver.Config, error) {
	// 创建一个自签名的证书，用于用户没有传预生成证书时使用
	// TODO have a "real" external address
	if err := o.RecommendedOptions.SecureServing.MaybeDefaultWithSelfSignedCerts("localhost", nil, []net.IP{netutils.ParseIPSloppy("127.0.0.1")}); err != nil {
		return nil, fmt.Errorf("error creating self-signed certificates: %v", err)
	}

	// 设置 Storage Paging
	o.RecommendedOptions.Etcd.StorageConfig.Paging = utilfeature.DefaultFeatureGate.Enabled(features.APIListChunking)

	// 注册 ExtraAdmissionInitializers
	// AdmissionInitializers 在 Option.ApplyTo 调用后执行一次，并将返回值 []admission.PluginInitializer 传递给 Admission.Applyto
	// 这里用来初始化了 ClientSet 与 Informer，并传递给了自身 Server 的 Admission Plugin
	// NOTE: 所以这里是用于让 Custom APIServer 与 Custom Admission Plugin 传递对象用的
	o.RecommendedOptions.ExtraAdmissionInitializers = func(c *genericapiserver.RecommendedConfig) ([]admission.PluginInitializer, error) {
		// 创建 ClientSet
		client, err := clientset.NewForConfig(c.LoopbackClientConfig)
		if err != nil {
			return nil, err
		}
		// 创建 CustomResource Informer，并记录到 Option 中
		informerFactory := informers.NewSharedInformerFactory(client, c.LoopbackClientConfig.Timeout)
		o.SharedInformerFactory = informerFactory

		// 构建 Admission Plugin Initializer，传递了 Informer
		// Initializer.Initialize 将 Informer 传递给了 Admission Plugin
		return []admission.PluginInitializer{wardleinitializer.New(informerFactory)}, nil
	}

	// 新建 RecommendedConfig
	serverConfig := genericapiserver.NewRecommendedConfig(apiserver.Codecs)

	// 配置生成 OpenAPI 相关配置
	serverConfig.OpenAPIConfig = genericapiserver.DefaultOpenAPIConfig(sampleopenapi.GetOpenAPIDefinitions, openapi.NewDefinitionNamer(apiserver.Scheme))
	serverConfig.OpenAPIConfig.Info.Title = "Wardle"
	serverConfig.OpenAPIConfig.Info.Version = "0.1"

	// 将 Option 转换为 Config
	if err := o.RecommendedOptions.ApplyTo(serverConfig); err != nil {
		return nil, err
	}

	// 得到 APIServer Config
	// 包含:
	//	1. RecommendedConfig - 预定的通用 Config
	//	2. ExtraConfig - 自身可传递的一些 Config
	config := &apiserver.Config{
		GenericConfig: serverConfig,
		ExtraConfig:   apiserver.ExtraConfig{},
	}
	return config, nil
}
```

可以看到，Option 转化为 Config 过程中，执行了之前讲过的向 Admission Plugin 传递基础组件的过程。

`k8s.io/apiserver` 中，我们基于 RecommendOptions 来实现所有的选项。

```go
import ( 
    // ... 
    informers "github.com/programming-kubernetes/pizza-apiserver/pkg/generated/informers/externalversions" 
) 

// etcd 中存储数据的键前缀
const defaultEtcdPathPrefix = "/registry/restaurant.programming-
kubernetes.info" 

// 自定义的 Options 结构
type CustomServerOptions struct { 
    RecommendedOptions *genericoptions.RecommendedOptions 
    SharedInformerFactory informers.SharedInformerFactory 
} 
 
func NewCustomServerOptions(out, errOut io.Writer) *CustomServerOptions { 
    o := &CustomServerOptions{ 
        RecommendedOptions: genericoptions.NewRecommendedOptions( 
            defaultEtcdPathPrefix, 
            apiserver.Codecs.LegacyCodec(v1alpha1.SchemeGroupVersion), 
            genericoptions.NewProcessInfo("pizza-apiserver", "pizza-
apiserver"), 
        ), 
    } 
 
    return o 
}
```

### 7.2 Config

启动 Server 的第二阶段是 Config 类，与 Option 类似，我们会包含 k8s.io/apiserver 库提供的 `RecommendedConfig`:

```go
type Config struct {
	GenericConfig *genericapiserver.RecommendedConfig
	ExtraConfig   ExtraConfig
}
```

`RecommendedConfig` 有着许多的配置项，所以我们不深入其实现，主要观察如何使用它。也就是如何从 Config，得到 APIServer 对象。

正如之前看到的，由 Option 转化为 Config 对象：

```go
	// 新建 RecommendedConfig
	serverConfig := genericapiserver.NewRecommendedConfig(apiserver.Codecs)

	// 配置生成 OpenAPI 相关配置
	serverConfig.OpenAPIConfig = genericapiserver.DefaultOpenAPIConfig(sampleopenapi.GetOpenAPIDefinitions, openapi.NewDefinitionNamer(apiserver.Scheme))
	serverConfig.OpenAPIConfig.Info.Title = "Wardle"
	serverConfig.OpenAPIConfig.Info.Version = "0.1"

	// 将 Option 转换为 Config
	if err := o.RecommendedOptions.ApplyTo(serverConfig); err != nil {
		return nil, err
	}

	// 得到 APIServer Config
	// 包含:
	//	1. RecommendedConfig - 预定的通用 Config
	//	2. ExtraConfig - 自身可传递的一些 Config
	config := &apiserver.Config{
		GenericConfig: serverConfig,
		ExtraConfig:   apiserver.ExtraConfig{},
	}
```

从 Config 得到 Server 对象就很简单了：

```go
	server, err := config.Complete().New()
	if err != nil {
		return err
	}
```

#### 7.2.1 Complete Config

Complete 过程依旧简单，执行 `GenericConfig.Complete()` 以及设置到版本号即可，返回一个 `CompletedConfig` 对象：

```go
// CompletedConfig embeds a private pointer that cannot be instantiated outside of this package.
type CompletedConfig struct {
	*completedConfig
}

func (cfg *Config) Complete() CompletedConfig {
	c := completedConfig{
		cfg.GenericConfig.Complete(),
		&cfg.ExtraConfig,
	}

	c.GenericConfig.Version = &version.Info{
		Major: "1",
		Minor: "0",
	}

	return CompletedConfig{&c}
}
```

{{< admonition tip "为何使用新的对象 CompletedConfig?">}}
CompletedConfig 才包含 New() 方法，这样能保证 New APIServer 时一定是经过 Complete Config 的。
{{< /admonition >}}

#### 7.2.2 New APIServer

New APIServer 过程在创建一个 APIServer 对象后，就完成了 HTTP API Handle 的注册了，而这就是最关键的地方：

```go
// New returns a new instance of WardleServer from the given config.
func (c completedConfig) New() (*WardleServer, error) {
	// CompletedConfig 得到一个 GenericAPIServer
	genericServer, err := c.GenericConfig.New("sample-apiserver", genericapiserver.NewEmptyDelegate())
	if err != nil {
		return nil, err
	}

	// Custom APIServer 对象
	s := &WardleServer{
		GenericAPIServer: genericServer,
	}

	// 构建 APIGroupInfo
	apiGroupInfo := genericapiserver.NewDefaultAPIGroupInfo(wardle.GroupName, Scheme, metav1.ParameterCodec, Codecs)

	// 构建 APIGroup
	// 一个 rest.Storage 对应了一个 HTTP API Endpoint
	// 即 /apis/<group>/<version>/<resource>

	// 下面注册了两个 Endpoint
	//	+ /apis/wardle.example.com/v1alpha1/flunders
	//	+ /apis/wardle.example.com/v1alpha1/fischers
	v1alpha1storage := map[string]rest.Storage{}
	v1alpha1storage["flunders"] = wardleregistry.RESTInPeace(flunderstorage.NewREST(Scheme, c.GenericConfig.RESTOptionsGetter))
	v1alpha1storage["fischers"] = wardleregistry.RESTInPeace(fischerstorage.NewREST(Scheme, c.GenericConfig.RESTOptionsGetter))
	apiGroupInfo.VersionedResourcesStorageMap["v1alpha1"] = v1alpha1storage

	// 下面注册了一个 Endpoint
	//	+ /apis/wardle.example.com/v1beta1/flunders
	v1beta1storage := map[string]rest.Storage{}
	v1beta1storage["flunders"] = wardleregistry.RESTInPeace(flunderstorage.NewREST(Scheme, c.GenericConfig.RESTOptionsGetter))
	apiGroupInfo.VersionedResourcesStorageMap["v1beta1"] = v1beta1storage

	// 注册 APIGroup 到 Kubernetes APIServer
	if err := s.GenericAPIServer.InstallAPIGroup(&apiGroupInfo); err != nil {
		return nil, err
	}

	return s, nil
}
```

### 7.3 Server

总算来到最后的阶段，Server 对象就是一个 APIServer 的实现了，我们只需要调用库提供的 `GenericAPIServer` 实现的 PrepareRun() 与 Run() 接口实现即可运行。

```go
	server, err := config.Complete().New()
	if err != nil {
		return err
	}

	// 注册一个 PostStart Hook
	// Hook 会在 HTTPs Server 启动并监听后被调用
	server.GenericAPIServer.AddPostStartHookOrDie("start-sample-server-informers", func(context genericapiserver.PostStartHookContext) error {
		// 启动原生对象的 SharedInformer
		config.GenericConfig.SharedInformerFactory.Start(context.StopCh)
		// 启动 Custom Resource Informer
		o.SharedInformerFactory.Start(context.StopCh)
		return nil
	})

	// PrepareRun 执行启动前的准备工作
	// Run 运行 HTTPs Server
	return server.GenericAPIServer.PrepareRun().Run(stopCh)
```

## 8 部署

### 8.1 部署清单

清单：
* `APIService`
* `Service`
* `Deployment`
* `ServiceAccount` + `ClusterRole` + `ClusterRoleBinding`

之前提到，APIService 对象向原生 Kubernetes APIServer 注册一个 Custom APISever。因此这是必须要部署的。

```yaml
apiVersion: apiregistration.k8s.io/v1
kind: APIService
metadata:
  name: v1alpha1.wardle.example.com
spec:
  insecureSkipTLSVerify: true
  group: wardle.example.com
  groupPriorityMinimum: 1000
  versionPriority: 15
  service:
    name: api
    namespace: wardle
  version: v1alpha1
```

注意，测试环境我们将 `spec.insecureSkipTLSVerify` 设为 true，而生产环境不能这么做。

APIService 仅仅是让 Kubernetes APIServer 知晓 Custom APIServer 存在，为了能够转发请求，我们还需要部署 Custom APIServer 使用的 Service 对象。

```yaml
apiVersion: v1
kind: Service
metadata:
  name: api
  namespace: wardle
spec:
  ports:
  - port: 443
    protocol: TCP
    targetPort: 443
  selector:
    apiserver: "true"
```

运行我们 Custom APIServer 程序的 Deployment。

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: wardle-server
  namespace: wardle
  labels:
    apiserver: "true"
spec:
  replicas: 1
  selector:
    matchLabels:
      apiserver: "true"
  template:
    metadata:
      labels:
        apiserver: "true"
    spec:
      serviceAccountName: apiserver
      containers:
      - name: wardle-server
        # build from staging/src/k8s.io/sample-apiserver/artifacts/simple-image/Dockerfile
        # or
        # docker pull k8s.gcr.io/e2e-test-images/sample-apiserver:1.17.4
        # docker tag k8s.gcr.io/e2e-test-images/sample-apiserver:1.17.4 kube-sample-apiserver:latest
        image: kube-sample-apiserver:latest
        imagePullPolicy: Never
        args: [ "--etcd-servers=http://localhost:2379" ]
      - name: etcd
        image: quay.io/coreos/etcd:v3.5.0
```

{{< admonition note Note>}}
APIServer 是无状态的，所以如果你只部署一个 etcd 作为后端存储情况下，可以运行多个 Custom APIServer，并且不需要进行选举。
{{< /admonition >}}

访问 Kubernetes APIServer 使用的 ServiceAccount：

```yaml
kind: ServiceAccount
apiVersion: v1
metadata:
  name: apiserver
  namespace: wardle
```

ClusterRole 提供相关的访问权限，主要包括：

* namespace - 为了实现 Namespace 删除时，其下相关对象也被删除，需要 namespace 相关权限
* admission webhook - 为了能够支持 admission webhook

```yaml
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: aggregated-apiserver-clusterrole
rules:
- apiGroups: [""]
  resources: ["namespaces"]
  verbs: ["get", "watch", "list"]
- apiGroups: ["admissionregistration.k8s.io"]
  resources: ["mutatingwebhookconfigurations", "validatingwebhookconfigurations"]
  verbs: ["get", "watch", "list"]
```

ClusterRoleBinding 连接 ClusterRole 与 ServiceAccount，并且还需要绑定一些预创建的 ClusterRole。

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: sample-apiserver-clusterrolebinding
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: aggregated-apiserver-clusterrole
subjects:
- kind: ServiceAccount
  name: apiserver
  namespace: wardle

apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: wardle:system:auth-delegator
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:auth-delegator
subjects:
- kind: ServiceAccount
  name: apiserver
  namespace: wardle

# 为了代理认证和授权
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: wardle-auth-reader
  namespace: kube-system
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: extension-apiserver-authentication-reader
subjects:
- kind: ServiceAccount
  name: apiserver
  namespace: wardle
```

### 8.2 配置证书

前面我们使用 APIService 中的 `spec.insecureSkipTLSVerify` 为 false 来让 Custom APIServer 与 Kubernetes APIServer 之前通信跳过 TLS 鉴权。

如果需要配置 TLS，可以在 APIService 中的 `spec.caBundle` 字段配置 Custom APISever 的根证书，这样 Kubernetes APIServer 就会使用该证书来对 Custom APISever 进行鉴权。

对于 Custom APIServer，我们创建好 Server 的证书与私钥后，创建一个 Secret 对象，然后将其挂载到 Pod 的 `/var/run/apiserver/serving-cert/tls.{crt,key}` 文件中。

{{< admonition note Note>}}
`/var/run/apiserver/serving-cert/tls.{crt,key}` 是默认的放置 APIServer 证书与私钥的目录。

你可以通过命令行参数 `--cert-dir "dir"` 指定证书与私钥的目录。

或者，通过 `--tls-cert-file "file"` 与 `--tls-private-key-file "file"` 指定证书文件与私钥文件。
{{< /admonition >}}

## 参考

* [**k8s.io/apiserver**](https://github.com/kubernetes/apiserver)
* [**《Programming Kubernetes》**](https://book.douban.com/subject/33393681/)
* [**Configure the Aggregation Layer**](https://kubernetes.io/docs/tasks/extend-kubernetes/configure-aggregation-layer/)
