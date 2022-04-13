# Kubernetes RBAC 授权机制


## 1 概念

在 [**Kubernetes 认证与鉴权机制**](../authentication-and-authorization/) 中提到，Kubernetes 支持的授权机制有多种，其中 RBAC 是最常用的授权方式。RBAC 基于角色访问控制，全称 **`Role-Base Access Control`**。

对于理解 RBAC，首先需要知道三个关键的概念：

* **`Role`** ：角色，代表一组对 Kubernetes API 对象操作的权限。

* **`Subject`** ：被作用者，包括 User、Group、ServiceAccount

* **`RoleBinding`** ：定义 Role 与 Subject 的映射关系；

因此，**我们会预先创建一些 Role，然后创建 Subject 时，定义 RoleBinding 来表明对 Subject 的权限控制。**

{{< admonition note "为什么这么设计？">}}
通过 RoleBinding，实现了 Role Subject 之间的解耦。
{{< /admonition >}}

## 2 Role 与 ClusterRole

### 2.1 Role

Role 就是一个 Kubernetes 资源对象，并且是一个 namespaced Resouce，因此其 Role 控制的权限范围也只能是所属的 namespace 下。

基本定义如下：

```yaml
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  namespace: default
  name: pod-reader
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "watch", "list"]
```
* `metadata.namespace` ：指定 Role 所属的 namespace
* `rules.apiGroups` ：表明针对哪些 API Group 生效，为空表示 core API Group；
* `rules.resources` ：表明针对哪些 Resource 生效；
* `rules.verbs` ：允许执行的操作；

示例中 rules 定义的规则就是：允许对 "mynamespace" 下的所有 Pod 对象，执行 GET、Watch、List 操作。

能够支持限制的操作为："get", "list", "watch", "create", "update", "patch", "delete"，"deletecollection" 。

### 2.2 ClusterRole

ClusterRole 是一个 non-namespaced Resource，所以是针对所有 namespace 的资源生效，也可以实现控制 non-namespaced Resource 的权限。

基本定义如下：

```yaml
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  # "namespace" omitted since ClusterRoles are not namespaced
  name: secret-reader
rules:
- apiGroups: [""]
  resources: ["secrets"]
  verbs: ["get", "watch", "list"]
```
* `rules.apiGroups` ：表明针对哪些 API Group 生效，为空表示 core API Group；
* `rules.resources` ：表明针对哪些 Resource 生效；
* `rules.verbs` ：允许执行的操作；

示例中 rules 定义的规则是：允许对所有 namespace 下的 所有 Secret 对象，执行 GET、Watch、List 操作。

#### 2.2.1 聚合 ClusterRole

多个 ClusterRole 可以聚合为一个新的 ClusterRole，用于简化 ClusterRole 的管理工作。

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: monitoring
aggregationRule:
  clusterRoleSelectors:
  - matchLabels:
      rbac.example.com/aggregate-to-monitoring: "true"
rules: [] # rules 规则会自动被设置
```
`aggregationRule` 字段设置了匹配 ClusterRole 的规则，当一个 ClusterRole 被匹配到后，其 rules 自动会填充相关的规则。

### 2.3 对非资源对象限制

某些 Kubernetes API 包含下次子资源，例如 Pod 的日志。

```yaml
# ...
rules:
- apiGroups: [""]
  resource: ["pods", "pods/log"]
  verbs: ["get", "list"]
```

### 2.4 对指定资源对象限制

通过 `rules.resourceName` 字段可以限制对指定资源的权限：

```yaml
# ...
rules:
- apiGroups: [""]
  resource: ["configmap"]
  resourceName: ["mu-configmap"]
  verbs: ["update", "get"]
```

当然，因为是对指定的资源，因此无法限制 list、watch、create 或 deletecollections 操作。

### 2.5 内置 Role 与 ClusterRole

默认下，Kubernetes 已经内置了许多个 Role 与 ClusterRole，它们都是以 `system:` 开头命名。

所有系统默认的 ClusterRole 和 RoleBinding 都会使用 label `kubernetes.io/bootstrappiong=rbac-defaults` 来标记。

每次集群启动时，APIServer 会更新默认的集群 Role 缺失的权限，也会更新默认 Role 绑定的 Subject。这样可以防止一些破坏性的更改，保证集群升级情况下相关内容能够及时更新。

使用内置 Role 与 ClusterRole 已经完全可以满足大部分的使用场景。

内置的 Role 的含义见 [**Default roles and role bindings**](https://kubernetes.io/docs/reference/access-authn-authz/rbac/#default-roles-and-role-bindings)。


## 3 RoleBinding 与 ClusterRoleBinding

### 3.1 RoleBinding

RoleBinding 用于绑定一个 Role 与多个 Subject，通过 RoleBinding 才能让某个 Subject 有 namespace 相关资源的访问权限。

```yaml
# This role binding allows "jane" to read pods in the "default" namespace.
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: read-pods
  namespace: default
subjects:
- kind: User
  name: jane
  apiGroup: rbac.authorization.k8s.io
- kind: ServiceAccount
    name: default
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

{{< admonition tip Tip>}}
RoleBinding 也可以引用 ClusterRole，将其权限限制在了 RoleBinding 所属的 namespace 下。
{{< /admonition >}}

### 3.2 ClusterRoleBinding

ClusterRoleBinding 让 Subject 有着整个集群相关资源的访问权限。

#### 3.2.1 ClusterRole 聚合

在 ClusterRole 中可以通过 aggregationRule 来与其他 ClusterRole，也就是可以自动组合多个 ClusterRole 为一个。
```yaml
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: monitoring
aggregationRule:  # 通过 label selector 方式自动组合
  clusterRoleSelectors:
  - matchLabels:
      rbac.example.com/aggregate-to-monitoring: "true"
rules: [] # Rules 由组合起来的 ClusterRole 自动填充
---
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: monitoring-endpoints
  labels:
    rbac.example.com/aggregate-to-monitoring: "true"
# These rules will be added to the "monitoring" role.
rules:
- apiGroups: [""]
  resources: ["services", "endpoints", "pods"]
  verbs: ["get", "list", "watch"]
```

## 参考

* 官方文档：[**使用 RBAC 鉴权**](https://kubernetes.io/zh/docs/reference/access-authn-authz/rbac/)


