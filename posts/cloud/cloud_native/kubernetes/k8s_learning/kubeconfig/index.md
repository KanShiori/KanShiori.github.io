# Kubernetes - 理解 kubeconfig


## 1 基本概念

kubeconfig 文件是 kubectl 用以查找访问集群的配置。

kubeconfig 文件的结构如下：

```yaml
apiVersion: v1
kind: Config
current-context: federal-context
clusters:
- cluster:
    certificate-authority: path/to/my/cafile
    server: https://horse.org:4443
  name: horse-cluster
- cluster:
    insecure-skip-tls-verify: true
    server: https://pig.org:443
  name: pig-cluster
contexts:
- context:
    cluster: horse-cluster
    namespace: chisel-ns
    user: green-user
  name: federal-context
- context:
    cluster: pig-cluster
    namespace: saw-ns
    user: black-user
  name: queen-anne-context
preferences:
  colors: true
users:
- name: blue-user
  user:
    token: blue-token
- name: green-user
  user:
    client-certificate: path/to/my/client/cert
    client-key: path/to/my/client/key
```

配置文件中主要包含三部分：

* `clusters` - 包含各个 Kubernetes 集群的信息以及访问的地址
* `contexts` - 特定集群的访问参数配置
* `users` - 访问集群的用户信息

可以看到，Cluster 表示集群，User 表示用户，而 Context 表明作为哪个 User 访问 Cluster。根据当前使用的 Context，kubectl 读取 User 凭证信息去访问 Cluster。

{{< image src="img1.png" height=250 >}}

### 1.1 Cluster

每个 Cluster 包含一个 Kubernetes 集群的访问信息。Cluster 定义如下：

```go
// Cluster contains information about how to communicate with a kubernetes cluster
type Cluster struct {
	// Server is the address of the kubernetes cluster (https://hostname:port).
	Server string `json:"server"`
	// TLSServerName is used to check server certificate. If TLSServerName is empty, the hostname used to contact the server is used.
	TLSServerName string `json:"tls-server-name,omitempty"`
	// InsecureSkipTLSVerify skips the validity check for the server's certificate. This will make your HTTPS connections insecure.
	InsecureSkipTLSVerify bool `json:"insecure-skip-tls-verify,omitempty"`
	// CertificateAuthority is the path to a cert file for the certificate authority.
	CertificateAuthority string `json:"certificate-authority,omitempty"`
	// CertificateAuthorityData contains PEM-encoded certificate authority certificates. Overrides CertificateAuthority
	CertificateAuthorityData []byte `json:"certificate-authority-data,omitempty"`
	// ProxyURL is the URL to the proxy to be used for all requests made by this
	// client. URLs with "http", "https", and "socks5" schemes are supported.
	ProxyURL string `json:"proxy-url,omitempty"`
	// Extensions holds additional information. This is useful for extenders so that reads and writes don't clobber unknown fields
	Extensions []NamedExtension `json:"extensions,omitempty"`
}
```

* `Server` - Kubernetes 集群地址；
* `InsecureSkipTLSVerify` - 是否跳过验证 APIServer 证书；
* `CertificateAuthority` - 验证 APIServer 证书使用的 CA 证书文件路径；
* `CertificateAuthorityData` - 验证 APIServer 证书使用的 CA 证书；

```yaml
clusters:
- cluster:
    certificate-authority-data: DATA+OMITTED
    server: https://127.0.0.1:37958
  name: kind-kind
```

{{< admonition note "DATA+OMITTED">}}
执行 `kubectl config view` 会看到证书信息为一个特殊的值 `certificate-authority-data: DATA+OMITTED`。

这是为了避免打印出过大的证书信息，使用 "DATA+OMITTED" 来替换打印。实际读取配置文件时可以看到使用的证书。
{{< /admonition >}}

### 1.2 User

User 本质上不是代表集群的用户，而是提供访问集群的凭证信息。

