# K8s 学习 - RBAC 授权机制


## 1 概念
在 [**API 访问控制**](http://kanshiori.cn/posts/cloud_computing/k8s_learning/api-%E6%A6%82%E5%BF%B5/#52-authorization) 中提到，Kubernetes 支持的授权机制有多种，其中 RBAC 是最常用的授权方式。RBAC 基于角色访问控制，全称 **`Role-Base Access Control`**。

对于理解 RBAC，首先需要知道三个关键的概念：
* **`Role`** ：角色，代表一组对 Kubernetes API 对象操作的权限。
* **`Subject`** ：被作用者，包括 User、Group、ServiceAccount；
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
* rules
  * `apiGroups` ：表明针对哪些 API Group 生效，为空表示 core API Group；
  * `resources` ：表明针对哪些 Resource 生效；
  * `verbs` ：允许执行的操作；

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
* rules
  * `apiGroups` ：表明针对哪些 API Group 生效，为空表示 core API Group；
  * `resources` ：表明针对哪些 Resource 生效；
  * `verbs` ：允许执行的操作；

示例中 rules 定义的规则是：允许对所有 namespace 下的 所有 Secret 对象，执行 GET、Watch、List 操作。

### 2.3 内置 Role 与 ClusterRole
默认下，Kubernetes 已经内置了许多个 Role 与 ClusterRole，它们都是以 `system:` 开头命名。

使用内置 Role 与 ClusterRole 已经完全可以满足大部分的使用场景。

内置的 Role 的含义见 [**Default roles and role bindings**](https://kubernetes.io/docs/reference/access-authn-authz/rbac/#default-roles-and-role-bindings)。


## 3 Subject
Subject 表明 Role 作用的对象，可以是 **Group**、**User** 或者 **ServiceAccount**。

### 3.1 ServiceAccount
ServiceAccount 应该是最常用的 Subject，**针对于 Pod 中的进程使用，控制其调用 Kubernetes API 或者外部服务的权限**。
{{< admonition note Note>}}
一定要明确 ServiceAccount 对为了 Pod 中进程而设计的
{{< /admonition >}}

一个 ServiceAccount 的基本定义如下：
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: local-storage-admin
  namespace: kube-system
```

一般情况下，不需要指定其他的参数，仅仅创建一个 ServiceAccount 即可。创建后，**自动会为其创建一个 Secret 对象**。
```shell
$ kubectl get serviceaccounts  local-storage-admin -n kube-system  -o yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  creationTimestamp: "2021-06-04T09:24:42Z"
  name: local-storage-admin
  namespace: kube-system
  resourceVersion: "106213"
  uid: 1a073d5c-4b1f-409a-a3ff-b54af945404f
secrets:
- name: local-storage-admin-token-4v6g8
```

其 Secret 对象中包含三个数据：
```yaml
apiVersion: v1
data:
  ca.crt:  # …
  namespace: a3ViZS1zeXN0ZW0=
  token:  # …
kind: Secret
metadata:
  annotations:
    kubernetes.io/service-account.name: local-storage-admin
    kubernetes.io/service-account.uid: 1a073d5c-4b1f-409a-a3ff-b54af945404f
  creationTimestamp: "2021-06-04T09:24:42Z"
  name: local-storage-admin-token-4v6g8
  namespace: kube-system
  resourceVersion: "106211"
  uid: ccd3494c-34c7-48ab-8cb9-3b944d980fe3
type: kubernetes.io/service-account-token
```
* **ca.crt** ：基于 Kubernetes 根证书签发的证书，也就是 Kubernetes 认可的证书；
* **namespace** ：token 所属的 namespace；
* **token** ：访问需要的 token；

也就是说，基于这个 Secret 对象的内容，我们可以有权限去访问 APIServer。
{{< admonition tip "Token Controller">}}
Token Controller 会检测 ServiceAccount 的创建，并为其创建 Secret
{{< /admonition >}}

#### 3.1.1 使用 ServiceAccount
Pod 定义中通过 `sepec.serviceAccountName` 可以指定使用的 ServiceAccount（默认 default）。

Pod 提交后，自动会为其添加相关的 Volume 挂载：
```yaml
spec:
  containers:
   #...
    volumeMounts:
    - mountPath: /var/run/secrets/kubernetes.io/serviceaccount
      name: kube-api-access-k96lm
      readOnly: true
  volumes:
  - name: kube-api-access-k96lm
    projected:
      defaultMode: 420
      sources:
      - serviceAccountToken:
          expirationSeconds: 3607
          path: token
      - configMap:
          items:
          - key: ca.crt
            path: ca.crt
          name: kube-root-ca.crt
      - downwardAPI:
          items:
          - fieldRef:
              apiVersion: v1
              fieldPath: metadata.namespace
            path: namespace
```

每个 container 启动后，在 `/var/run/secrets/kubernetes.io/serviceaccount/` 目录下就会存在着对应的三个文件：
```shell
$ kubectl exec nginx-3137573019-md1u2 ls /var/run/secrets/kubernetes.io/serviceaccount
ca.crt
namespace
token
```

如果我们的 ServiceAccount 已经通过 RoleBinding 授予了相关的权限，那么就可以进行证书与 token 访问 APIServer：
```shell
$ TOKEN=$(cat token) && curl "https://kubernetes.default/api/v1/namespaces/mycluster/pods/"  --cacert ./ca.crt  -H "Authorization: Bearer $TOKEN"
{
"kind": "PodList",
  "apiVersion": "v1",
  "metadata": {
    "resourceVersion": "1589287"
  },
  "items": [
 // …
```

#### 3.1.1 default ServiceAccount
每个 namespace 创建时，会自动创建一个名为 "default" 的 ServiceAccount 对象。
```shell
$ kubectl get serviceaccounts -A | grep default
NAMESPACE  NAME                 SECRETS          AGE
default           default                              1         5d22h
kube-node-lease   default                              1         5d22h
kube-public       default                              1         5d22h
kube-system       default                              1         5d22h
mycluster         default                              1         5d
mytest            default                              1         2d
tidb-admin        default                              1         5d
```

因为默认不会创建 RoleBinding 来绑定 default ServiceAccount 与 Role，所以默认其实没有任何权限。
	

## 4 RoleBinding 与 ClusterRoleBinding

### 4.1 RoleBinding
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

### 4.2 ClusterRoleBinding
ClusterRoleBinding 让 Subject 有着整个集群相关资源的访问权限。
#### 4.2.1 ClusterRole 聚合
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


