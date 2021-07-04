# K8s 学习 - API Server 认证


## 1 概述
Kubernetes 中所有的资源访问与操作都是通过 API Server 执行的，因此 API Server 有着一套安全机制。

APIServer 的权限检查从大体上分为三个阶段：
* **Authentication** ：身份认证，用于验证请求者的身份是否合法。
* **Authorization** ：权限认证，验证请求者是否包含对应资源的控制权限。
* **Admission Control** ：可扩展的通用检查。


## 2 Subject
客户端访问 API Server 通常包含三种途径：kubectl、client 库、REST **API。而这些此类请求的对象通常为：UserAccount** 和 **ServiceAccount**。

{{< admonition note Note>}}
UserAccount 与 ServiceAccount 仅仅是身份，而需要通过授权才能访问 APIServer。
{{< /admonition >}}

### 2.1 UserAccount
**`UserAccount`** 是独立于 Kubernetes 之外的用户账号，包括
* 负责分发私钥的管理员
* 类似 Keystone 或者 Google Account 这类的数据库
* 包含用户名密码的文件

Kubernetes 不会存储用户的信息，而是通过其 CA 证书来进行身份限制。证书中的 Subject Common Name 会被作用用户名，例如 "CN=bob"。

#### 2.1.1 创建 UserAccount
下面尝试新建一个 User，让其能够访问 Kubernetes:
1. 使用 Kubernetes 的根证书签发 UserAccount 使用证书：

    ```shell
    $ openssl genrsa -out shiori.key 2048
    $ openssl req -new -key shiori.key -out shiori.csr -subj "/CN=shiori"
    $ openssl x509 -req -in shiori.csr -CA /etc/kubernetes/pki/ca.crt -CAkey /etc/kubernetes/pki/ca.key -CAcreateserial -out shiori.crt -days 3650
    ```
2. 在 Kubernetes 中创建一个新的用户，传入对应的证书与私钥

    ```shell
    $ kubectl config set-credentials shiori --client-certificate=./shiori.crt --client-key=./shiori.key --embed-certs=true
    User "shiori" set.
    ```
3. 创建一个 context，指定其用户。这样我们接下来来使用的就是新用户来访问 APIServer 了。
    
    ```shell
    $ kubectl config set-context shiori@kubernetes --cluster=kubernetes --user=shiori
    $ kubectl config use-context shiori@kubernetes
    ```

### 2.2 ServiceAccount
**`ServiceAccount`** 是 Kubernetes 管理的账号，**提供给 Pod 里的进程使用，为 Pod 里的进程提供身份证明**。
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
spec:
  automountServiceAccountToken: false
```
* `spec.automountServiceAccountToken` 表明不进行自动创建对应 Secret 对象

#### 2.2.1 Service Account Atuh
在 Pod 中访问 Kubernetes APIServer 时，会使用 Kubernetes 内置的 HTTP Token 认证方式 Service Account Auth。
{{< admonition note Note>}}
使用 [**HTTP Bearer Token**](#32-http-bearer-token-认证) 的方式
{{< /admonition >}}

* 对于 APIServer 的认证，**Pod 内进程使用下发的根证书，来验证 APIServer 的证书是否合法**。
    
    根证书数据挂载到 Pod 内 `/run/secrets/kubernetes.io/serviceaccount/ca.crt`，可以看到其和 APIServer 启动使用的根证书内容是一致的。
    ```shell
    $ cat /etc/kubernetes/pki/ca.crt

    $ kubectl exec -it ${POD} -c ${CONTAINER} -- cat /run/secrets/kubernetes.io/serviceaccount/ca.crt
    ```

* 对于 Pod 内进程的认证，**通过 HTTP Header 中的 Token 字符串数据，来验证 Pod 内进程**。
    
    Token 为Kubernetes Controller 进程用 APIServer 的私钥（*--service-account-private-key-file* 指定）生成的一个 JWT Secret。

    数据会挂载到 Pod 内 `/run/secrets/kubernetes.io/serviceaccount/token` 文件。

可以看到，使用 ServiceAccount 时涉及到的 Pod 内的三个文件：
* `/run/secrets/kubernetes.io/serviceaccount/ca.crt`
* `/run/secrets/kubernetes.io/serviceaccount/token`
* `/run/secrets/kubernetes.io/serviceaccount/namespace`（Client 使用该 namespace 作为调用 API 的参数）
```shell
$ kubectl exec -it ${POD} -c ${CONTAINER} -- ls /run/secrets/kubernetes.io/serviceaccount
ca.crt     namespace  token
```

#### 2.2.2 ServiceAccount 与 Secret 对象
上述的文件都是 Kubernetes Secret 对象传递的，当创建一个 ServiceAccount 时，**自动会创建对应的 Secret 对象**。而 Secret 对象就包含这三个数据。
```shell
$ kubectl get secrets
NAME                                             TYPE                                  DATA   AGE
default-token-4l585                              kubernetes.io/service-account-token   3      16d