在 [**Kubernetes - 认证与鉴权机制**](../authentication-and-authorization/#2-身份认证) 中提到，APIServer 支持多种的身份认证方式。因此，User 字段中也可以包含任意一个认证信息。

User 的字段定义如下：

```go
// AuthInfo contains information that describes identity information.  This is use to tell the kubernetes cluster who you are.
type AuthInfo struct {
	// ClientCertificate is the path to a client cert file for TLS.
	ClientCertificate string `json:"client-certificate,omitempty"`
	// ClientCertificateData contains PEM-encoded data from a client cert file for TLS. Overrides ClientCertificate
	ClientCertificateData []byte `json:"client-certificate-data,omitempty"`
	// ClientKey is the path to a client key file for TLS.
	ClientKey string `json:"client-key,omitempty"`
	// ClientKeyData contains PEM-encoded data from a client key file for TLS. Overrides ClientKey
	ClientKeyData []byte `json:"client-key-data,omitempty" datapolicy:"security-key"`
	// Token is the bearer token for authentication to the kubernetes cluster.
	Token string `json:"token,omitempty" datapolicy:"token"`
	// TokenFile is a pointer to a file that contains a bearer token (as described above).  If both Token and TokenFile are present, Token takes precedence.
	TokenFile string `json:"tokenFile,omitempty"`
	// Impersonate is the username to imperonate.  The name matches the flag.
	Impersonate string `json:"as,omitempty"`
	// ImpersonateGroups is the groups to imperonate.
	ImpersonateGroups []string `json:"as-groups,omitempty"`
	// ImpersonateUserExtra contains additional information for impersonated user.
	ImpersonateUserExtra map[string][]string `json:"as-user-extra,omitempty"`
	// Username is the username for basic authentication to the kubernetes cluster.
	Username string `json:"username,omitempty"`
	// Password is the password for basic authentication to the kubernetes cluster.
	Password string `json:"password,omitempty" datapolicy:"password"`
	// AuthProvider specifies a custom authentication plugin for the kubernetes cluster.
	AuthProvider *AuthProviderConfig `json:"auth-provider,omitempty"`
	// Exec specifies a custom exec-based authentication plugin for the kubernetes cluster.
	Exec *ExecConfig `json:"exec,omitempty"`
	// Extensions holds additional information. This is useful for extenders so that reads and writes don't clobber unknown fields
	Extensions []NamedExtension `json:"extensions,omitempty"`
}
```

我们从不同的身份认证方式来看对应的各个字段：

* 基于证书的认证方式：[**X509 Client Cert**](../authentication-and-authorization/#21-x509-client-cert)

  * `ClientCertificate` / `ClientCertificateData` - 提供 client 证书或者证书文件；
  * `ClientKey` / `ClientKeyData` - 通过 client 私钥或者私钥文件；

  ```yaml
  users:
  - name: kind-kind
    user:
      client-certificate-data: REDACTED
      client-key-data: REDACTED
  ```

* 基于 Token 的认证方式：[**Static Token File**](../authentication-and-authorization/#22-static-token-file), [**Static Token File**](../authentication-and-authorization/#24-serviceraccount-token), [**Static Token File**](../authentication-and-authorization/#26-webhook-token) 等
  
  * `Token` / `TokenFile` - 提供 Token 字符串或者 Token 文件；

  ```yaml
  users:
  - name: kind-kind
    user:
      token: REDACTED
  ```

* 基于 username 与 password 的认证方式：[**Authentication Proxy**](../authentication-and-authorization/#27-authentication-proxy)
  
  一些 Authn Proxy 能够支持使用 username 与 password 方式来鉴权（例如 Nginx 作为 Proxy）。kubectl 也支持配置 username 与 password。

  * `Username` / `Password` - 配置用户名与密码；
  
  配置后，kubectl 会在请求中加入 HTTP HEAD `Authorization: Basic ${base64(user:password)}`。

  ```yaml
  users:
  - name: kind-kind
    user:
      username: foo
      password: bar
  ```

除了直接发送凭证信息外，kubectl 还支持通过 plugin 方式，在本地执行命令得到凭证信息后，再进行访问。

* 执行本地命令行，结果返回 Token
  
  通过 `Exec` 字段配置，可以让 kubectl 先执行配置的命令行，命令行返回格式**必须**为 [**ExecCredential**](https://kubernetes.io/zh/docs/reference/config-api/client-authentication.v1beta1/#client-authentication-k8s-io-v1beta1-ExecCredential) 格式。

  以 EKS 访问为例，其会执行 `aws` 命令来获取 Token：

  ```yaml
  - name: arn:aws:eks:us-west-2:123123123:cluster/test
    user:
      exec:
        apiVersion: client.authentication.k8s.io/v1alpha1
        args:
        - --region
        - us-west-2
        - eks
        - get-token
        - --cluster-name
        - staging-seed-us-west-2
        command: aws
        env: null
        provideClusterInfo: false
  ```

* 内置的 auth provider
  
  上面的命令行可以看做是非常通用的配置，一些方式已经集成到 kubectl 代码中，只需要提供相关的配置即可。

  通过 `AuthProvider` 提供一些特定 k/v 信息，kubectl 在本地获取对应的 Token：

  以访问 GKE 为例：

  ```yaml
  - name: gke_test
    user:
      auth-provider:
        config:
          cmd-args: config config-helper --format=json
          cmd-path: /usr/local/google-cloud-sdk/bin/gcloud
          expiry-key: '{.credential.token_expiry}'
          token-key: '{.credential.access_token}'
        name: gcp
  ```

除了凭证信息相关字段，还有一些字段是由于 [**User impersonation**](../authentication-and-authorization/#29-用户伪装-user-impersonation) 使用：

* `Impersonate` / `ImpersonateGroups` / `ImpersonateUserExtra`

### 1.3 Context

Cluster 提供了集群的访问方式，User 提供了访问集群的凭证信息，Context 就是将 Cluster 与 User 绑定。

kubeconfig 中可以保存多个 Context，而 kubectl 会使用 `current-context` 决定访问哪个集群。

```yaml
current-context: federal-context
contexts:
- context:
    cluster: horse-cluster
    namespace: chisel-ns
    user: green-user
  name: federal-context
- context:
    cluster: pig-cluster
    namespace: saw-ns
    user: black-user
  name: queen-anne-context
```

Context 可以认为是本地的绑定信息，使用 `kubectl config use-context ${context}` 可以方便的切换当前使用的 Context。

## 2 配置 kubeconfig

kubectl 使用的 kubeconfig 支持多种方式配置，按优先级排序如下：

* 命令行参数指定
  
  使用 `kubectl --kubeconfig` 指定使用的 kubeconfig 文件路径。

* KUBECONFIG 环境变量
  
  KUBECONFIG 环境变量可以包含 kubeconfig 文件列表，`:` 号分隔。kubectl 会将所有的文件配置合并为一个 kubeconfig 后使用。

* 默认路径
  
  如果前两种方式都没有配置，kubectl 会默认使用 `$HOME/.kube/config` 文件。

除了配置 kubeconfig 文件之外，kubectl 还支持配置凭证信息的命令行参数，例如：

* `--cluster` - 指定使用的 Cluster
* `--user` - 指定使用的 User
* `--context` - 指定使用的 Context
* `--server` - 指定访问的集群
* `--client-certificate` / `--client-key` / `--token` 等 - 指定访问集群使用的凭证信息

## 3 kubectl config

`kubectl config` 子命令用于配置 kubeconfig 文件，获取配置上下文。

一些常用的子命令包括：

* `current-context` - 打印出当前使用的 Context
  
* `use-context` - 切换使用的 Context
  
* `set` - 修改当前 Context 的某个字段

* `view` - 打印出合并后的 kubeconfig 配置，`--minify` 支持打印出当前使用的配置
  
* `delete-cluster` / `delete-context` / `delete-user` - 删除某个 Cluster / Context / User
  
* `get-clusters` / `get-contexts` / `get-users` - 查询某个 Cluster / Context / User
  
* `set-cluster` / `set-context` / `set-credentials` - 配置或者新增一个 Cluster / Context / User

## 参考

* 官方文档：[**Organizing Cluster Access Using kubeconfig Files**](https://kubernetes.io/docs/concepts/configuration/organize-cluster-access-kubeconfig/)
