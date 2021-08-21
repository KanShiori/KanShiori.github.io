# K8s 编程 - 代码生成


> **Kubernetes 编程系列** 主要记录一些开发 Controller 所相关的知识，大部分内容来自于[《Programming Kubernetes》](https://www.oreilly.com/library/view/programming-kubernetes/9781492047094/)（推荐直接阅读）。

最简单的方式是调用 [**generate-groups.sh**](https://github.com/kubernetes/code-generator/blob/master/generate-groups.sh) 或者 [**hack/update-codegen.sh**](https://github.com/kubernetes/code-generator/blob/master/hack/update-codegen.sh) 脚本来生成。

```shell
$ vendor/k8s.io/code-generator/generate-groups.sh all \ 
    github.com/programming-kubernetes/cnat/cnat-client-go/pkg/generated \
    github.com/programming-kubernetes/cnat/cnat-client-go/pkg/apis \ 
    cnat:v1alpha1 \ 
    --output-base "${GOPATH}/src" \ 
    --go-header-file "hack/boilerplate.go.txt"
```
* arg2 - 指定要生成的 client、list、informer 的包名；
* arg3 - APIGroup 所在的包；
* arg4 - Group:Version；
* --output-base 作为参数传递给所有 generator，作为查找包的基础路径；
* --go-header 提供生成代码后加上的版权头信息；

all 参数意味着会调用四种标准的 code generator：
* **deepcopy-gen** - 生成 `func(t *T) DeepCopy()` 和 `func(t *T) DeepCopyInto(*T)` 方法，用于对象深拷贝。
* **client-gen** - 生成强类型的 clientset。
* **informer-gen** - 生成自定义资源的 Informer。
* **lister-gen** - 生成自定义资源的 Lister 对象，为 Get 和 List 请求提供只读缓存层。

另外，code-generator 还有一些额外的 generator，当你需要开发自己的 APIServer 时可能会用到：
* **conversion-go** - 创建用于转换内部类型与外部类型的函数。
* **defaulter-gen** - 生成处理默认值字段的代码。

{{< admonition tip "verify-codegen.sh">}}
代码仓库中还会包含 [**hack/verify-codegen.sh**](https://github.com/kubernetes/code-generator/blob/48d3d44c66d61e1a6cf0bd5bd1ecf94c3e46a270/hack/verify-codegen.sh)，用于检查生产的代码是否是最新的，常常用于 CI 中。
{{< /admonition >}}

除了通过命令行参数，可以通过在源码中使用特定的标签来控制代码生成。标签分为两种：
* doc.go 文件中 package 行前面的全局标签
* 类声明前的局部标签。

## 1 全局标签

**`全局标签`** 会写在 doc.go 文件中。
```go
// +k8s:deepcopy-gen=package
 
// Package v1 is the v1alpha1 version of the API.
// +groupName=cnat.programming-kubernetes.info
package v1alpha1
```
* L1 - **表明要为包中的所有类型生成 DeepCopy() 方法。**
  
  如果后续某些类型不需要生成 DeepCopy，可以通过 `// +k8s:deepcopy-gen=false` 表明。
* L4 - **定义了 APIGroup 的全名**，如果 Go 的包名不遵循 Group 的命名，那么可以用这个标签指定 Group。
  
  通过 `// +groupName` 标签，代码生成器才能使用正确的 HTTP API Path。

{{< admonition tip "+groupGoName">}}
除了 +groupName，还有 `// +groupGoName` 可以自定义 CR 类型的 Kind。默认下，变量与类型的名字与 Kind 相同，都是以首字母大写作为表示。

例如 Cnat，更好的命名应该是 CNAt，通过 `// +groupGoName=CNAt`，client-gen 生成的代码就是下面样子：
```go
type Interface interface { 
    Discovery() discovery.DiscoveryInterface 
    CNatV1() atv1alpha1.CNatV1alpha1Interface 
}
```
{{< /admonition >}}

## 2 局部标签

**`局部标签`** 写在 API 类型的前面，或者它前面第二个注释块中：
```go
// AtSpec defines the desired state of At
type AtSpec struct { 
    // Schedule is the desired time the command is supposed to be executed.
    // Note: the format used here is UTC time https://www.utctime.net
    Schedule string `json:"schedule,omitempty"` 
    // Command is the desired command (executed in a Bash shell) to be executed.
    Command string `json:"command,omitempty"` 
    // Important: Run "make" to regenerate code after modifying this file
} 
 
// AtStatus defines the observed state of At
type AtStatus struct { 
    // Phase represents the state of the schedule: until the command is executed
    // it is PENDING, afterwards it is DONE.
    Phase string `json:"phase,omitempty"` 
    // Important: Run "make" to regenerate code after modifying this file
} 
 
// +genclient
// +k8s:deepcopy-gen:interfaces=k8s.io/apimachinery/pkg/runtime.Object
 
// At runs a command at a given schedule.
type At struct { 
    metav1.TypeMeta   `json:",inline"` 
    metav1.ObjectMeta `json:"metadata,omitempty"` 
 
    Spec   AtSpec   `json:"spec,omitempty"` 
    Status AtStatus `json:"status,omitempty"` 
} 
 
// +k8s:deepcopy-gen:interfaces=k8s.io/apimachinery/pkg/runtime.Object

// AtList contains a list of At
type AtList struct { 
    metav1.TypeMeta `json:",inline"` 
    metav1.ListMeta `json:"metadata,omitempty"` 
    Items           []At `json:"items"` 
}
```

## 3 deepcopy-gen 标签

通常，我们会使用 `// +k8s:deepcopy-gen=package` 默认为所有类型都生成 DeepCopy() 方法。

当然，我们可以为一个**特定的类型禁止生成 DeepCopy() 方法**。
```go
// +k8s:deepcopy-gen=false
//
// Helper is a helper struct, not an API type.
type Helper struct { 
    ... 
}
```

## 4 runtime.Object 与 DeepCopyObject
```go
// +k8s:deepcopy-gen:interfaces=k8s.io/apimachinery/pkg/runtime.Object
```

前面提到，runtime.Object 必须实现 DeepCopyObject() 方法，而该标签为类型生成对应的 DeepCopyObject 方法。

我们应该为**所有的 ”顶级对象“（内嵌了 metav1.TypeMeta）使用该标签**。

## 5 client-gen 标签

**`// +genclient`** 标签用于告诉 client-gen 需要**为这个类型创建一个 client**。但是，你不需要也不能把这个标签用到 List 类型上。
```go
// +genclient
```

如果有些类型没有 status 字段，或者没有进行 spec-status 分离。这些情况下，可以使用 **`// +genclient:noStatus`** 标签来避免生成 UpdateStatus() 方法。
```go
// +genclient:noStatus
```
{{< admonition note Note>}}
如果没有这个标签，client-gen 会直接生成 UpdateStatus() 方法。但是，只有使用了 /status 子资源，才会分离出 /status。

**如果没有开启 /status 子资源，调用该接口只会直接返回错误。**
{{< /admonition >}}

对于集群范围的资源（不属于 namespace），可以使用下面标签：
```go
// +genclient:noNamespaced
```

如果你想要控制 client 能够访问的 HTTP 方法，可以通过一系列标签指定：
```go
// +genclient:noVerbs
// +genclient:onlyVerbs=create,delete
// +genclient:skipVerbs=get,list,create,update,patch,delete,watch
// +genclient:method=Create,verb=create,
// result=k8s.io/apimachinery/pkg/apis/meta/v1.Status
```
大多数看名字就能理解，最后需要一些解释：指定了 L5 标签后，生成的 Create 方法中只会执行创建动作，并返回一个 metav1.Status 对象，而不是它自身的 API 类型对象。对于自定义 APIServer 来说，可能会用到这样的资源。

使用 `// +genclient:method` 的常见场景是为资源增加 scale 相关方法。在启动 /scale 子资源后，下面标签就可以生成对应的 client 方法。
```go
// +genclient:method=GetScale,verb=get,subresource=scale,\
//    result=k8s.io/api/autoscaling/v1.Scale
// +genclient:method=UpdateScale,verb=update,subresource=scale,\
//    input=k8s.io/api/autoscaling/v1.Scale,result=k8s.io/api/autoscaling/v1.Scale
```

## 6 informer-gen 和 lister-gen

informer-gen 和 lister-gen 都会处理 client-gen 的 `// +genclient` 标签，**所有指定了生成客户端的类型都会自顶生成相应的 Informer 和 Lister**。


## 参考
* [**《Programming Kubernetes》**](https://book.douban.com/subject/33393681/)