$ kubectl get secrets default-token-4l585  -o yaml
apiVersion: v1
data:
  ca.crt: ...
  namespace: ...
  token: ...
kind: Secret
metadata:
  annotations:
    kubernetes.io/service-account.name: default
    kubernetes.io/service-account.uid: a4d19cee-45e3-41ac-af66-ca8b070fa5d9
  creationTimestamp: "2021-06-16T09:25:37Z"
  name: default-token-4l585
  namespace: tidb-cluster-dev
  resourceVersion: "3996078"
  uid: a068d2ad-79b9-4b3c-be8c-c6e245c1865f
type: kubernetes.io/service-account-token
```

可以看到，对应的 Secret 对象名为 `<service_account>-token-<random>`，type 为 `kubernetes.io/service-account-token`。Secret 中也通过两个 annotations 来表明其所属的 ServiceAccount。

#### 2.2.3 使用 ServiceAccount
Pod 定义中通过 `spec.serviceAccountName` 可以指定使用的 ServiceAccount（默认 default）。
```yaml
spec:
  # ...
  serviceAccountName: myserviceaccount
```

接着，对应的文件就会被挂载到 Pod 内，那么我们使用 Client 库或者 HTTP 访问等方式，就可以访问 APIServer 了：
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

## 3 Authentication
APIServer 安全的第一步检查是 Authentication，用于验证请求的身份。

目前包含以下身份验证方式：
* **X509 客户证书认证**：基于 Kubernetes 根证书签名的双向认证方式
* **HTTP Bearer Token 认证**：通过 Bearer Token 识别合法用户
* **OpenID Connect Token 第三方认证**：通过第三方 OIDC 协议进行认证
* **Webhook Token 认证**：通过外部 Webhook 服务进行认证
* **Authenticating Proxy 认证**：通过认证代理程序进行认证

当开启了多个身份认证模块时，只需要任一模块认证通过。APISever 不会保证身份认证模块的运行顺序。

### 3.1 X509 客户证书认证
通过 APIServer 启动参数 *"--client-ca-file=SOMEFILE"*，可以启动客户端证书验证方式。**其参数指定了用于验证客户端证书的根证书文件**。

证书验证通过后，客户端证书中 **Subject Common Name 会被用作请求的用户名**，用于后续的授权检查。

### 3.2 HTTP Bearer Token 认证
通过 APIServer 启动参数 *"--token-auth-file=SOMEFILE"*，指定其**使用的静态 Token 文件**。文件为 CSV 文本格式，每行字段为：
```shell
# token,user,uid[,groupnames]
31ada4df-abedx-a11z-123z-124sgtszxvr3,join,2,"group1,group2"
```
* token - Token 字符串
* user - 用户名
* uid - 用户 ID
* groupnames - 用户组

通过 HTTP 访问 APIServer 时，其 HTTP Header 中 Token 字段来表明客户身份。APIServer 通过读取该 Token 来保存的 Token 中进行验证，就可以检查其身份，并知晓对应的用户了。
{{< admonition note ServiceAccount>}}
ServiceAccount 就是使用 Bearer Token 方式认证，只是其 Token 是由 Kubernetes 动态生成的。
{{< /admonition >}}

### 3.3 OpenID Connect Token 第三方认证
Kubernetes 也支持使用 **`OpenID Conntect 协议（简称 OIDC）`** 进行身份验证，不过没有使用 OIDC 的权限管理。

**用户通过 OIDC Sever 得到一个合法的 ID Token，请求时传递给 APIServer。APIServer 通过验证该 Token 是否合法来验证用户的身份。**
{{< find_img "img2.png" >}}

要使用 OIDC Token 认证方式，APIServer 需要配置一些参数，见 [**配置 API 服务器**](https://kubernetes.io/zh/docs/reference/access-authn-authz/authentication/#configuring-the-api-server)

### 3.4 Webhook Token 认证
Kubernetes 也支持通过外部 Webhook 认证服务器，配置 HTTP Bearer Token 来实现自定义的用户身份认证功能。

其工作步骤如下：
1. 开启并配置 API Server 的 Webhook Token Authentication 功能。
2. **APIServer 接受到一个需要认证的请求后，从 HTTP Header 中取出 Token 信息，然后将包含该 Token 的 TokenReview 资源以 HTTP POST 方法发送到远程 Webhook 服务进行认证。**
3. APIServer 根据 Webhook 返回的结果判断是否认证成功。

远端 Webhook 服务回复也需要是一个 **`TokenReview`** 资源对象，并且 apiVersion 要与 APIServer 发送的 apiVersion 一致。

要使用 Webhook Token 认证方式，APIServer 需要以下启动参数：
* *"--authentication-token-webhook-config-file"*: 指向一个配置文件，文件描述了如何访问远程的 Webhook 服务
* *"--authentication-token-webhook-cache-ttl"*: 缓存 Webhook 服务返回的结果的时间，默认为 2min
* *"-authentication-token-webhook-version"*: 发送给 Webhook 服务的 TokenReview 资源 API 版本号，"v1beta1" 或 "v1"

配置文件使用 kubeconfig 文件格式：
```yaml
apiVersion: v1
kind: Config
clusters:
  - name: name-of-remote-authn-service
    cluster:
      certificate-authority: /path/to/ca.pem         # 用来验证远程服务的 CA
      server: https://authn.example.com/authenticate # 要查询的远程服务 URL。必须使用 'https'。
