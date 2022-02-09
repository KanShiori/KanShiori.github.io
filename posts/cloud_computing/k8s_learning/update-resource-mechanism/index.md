# Kubernetes 资源更新机制与 CLI 实现


## 1 Update 机制

**`Update 机制`** 指的是通过 <important>HTTP Put 方法</important> 上传**完整的资源对象**，而 Kubernetes 使用完整的资源对象更新原来的资源对象。在更新对象时，Kubernetes 通过 resourceVersion 来检查是否发生冲突。

`metadata.resourceVersion` 可以代表资源的**版本号**，本质上是由 etcd 提供 modified index。在对象写入 etcd 后，resourceVersion 都会发生变化。

{{< admonition tip "resourceVersion 无法比较大小">}}
[**官方文档**](https://kubernetes.io/zh/docs/reference/using-api/api-concepts/#resource-versions)中指出，resourceVersion 仅仅只能用来比较是否相等，而没有大小之分。

也就是说，不能按照 resourceVersion 的大小来判断对象的新旧。
{{< /admonition >}}

写入对象到 etcd 时，**会比较对象的 resourceVersion 是否与当前存储的对象 resourceVersion 相等**，如果不相等就表明此次更新发生冲突。

{{< admonition note "代码中的 Update()">}}
Controller 中使用的 Update() 接口更新资源，都是 Update 机制，返回冲突错误也就是遇到了 resourceVersion 不相等的情况。
{{< /admonition >}}

## 2 Patch 机制

**`Patch 机制`** 指的是通过 <important>HTTP Patch 方法</important> 上传**增量的对象字段**（除了 Server Side Apply 是上传完整对象）。

Kubernetes 会直接接受 Patch 请求，将 patch 打到对象上，同时更新 resourceVersion。也就是说，<important>Patch 机制默认没有冲突机制</important>（除了 Server Side Apply）。

目前有着 4 种 Patch 的方式，各种方式通过 HTTP Content-Type Header 来区分：
* JSON Patch
* Merge Patch
* Strategic Merge Patch
* Apply Patch

### 2.1 JSON Patch

[**JSON Patch**](http://jsonpatch.com/) 是一种标准的 Patch 协议（RFC 6901），使用 "Content-Type: application/json-patch+json" 标识。

JSON Patch 存在一些问题，例如在修改数组时，需要通过编号来指定元素。

```json
[
    {
        "op": "replace",
        "path": "/spec/template/spec/containers/0/image",
        "value": "app-image:v2"
    }
]
```

但是在多个同时修改一个对象的情况下，因为没有冲突机制，所以会导致不可预期的结果。例如当该请求发出后，该对象已经修改了数组，那么该 0 编号的元素很可能变化了，从而该请求导致了非期望的修改。

### 2.2 Merge Patch

[**Merge Patch**](http://mirrors.nju.edu.cn/rfc/inline-errata/rfc7386.html) (也称为 JSON Merge Patch）是一种标准的 Patch 协议（RFC 7386），使用 "Content-Type: application/merge-patch+json" 标识。

Merge Patch 直接上传需要修改的字段，上传的字段都是增加或更新。如果需要删除某个字段，将上传值设为 null，表示删除。

{{< admonition warning "缺陷">}}
1. 不允许单独修改数组中的某个元素，需要提供整个数组来进行修改
2. 无法表示 null 值。
{{< /admonition >}}

例如，下面请求表明直接覆盖当前对象的 container，而不能单独修改数组。

```json
{
    "spec": {
        "template": {
            "spec": {
                "containers": [
                    {
                        "name": "app",
                        "image": "app-image:v2"
                    },
                    {
                        "name": "nginx",
                        "image": "nginx:alpline"
                    }
                ]
            }
        }
    }
}
```

可见，Merge Patch 不适合对列表深层的字段做更新。

### 2.3 Strategic Merge Patch

**`Strategic Merge Patch`** 是 Kubernetes 设计的一种 Patch 方式，通过与字段数据结构的一些注释，指明数组中元素的 key，以解决 Merge Patch 的一些问题。

例如，下面是 Pod 的 Container 字段的源码，其中 `patchMergeKey=name` 表明使用 container.name 字段为元素的 key。

```go
// ...
// +patchMergeKey=name
// +patchStrategy=merge
Containers []Container `json:"containers" patchStrategy:"merge" patchMergeKey:"name" protobuf:"bytes,2,rep,name=containers"` 
```

知晓了数组元素的 key 后，可以将数组类似于 map 了，这样就可以在 Merge Patch 基础上更新某个数组中的元素了。

下面的请求仅仅只会更新 nginx container 配置，而不会影响到对象中已有的 container 配置。
```json
{
    "spec": {
        "template": {
            "spec": {
                "containers": [
                    {
                        "name": "nginx",
                        "image": "nginx:mainline"
                    }
                ]
            }
        }
    }
}
```

可以看到，使用 Strategic Merge Patch 方式需要 Kubernetes 知晓其数据结构的注释，**因此只能应用于 Kubernetes 原生的资源对象。对于 Custom Resource 是无法使用的**。

### 2.4 Server Side Apply

Kubernetes 1.16 之后，默认开启了 [**Server Side Apply**](https://kubernetes.io/zh/docs/reference/using-api/server-side-apply/) 特性，在前面三种 Apply 方式的基础上，为 Patch 机制提供了冲突检查的能力。

#### 2.4.1 字段管理

Server Side Apply 将对象的每个字段都认为由一个 [**Manager**](https://kubernetes.io/zh/docs/reference/using-api/server-side-apply/#managers) 来管理。

使用 Server Side Apply 想要更新对象的某个字段时，如果该字段是由其他 Manager 管理时，就会触发 [**Conlict**](https://kubernetes.io/zh/docs/reference/using-api/server-side-apply/#conflicts)。这表明，**该字段的所有权是属于其他 Manager 的**。

{{< admonition note Note>}}
Conflict 只在使用 Server Side Apply 时触发，并且字段管理的特性是默认开启的，因此其他任何更新对象的操作会直接替换字段的 Manager。
{{< /admonition >}}

字段管理的信息记录在 `metadata.managedFields` 中：

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: test-cm
  namespace: default
  labels:
    test-label: test
  managedFields:
  - manager: kubectl
    operation: Apply
    apiVersion: v1
    time: "2010-10-10T0:00:00Z"
    fieldsType: FieldsV1
    fieldsV1:
      f:metadata:
        f:labels:
          f:test-label: {}
      f:data:
        f:\key: {}
data:
  key: some value
```

例如上述字段表明，目前 `data.key` 和 `metadata.labels.test-label` 是由 kubectl 管理的，更新字段的操作是 Apply。

如果多个 Manager 将同一个字段设为相同的值，那么他们共享此字段的所有权，称为 **`Shared Manager`**。**后续任何 Manager 尝试改变字段值时都会触发 Conflict**。

managedFields 字段的格式说明见 [**API 文档**](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.23/#fieldsv1-v1-meta)。

发送任何更新对象的 HTTP 请求时，可以通过 `fieldManager` query 标识 Manager（对于 Apply 操作是必须指定的）。

#### 2.4.2 Merge 策略

Server Side Apply 直接使用 Strategic Merge Patch 和 Merge Patch 方式来更新对象。上传完整的请求资源对象后，APISever 读取当前保存的对象进行 diff，之后通过 Strategic Merge Patch 或者 Merge Patch 执行 Patch 操作。

{{< admonition note Note>}}
Server Side Apply 与 `kubectl apply` 类似，只不过 diff 行为发生在 APISever。
{{< /admonition >}}

#### 2.4.3 Conflict 处理

当字段的更新触发 Conflict 时，有三种处理方式：

* **强制更新，转移所有权**：使用 **`force query`** 参数，再发送一次请求。**这会强制更新资源，并将更新的字段的 Manager 变更为当前请求的 Manager**。
* **放弃修改**：在请求中去除字段，表明不更改其字段。
* **成为 Shared Manager**：请求中字段设为当前值，不更改字段，但是会成为 Shared Manager。

## 3 CLI 实现

### 3.1 kubectl apply

#### 3.1.1 client side apply

`kubectl apply` 默认执行的是 client side apply，是基于对象的 `last-applied-configuration` annotation 进行 diff。

总的来说 `kubectl apply` 分为三种情况：

* **apply 的对象当前不存在** - 执行 POST HTTP 创建对象，并设置 last-applied-configuration annotation。
* **apply 的对象存在，对象为 Kubernetes 原生对象** - 执行 PATCH HTTP，Patch 类型为 strategic merge patch。
* **apply 的对象存在，对象不是 Kubernetes 原生对象** - 执行 PATCH HTTP，Patch 类型为 merge patch。

对象不存在，`kubectl apply` 等价于 `kubectl create` 命令，仅仅多设置了 last-applied-configuration annotation，为后续更新做准备。

对象存在，`kubectl apply` 等价与 `kubectl patch` 命令，区别在于 `kubectl apply` 输入是一个完整的对象，命令通过 `last-applied-configuration` annotation 得到上一次的对象配置，然后进行 diff 生成 Patch 请求。

#### 3.1.2 server side apply

`kubectl apply` 加上 `--server-side` 参数时使用的就是 server side apply。使用 `--force-conflicts` 参数可以强制更新，使用 `--field-manager` 参数可以指定 Manager 名字（默认为 kubectl）。

server side apply 的过程是完全交给 APIServer 处理，kubectl 不需要读取 annotation 进行 diff，直接通过 PATCH HTTP 提交完整的对象即可，由 APISever 来进行 diff 操作，并修改资源。

### 3.2 kubectl patch

`kubectl patch` 发送的是 Patch 请求，只需要上传增量的数据（等于说是由用户做 Diff 操作），默认使用 [**Strategic Merge Patch**](#23-strategic-merge-patch) 类型。可以使用 `--type` 来指定 Patch 类型。

### 3.3 kubectl edit

`kubectl edit` 比 `kubectl apply` 更加简单，因为不需要通过 annotation 读取变化，而是直接 diff 编辑前后的对象来生成 Patch 请求。

当然，Patch 类型也是基于对象是否是 Kubernetes 原生的决定的。

## 参考

* [**官方文档：服务器端应用（Server-Side Apply）**](https://kubernetes.io/zh/docs/reference/using-api/server-side-apply/)
* [**Blog：K8s 资源更新机制**](https://developer.aliyun.com/article/763212)
