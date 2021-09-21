# K8s 编程 - 7 - Admission Webhook


> **Kubernetes 编程系列** 主要记录一些开发 Controller 所相关的知识，大部分内容来自于[《Programming Kubernetes》](https://www.oreilly.com/library/view/programming-kubernetes/9781492047094/)（推荐直接阅读）。

> 下面代码都来自于 [*agnhost/webhook*](https://github.com/kubernetes/kubernetes/tree/master/test/images/agnhost/webhook)，这是官方的 Admission Webhook 的示例

在 [**Custom APIServer/Admission**](../6-custom-api-server/#6-admission) 一节中看到，APISever 提供了 Admission Plugin 机制来进行 Mutating 与 Validating 的插件式扩展。不过 Admission Plugin 是要重新编译 APISever 来实现的，因此 APISever 提供了更加灵活的 Admission Webhook。

Admission Webhook 的调用时机在 Admission Plugin 的尾部，在 Quota Plugin 之前。
{{< find_img "img1.png" >}}

Admission Webhook 的使用包含：
* 部署 Webhook Server，用于 APISever 转发请求；
* 部署 ValidatingWebhookConfiguration/MutatingWebhookConfiguration 资源，向 APISever 注册 Webhook Server；
* RBAC（如果 Webhook Server 需要）

## 1 WebhookConfiguration

通过部署 ValidatingWebhookConfiguration/MutatingWebhookConfiguration 来向 APIServer 注册 Validating/Mutating Webhook server。下面是一个 ValidatingWebhookConfiguration 示例，而 MutatingWebhookConfiguration 是类似的。
```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingWebhookConfiguration
metadata:
  name: "pod-policy.example.com"
webhooks:
- name: "pod-policy.example.com"
  # objectSelector:
  #   matchLabels:
  #     foo: bar
  # namespaceSelector:
  #   matchExpressions:
  #   - key: runlevel
  #     operator: NotIn
  #     values: ["0","1"]
  matchPolicy: Equivalent
  rules:
  - apiGroups:   [""]
    apiVersions: ["v1"]
    operations:  ["CREATE"]
    resources:   ["pods"]
    scope:       "Namespaced"
  clientConfig:
    # url: " https://my-webhook.example.com:9443/my-webhook-path"
    service:
      namespace: "example-namespace"
      name: "example-service"
    caBundle: "Ci0tLS0tQk...<`caBundle` is a PEM encoded CA bundle which will be used to validate the webhook's server certificate.>...tLS0K"
  admissionReviewVersions: ["v1", "v1beta1"]
  sideEffects: None
  timeoutSeconds: 5
  reinvocationPolicy: IfNeeded
  failurePolicy: Fail
```
* webhooks: 定义一个或者多个 webhook servers
  * name - 定义多个 webhook 情况下，需要定义一个唯一的 name
  * clientConfig - 定义 APIServer 转发的目标，可以为 URL 或者 Service
  * admissionReviewVersions - 表明 webhook server 支持的 AdmissionReview 的版本
  * sideEffects - 表明 Webhook 是否支持 Side effect
  * timeoutSeconds（1-30) - 配置 APIServer 等待 webhook server 回复的超时时间，超时后根据 failure policy 处理
  * rules - 定义转发的匹配规则
  * objectSelector - 根据 object 的 label 来筛选 APIServer 需要转发的请求
  * namespaceSelector - 根据 namespace 来筛选 APIServer 需要转发的请求
  * matchPolicy - 定义 rule 如何匹配请求
  * reinvocationPolicy - 定义是否需要重跑 webhook，在 object 修改后
  * failurePolicy - 定义请求失败后的行为

### 1.1 请求匹配规则

#### 1.1.1 rules

每个 webhook 必须指定一组 rules，用以让 APIServer 决定是否转发请求给 webhook server。当一个请求能够匹配 operation、group、version、resource、scope，那么请求就会转发。
```yaml
  rules:
  - apiGroups:   [""]
    apiVersions: ["v1"]
    operations:  ["CREATE"]
    resources:   ["pods"]
    scope:       "Namespaced"
```

* operations - 匹配请求的操作，支持 CREATE UPDATE DELETE CONNECT *
* apiGroups - 匹配请求的操作的 API Group，"" 表示 core API，"*" 匹配所有的 API Group
* apiVersions - 匹配 API Group 下的多个版本，"*" 匹配所有版本
* resources - 匹配 API Group 下多个 resource
  * "*" 表示匹配所有 resource，但是不包含 subresource
  * "*/" 表示匹配所有 resource 与 subresource
  * "pods/" 表示匹配 Pod 下的所有 subresource
  * "*/status" 表示匹配所有 resource 下的 status subresource
* scope - 定义需要匹配 resource 与 subresource 的范围，支持 Cluster Namespaced *

#### 1.1.2 objectSelector

通过 objectSelector 根据请求操作的 object 的 label 来筛选请求，成功匹配的请求才会被转发。

下面示例中，经过 rules 匹配后的请求，还会经过 objectSelector 筛选（需要包含 label foo:bar）。
```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: MutatingWebhookConfiguration
# ...
webhooks:
- name: my-webhook.example.com
  objectSelector:
    matchLabels:
      foo: bar
  rules:
  - operations: ["CREATE"]
    apiGroups: ["*"]
    apiVersions: ["*"]
    resources: ["*"]
    scope: "*"
  # ...
```

#### 1.1.3 namespaceSelector 

通过 namespaceSelector 根据请求操作的 object 的 namespace 来筛选请求。
```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: MutatingWebhookConfiguration
# ...
webhooks:
- name: my-webhook.example.com
  namespaceSelector:
    matchExpressions:
    - key: runlevel
      operator: NotIn
      values: ["0","1"]
  rules:
  - operations: ["CREATE"]
    apiGroups: ["*"]
    apiVersions: ["*"]
    resources: ["*"]
    scope: "Namespaced"
  # ...
```

#### 1.1.4 matchPolicy

一个资源可能属于多个 API Group，这在升级资源版本时很常见。例如，Deployment 支持 extensions/v1beta1，apps/v1beta1 等。

matchPolicy 用于定义 rules 如何匹配请求，其值可以为：
* Exact - 表示请求需要完全匹配
* Equivalent（默认值） - 表示请求可以匹配不同 APIGroup 的相同资源

例如下面示例，通过 matchPolicy 也可以匹配 extensions/v1beta1 的 Deployment 了。
```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingWebhookConfiguration
# ...
webhooks:
- name: my-webhook.example.com
  matchPolicy: Equivalent
  rules:
  - operations: ["CREATE","UPDATE","DELETE"]
    apiGroups: ["apps"]
    apiVersions: ["v1"]
    resources: ["deployments"]
    scope: "Namespaced"
  # ...
```

{{< admonition note "如何实现的？">}}
这是根据资源版本转换的实现，因为 extensions/v1beta1 Deployment 在 APIServer 中会被转化为 v1 请求，从而可以匹配到 Webhook 转发。
{{< /admonition >}}

### 1.2 与 Webhook 通信方式

#### 1.2.1 URL

URL 为一个 webhook server 地址，以标准的 URL 格式。但是，不允许使用用户或者基本身份认证（例如 URL 中的 user:password@），也不允许 URL 中使用 # 和 ? 传参。
```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: MutatingWebhookConfiguration
# ...
webhooks:
- name: my-webhook.example.com
  clientConfig:
    url: " https://my-webhook.example.com:9443/my-webhook-path"
  # ...
```

#### 1.2.2 Service

如果 Webhook Server 运行在集群中，可以指定 Service 来转发请求。其 port 默认为 443，path 默认为 "/"。
```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: MutatingWebhookConfiguration
# ...
webhooks:
- name: my-webhook.example.com
  clientConfig:
    caBundle: "Ci0tLS0tQk...<base64-encoded PEM bundle containing the CA that signed the webhook's serving certificate>...tLS0K"
    service:
      namespace: my-service-namespace
      name: my-service-name
      path: /my-path
      port: 1234
  # ...
```

### 1.3 Side effects

有些情况下，Webhook Server 不仅仅是处理 AdmissionReview 对象，而要进行一些“带外”操作（指操作实际的资源），这也被称为 Side effect。

sideEffects 字段用于指定 Webhook Server 能否处理 dryRun 的 请求（dryRun:true）。值可以为：
* Unknown - 未知，对于 dryRun 请求要转发给 Webhook 的，直接视为请求失败。
* None - 表明 Webhook 没有 Side effect。
* Some - 表明 Webhook 有一些 side effect，对于 dryRun 请求要转发给 Webhook 的，将直接视为请求失败。
* NoneOnDryRun - 表明 Webhook 有一些 side effect，而 Webhook 能够处理 dryRun 请求。

```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingWebhookConfiguration
# ...
webhooks:
- name: my-webhook.example.com
  sideEffects: NoneOnDryRun
  # ...
```

### 1.4 Reinvocation policy

对于 Mutating Admission Plugin，顺序调用不适用于所有的情况。可能 Webhook 修改了对象的结构，但是其他 Mutating Plugin 对于其新结构是拒绝的（但是其已经被调用过了，因此不再被调用）。

而 1.15 后，如果 Mutating Webhook 更改对象，内置的 Mutating Admission Plugin 会重跑。

reinvocationPolicy 字段可以控制 Webhook Server 在这种情况下，是否需要重跑。值可以为：
* Never - 一次中 Admission Control 不能多次调用 Webhook。
* IfNeeded - Webhook 调用后的对象又被其他 Admission Plugin 修改了，那么 Webhook 会再次重跑。

```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: MutatingWebhookConfiguration
# ...
webhooks:
- name: my-webhook.example.com
  reinvocationPolicy: IfNeeded
  # ...
```

不过需要注意：
* 不保证额外调用次数是一次
* 如果额外调用，导致对象再一次被修改，那么不保证还会再次调用 Webhook
* 使用此选项的 Webhook 可能会被重新排序，以减少额外调用数。
* 要保证修改后验证对象，应该使用 Validate Admission Webhook

这也表明了，Mutating Webhook 必须是幂等的。

### 1.5 Failure policy

failurePolicy 字段定义了，当 APIServer 请求 Webhook 失败后，如何处理（包括超时错误）。
* Ignore - Webhook 返回的错误会被忽略，API 请求继续
* Fail（默认） - Webhook 返回错误，API 请求被拒绝

```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: MutatingWebhookConfiguration
# ...
webhooks:
- name: my-webhook.example.com
  failurePolicy: Fail
  # ...
```


## 2 Request 与 Response

### 2.1 Request

APIServer 发送 POST HTTP 请求，其设置了 HTTP Header 中 `Content-Type: application/json`，body 内容为 AdmissionReview 对象的 JSON 格式，设置了其中的 "request" 字段。

下面示例展示了对于 apps/v1 Deployment scale 子资源的调用请求，对应的 AdmissionReview 对象内容。
```json
{
  "apiVersion": "admission.k8s.io/v1",
  "kind": "AdmissionReview",
  "request": {
    # 随机 uid，用于标识这次 admission 调用
    "uid": "705ab4f5-6393-11e8-b7cc-42010a800002",

    # 请求中对象的 GVK
    "kind": {"group":"autoscaling","version":"v1","kind":"Scale"},
    # 请求中对象的 GVR
    "resource": {"group":"apps","version":"v1","resource":"deployments"},
    # 请求对象的 subresource
    "subResource": "scale",

    # 请求源对象的 GVK
    # 因为 `matchPolicy: Equivalent` 可能会让请求转变，而这里保存转变前的 GVK
    "requestKind": {"group":"autoscaling","version":"v1","kind":"Scale"},
    # 请求源对象的 GVR
    # 因为 `matchPolicy: Equivalent` 可能会让请求转变，而这里保存转变前的 GVR
    # Fully-qualified group/version/kind of the resource being modified in the original request to the API server.
    # This only differs from `resource` if the webhook specified `matchPolicy: Equivalent` and the
    # original request to the API server was converted to a version the webhook registered for.
    "requestResource": {"group":"apps","version":"v1","resource":"deployments"},
    # 请求源对象的 subresource
    # 因为 `matchPolicy: Equivalent` 可能会让请求转变，而这里保存转变前的 subresource
    "requestSubResource": "scale",

    # 请求对象的 name
    "name": "my-deployment",
    # 请求对象的 namespace
    "namespace": "my-namespace",

    # 请求的操作，CREATE UPDATE DELETE CONNECT
    "operation": "UPDATE",

    "userInfo": {
      # APIServer 身份认证的 username
      "username": "admin",
      # APIServer 身份认证的 uid
      "uid": "014fbff9a07c",
      # APIServer 身份认证的 group
      "groups": ["system:authenticated","my-admin-group"],
      # Arbitrary extra info associated with the user making the request to the API server.
      # This is populated by the API server authentication layer and should be included
      # if any SubjectAccessReview checks are performed by the webhook.
      "extra": {
        "some-key":["some-value1", "some-value2"]
      }
    },

    # 请求操作的对象，可以通过 scheme 解析为具体的结构
    "object": {"apiVersion":"autoscaling/v1","kind":"Scale",...},
    # 当前集群中的对象
    # It is null for CREATE and CONNECT operations.
    "oldObject": {"apiVersion":"autoscaling/v1","kind":"Scale",...},
    # 操作的 Option，包含 CreateOptions、UpdateOptions、DeleteOptions
    "options": {"apiVersion":"meta.k8s.io/v1","kind":"UpdateOptions",...},

    # dryRun 表明请求是否是 dry run mode
    # See http://k8s.io/docs/reference/using-api/api-concepts/#make-a-dry-run-request for more details.
    "dryRun": false
  }
}
```

### 2.2 Response

Webhook server 应该返回 200 HTTP code，并设置了 HTTP Header 中 `Content-Type: application/json`，body 内容为 AdmissionReview 对象的 JSON 格式，设置了其中的 "response" 字段。

不同的操作可能有着不同的 response，不过至少其必须包含以下字段：
```json
{
  "apiVersion": "admission.k8s.io/v1",
  "kind": "AdmissionReview",
  "response": {
    "uid": "<value from request.uid>",
    "allowed": true
  }
}
```
* uid - 复制 request.uid
* allowed - 设置为 true 或者 false，表明允许或者禁止

拒绝请求时，可以提供需要 APIServer 返回给 client 的 HTTP code 与 message。
```json
{
  "apiVersion": "admission.k8s.io/v1",
  "kind": "AdmissionReview",
  "response": {
    "uid": "<value from request.uid>",
    "allowed": false,
    "status": {
      "code": 403,
      "message": "You cannot do this because it is Tuesday and your name starts with A"
    }
  }
}
```

对于 MutatingWebhook 可以对对象进行修改，这是通过 [**JSON Patch**](http://jsonpatch.com/) 实现的。
```json
{
  "apiVersion": "admission.k8s.io/v1",
  "kind": "AdmissionReview",
  "response": {
    "uid": "<value from request.uid>",
    "allowed": true,
    "patchType": "JSONPatch",
    "patch": "W3sib3AiOiAiYWRkIiwgInBhdGgiOiAiL3NwZWMvcmVwbGljYXMiLCAidmFsdWUiOiAzfV0="
  }
}
```
* patchType - 指定 patch 类型，目前仅仅支持 JSONPatch
* patch - patch 操作的 base64 编码，示例中为 "[{"op": "add", "path": "/spec/replicas", "value": 3}]"

1.19 后，也可以选择返回一个 WARN 信息。
```json
{
  "apiVersion": "admission.k8s.io/v1",
  "kind": "AdmissionReview",
  "response": {
    "uid": "<value from request.uid>",
    "allowed": true,
    "warnings": [
      "duplicate envvar entries specified with name MY_ENV",
      "memory request less than 4MB specified for container mycontainer, which will not start successfully"
    ]
  }
}
```
* warn 信息不能超过 120 字节


## 3 与 APIServer 的鉴权方式

Admission Webhook 与 APIServer 支持三种鉴权方式：basic auth、bearer token、cert。

需要三个阶段来进行配置：
1. 启动 APIServer 时，通过 --admission-control-config-file 参数指定 Admission Control 配置文件。
2. 在 Admission Control 配置文件中，指定 MutatingAdmissionWebhook  Controller 与 ValidatingAdmissionWebhook  Controller 读取的证书。
   
   在配置文件，通过 `kubeConfigFile` 字段指定对应的 kubeconfig 文件。
    ```yaml
    apiVersion: apiserver.config.k8s.io/v1
    kind: AdmissionConfiguration
    plugins:
    - name: ValidatingAdmissionWebhook
      configuration:
        apiVersion: apiserver.config.k8s.io/v1
        kind: WebhookAdmissionConfiguration
        kubeConfigFile: "<path-to-kubeconfig-file>"
    - name: MutatingAdmissionWebhook
      configuration:
        apiVersion: apiserver.config.k8s.io/v1
        kind: WebhookAdmissionConfiguration
        kubeConfigFile: "<path-to-kubeconfig-file>"
    ```

3. 在指定的 kubeConfig 文件中，提供需要的证书
   ```yaml
    apiVersion: v1
    kind: Config
    users:
    # name should be set to the DNS name of the service or the host (including port) of the URL the webhook is configured to speak to.
    # If a non-443 port is used for services, it must be included in the name when configuring 1.16+ API servers.
    #
    # For a webhook configured to speak to a service on the default port (443), specify the DNS name of the service:
    # - name: webhook1.ns1.svc
    #   user: ...
    #
    # For a webhook configured to speak to a service on non-default port (e.g. 8443), specify the DNS name and port of the service in 1.16+:
    # - name: webhook1.ns1.svc:8443
    #   user: ...
    # and optionally create a second stanza using only the DNS name of the service for compatibility with 1.15 API servers:
    # - name: webhook1.ns1.svc
    #   user: ...
    #
    # For webhooks configured to speak to a URL, match the host (and port) specified in the webhook's URL. Examples:
    # A webhook with `url: https://www.example.com`:
    # - name: www.example.com
    #   user: ...
    #
    # A webhook with `url: https://www.example.com:443`:
    # - name: www.example.com:443
    #   user: ...
    #
    # A webhook with `url: https://www.example.com:8443`:
    # - name: www.example.com:8443
    #   user: ...
    #
    - name: 'webhook1.ns1.svc'
      user:
        client-certificate-data: "<pem encoded certificate>"
        client-key-data: "<pem encoded key>"
    # The `name` supports using * to wildcard-match prefixing segments.
    - name: '*.webhook-company.org'
      user:
        password: "<password>"
        username: "<name>"
    # '*' is the default match.
    - name: '*'
      user:
        token: "<token>"
    ```

{{< admonition note Note>}}
配置中仅仅配置了 client 证书与私钥，而在 ValidatingWebhookConfiguration/MutatingWebhookConfiguration 定义中指定了 CA Bundle，即 CA 证书。
{{< /admonition >}}

## 参考
* [**《Programming Kubernetes》**](https://book.douban.com/subject/33393681/)
* [**Dynamic Admission Control**](https://kubernetes.io/docs/tasks/extend-kubernetes/configure-aggregation-layer/)