users:
  - name: name-of-api-server
    user:
      client-certificate: /path/to/cert.pem # Webhook 插件要使用的证书
      client-key: /path/to/key.pem          # 与证书匹配的密钥
# kubeconfig Context
current-context: webhook
contexts:
- context:
    cluster: name-of-remote-authn-service
    user: name-of-api-sever
  name: webhook
```
* `clusters`: 设置 Webhook 远程服务
* `users`: APIServer 使用的 Webhook 配置

APIServer 是收到请求，提取出 Token 后，将发送如下 TokenReview 资源对象并发送。
```json
{
  "apiVersion": "authentication.k8s.io/v1beta1",
  "kind": "TokenReview",
  "status": {
    "authenticated": true,
    "user": {
      "username": "janedoe@example.com",
      "uid": "42",
      "groups": [
        "developers",
        "qa"
      ],
      "extra": {
        "extrafield1": [
          "extravalue1",
          "extravalue2"
        ]
      }
    }
  }
}
```

回复中通过 `status.authenticated` 字段来表明是否授权成功：
```json
{
  "apiVersion": "authentication.k8s.io/v1beta1",
  "kind": "TokenReview",
  "status": {
    "authenticated": true,
    "user": {
      "username": "janedoe@example.com",
      "uid": "42",
      "groups": [
        "developers",
        "qa"
      ],
      "extra": {
        "extrafield1": [
          "extravalue1",
          "extravalue2"
        ]
      }
    }
  }
}
```
```json
{
  "apiVersion": "authentication.k8s.io/v1beta1",
  "kind": "TokenReview",
  "status": {
    "authenticated": false
  }
}
```

### 3.5 Authenticating Proxy
APIServer 可以配置为从 HTTP Header（例如 X-Remote-User）来进行用户身份验证。由 Authenticating Proxy 程序设置 HTTP Header 相关的值。

首先，Authenticating Proxy 需要向 APIServer 配置对应的客户端 CA 证书，保证 Proxy 能与 APIServer 进行 HTTPS 连接。

通过以下 APIServer 启动参数配置：
* *--requestheader-client-ca-file*: Authenticating Proxy 客户端 CA 证书文件路径
* *--requestheader-allowed-names*: （可选）设置允许的 Common Name 列表
* *--requestheader-username-headers*: 设置用户名的 Header，通常为 "X-Remote-User"
* *--requestheader-group-headers*： 设置用户组的 Header，通常为 "X-Remote-Group"
* *--requestheader-extra-headers-prefix*: Header 字段前缀用于确定用户一些其他信息，通常为 "X-Remote-Extra-"

HTTPS 连接建立后，APIServer 才会校验 HTTP Header 中设置的用户名。一个请求类似于：
```http
GET / HTTP/1.1
X-Remote-User: fido
X-Remote-Group: dogs
X-Remote-Group: dachshunds
X-Remote-Extra-Acme.com%2Fproject: some-project
X-Remote-Extra-Scopes: openid
X-Remote-Extra-Scopes: profile
```

### 3.6 Anonymous Requests
**请求中没有被任何已配置的身份认证方法拒绝，那么会被视为** **`Anonymous Requests`**。这类请求会被视为用户 `system:anonymous` 和 对应的用户组 `system:unauthenticated`。

1.6 版本后，如果所使用的鉴权模式不是 AlwaysAllow，则匿名访问默认是被启用的。对于匿名请求，ABAC 和 RBAC 要求必须给出显式的权限判定。

### 3.7 User Impersonation
一个用户可以通过 **`Impersonation Header`** 来以另一个用户身份执行操作。
1. 用户发起 API 请求，提供自身的认证，同时提供要伪装的头部字段信息。
2. APIServer 对用户进行身份认证。
3. APIServer 确认用户有着伪装的权限。
4. 请求的用户信息被替换为伪装的用户信息。
5. 针对伪装的用户进行验证与授权管理。

可以通过一下 HTTP Header 来进行伪装：
* **Impersonate-User**：要伪装成的用户名
* **Impersonate-Group**：要伪装成的用户组名。可以多次指定以设置多个用户组
* **Impersonate-Extra-<附加名称>**：一个动态的头部字段，用来设置与用户相关的附加字段

要伪装为某个用户或用户组时，需要有着  “impersonate” 权限，利用 RBAC 设置对应的 Role 如下：
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: impersonator
rules:
- apiGroups: [""]
  resources: ["users", "groups", "serviceaccounts"]
  verbs: ["impersonate"]
```

使用 kubectl 时，可以通过 *--as* 参数设置 Impersonate-User，通过 *--as-group* 参数设置 Impersonate-Group。
```shell
$ kubectl drain mynode --as=superman --as-group=system:masters
```


## 4 Authorization
经过了身份认证后，APIServer 就会进行授权检查的流程，即通过授权策略（Authorization Policy）决定是否有权限进行调用。

APIServer 目前有以下授权策略：
* AlwaysDeny: 拒绝所有请求，仅用于测试
* **AlwaysAllow**: 允许所有请求
* **ABAC**：基于属性的访问控制
* **RBAC**：基于角色的访问控制
* **Webhook**：通过调用外部的 REST 服务对用户授权
* **Node**：对 kubelet 进行的授权的特权模式

通过 APIServer 启动参数 *"--authorization-mode"* 配置多种授权策略。默认使用 Node 与 RBAC 策略。
```shell
--authorization-mode=Node,RBAC
```

当系统配置了多个授权模式时，Kubernetes 将按顺序使用每个模式。 **如果任何鉴权模块批准或拒绝请求，则立即返回该决定**，并且不会与其他授权模式协商。 如果所有模块对请求没有意见，则拒绝该请求。

当请求被拒绝时，会返回 HTTP Code 403。

### 4.1 ABAC 授权模式
**`ABAC（Attribute-Based Access Control）`** 是基于属性的访问控制。

#### 4.1.1 授权策略文件

集群管理员通过启动参数 *"--authorization-policy-file=SOME_FILENAME"* 指定授权策略文件的路径。

授权策略文件每一行都是一个 Map 类型 JSON 对象，称为 **“策略对象”**。
```json
{"apiVersion": "abac.authorization.kubernetes.io/v1beta1", "kind": "Policy", "spec": {"user": "alice", "namespace": "somenamespace", "resource": "*", "apiGroup": "*"}}
{"apiVersion": "abac.authorization.kubernetes.io/v1beta1", "kind": "Policy", "spec": {"group":"system:authenticated",  "nonResourcePath": "*", "readonly": true}}
```
* apiVersion：有效值为 "abac.authorization.kubernetes.io/v1beta1"
* kind: 有效值为 "Policy"
* spec.user: 用户名，来自于 *--token-auth-file* 文件中记录的 user
* spec.group: 设置为 "system:authenticated" 时，表示匹配所有已认证请求；设置为 "system:unauthenticated" 时，表示匹配所有未认证去哪个区
* spec.readonly: 表明仅用于 get list watch 操作
* 匹配资源
    * spec.apiGroup: 表示匹配哪些 API Group
    * spec.namespace: 表示允许访问哪个 namespace 下的资源
    * spec.resource: 表明要匹配的 API 资源对象
* 匹配非资源
    * spec.nonResourcePath: 如果匹配非资源，表明要匹配的请求路径

#### 4.1.2 授权算法
1. APIServer 收到请求后，识别出请求的策略对象属性。
2. 根据在策略文件中定义的策略，逐条进行匹配，以判断是否允许授权。
3. 如果至少一条成功，那么就通过授权。

#### 4.1.3 示例
Alice 可以对所有资源做任何事情：
```json
{"apiVersion": "abac.authorization.kubernetes.io/v1beta1", "kind": "Policy", "spec": {"user": "alice", "namespace": "*", "resource": "*", "apiGroup": "*"}}
```

Kubelet 可以读写事件：
```json
{"apiVersion": "abac.authorization.kubernetes.io/v1beta1", "kind": "Policy", "spec": {"user": "kubelet", "namespace": "*", "resource": "events"}}
```

任何人都可以对所有非资源路径进行只读请求：
```json
{"apiVersion": "abac.authorization.kubernetes.io/v1beta1", "kind": "Policy", "spec": {"group": "system:authenticated", "readonly": true, "nonResourcePath": "*"}}
{"apiVersion": "abac.authorization.kubernetes.io/v1beta1", "kind": "Policy", "spec": {"group": "system:unauthenticated", "readonly": true, "nonResourcePath": "*"}}
```

#### 4.1.4 对 Service Account 进行授权
**ServiceAccount 会自动生成一个 ABAC 用户名**，命名规则如下：
```
system:serviceaccount:<namespace>:<serviceaccountname>
```

因此，可以通过 ABAC 对某个 ServiceAccount 进行访问控制。例如希望 kube-system namespace 中的 "default" 有全部权限，修改策略文件:
```json
{"apiVersion":"abac.authorization.kubernetes.io/v1beta1","kind":"Policy","spec":{"user":"system:serviceaccount:kube-system:default","namespace":"*","resource":"*","apiGroup":"*"}}
```

当然，需要重启 APIServer 来重新加载策略文件。

### 4.2 RBAC 授权模式
见 [**K8s 学习 - RBAC 授权机制**](http://kanshiori.cn/posts/cloud_computing/k8s_learning/rbac/)

### 4.3 Webhook 授权模式
Webhook 模式使用参数 *--authorization-webhook-config-file=SOME_FILE* 来开启，文件为 kubeconfig 文件格式。
```yaml
# Kubernetes API 版本
apiVersion: v1
# API 对象种类
kind: Config
# clusters 代表远程服务。
clusters:
  - name: name-of-remote-authz-service
    cluster:
      # 对远程服务进行身份认证的 CA。
      certificate-authority: /path/to/ca.pem
      # 远程服务的查询 URL。必须使用 'https'。
      server: https://authz.example.com/authorize
users:
  - name: name-of-api-server
    user:
      client-certificate: /path/to/cert.pem # webhook plugin 使用 cert
      client-key: /path/to/key.pem          # cert 所对应的 key
current-context: webhook
contexts:
- context:
    cluster: name-of-remote-authz-service
    user: name-of-api-server
  name: webhook
```
* `user` 设置了 API Server 的配置
* `cluster` 设置了远端 Webhook 服务器的配置

#### 4.3.1 请求
授权时，APIServer 会生成一个 **`SubjectAccessReview`** 对象，通过 JSON 格式以 HTTP POST 发送给 Webhook 服务器。

SubjectAccessReview 对象中包含了用户访问资源请求动作的描述，以及访问的资源信息。

对于资源的访问请求如下：
```json
{
  "apiVersion": "authorization.k8s.io/v1beta1",
  "kind": "SubjectAccessReview",
  "spec": {
    "resourceAttributes": {
      "namespace": "kittensandponies",
      "verb": "get",
      "group": "unicorn.example.org",
      "resource": "pods"
    },
    "user": "jane",
    "group": [
      "group1",
      "group2"
    ]
  }
}
```
* `spec.resourceAttributes` 表明访问的资源的，以及访问的操作
* `spec.user` 请求的用户
* `spec.group` 请求的用户组

对于非资源的访问请求如下：
```json
{
  "apiVersion": "authorization.k8s.io/v1beta1",
  "kind": "SubjectAccessReview",
  "spec": {
    "nonResourceAttributes": {
      "path": "/debug",
      "verb": "get"
    },
    "user": "jane",
    "group": [
      "group1",
      "group2"
    ]
  }
}
```
* `spec.nonResourceAttributes` 对 API 的访问与操作

{{< admonition note Note>}}
因为 apiVersion 使用 "authorization.k8s.io/v1beta1"，因此 APIServer 需要开启该 API Group（*--runtime-config=authorization.k8s.io/v1beta1=true*）。
{{< /admonition >}}

#### 4.3.2 回复
Webhook 服务器返回的也是一个 SubjectAccessReview 对象，但是需要设置其中的 status 字段。
* 允许访问（allowed=true）

    ```json
    {
        "apiVersion": "authorization.k8s.io/v1beta1",
        "kind": "SubjectAccessReview",
        "status": {
            "allowed": true
        }
    }
    ```
* 不允许访问（allowed=false），但是如果有其他授权者，继续可以对请求进行授权

    ```json
    {
        "apiVersion": "authorization.k8s.io/v1beta1",
        "kind": "SubjectAccessReview",
        "status": {
            "allowed": false,
            "reason": "user does not have read access to the namespace"
        }
    }
    ```
* 不允许访问（allowed=false），同时立刻拒绝其他授权者（denied=true）

    ```json
    {
        "apiVersion": "authorization.k8s.io/v1beta1",
        "kind": "SubjectAccessReview",
        "status": {
            "allowed": false,
            "denied": true,
            "reason": "user does not have read access to the namespace"
        }
    }
    ```

### 4.4 Node 授权模式
Node 授权模式仅仅针对 Subject 为 Node，专门对 kubelet 发起的 API 请求进行授权的管理模式。


## 5 Admission Control
经过身份认证与授权后，请求最后还要通过 **`Admission Control（准入控制）`** 的控制链。

Admission Control 有一个准入控制器的插件列表，发送给 APIServer **任何请求都需要经过列表中每个准入控制器的检查**。任一一个控制器检查不通过，那么 APIServer 就会拒绝次调用。

**准入控制器还能够修改请求参数**，来实现一些自动化任务。

通过 APIServer 启动参数，可以配置哪些准入控制器启用，哪些准入控制器关闭：
* *"--enable-admission-plugins"* 参数配置开启的准入控制器：

    ```shell
    kube-apiserver --enable-admission-plugins=NamespaceLifecycle,LimitRanger ...
    ```
* *"--disable-admission-plugins"* 用以配置关闭的准入控制器，以此可以关闭默认开启的准入控制器：

    ```shell
    kube-apiserver --disable-admission-plugins=PodNodeSelector,AlwaysDeny ...
    ```

目前，Kubernetes 内置的准入控制器有 30 多个，见：[What does each admission controller do?](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/#what-does-each-admission-controller-do)

### 5.1 动态准入控制
除了内置的准入控制器，可以通过 Webhook 方式对接外部的 Admission Webhook 服务。

可以将其分为两类：
* Mutating Admission Webhook: 针对请求参数进行修改
* Validating Admission Webhook：针对请求进程检查

具体部署方式见：[动态准入控制](https://kubernetes.io/zh/docs/reference/access-authn-authz/extensible-admission-controllers/)

## 参考
* [官方文档：用户认证](https://kubernetes.io/zh/docs/reference/access-authn-authz/authentication/)
* [Blog: 基于 oAuth2 的 OIDC 原理 学习笔记](https://segmentfault.com/a/1190000018537009)
