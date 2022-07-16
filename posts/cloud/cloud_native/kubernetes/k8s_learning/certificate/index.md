# Kubernetes - 证书相关


## 1 Kubernetes 中的证书

Kubernetes 所有组件之间的通信都基于 TLS 进行加密，并且都基于 mTLS 进行双向认证。因此，每个组件作为调用者需要配置：client cert + client key + server ca cert，作为服务端需要配置：server cert + server key + client ca cert。

{{< admonition note Note>}}
Kubernetes 的双向 TLS 认证并不要求 Client 与 Server 证书都基于相同的 CA 签发，因此需要各自配置对方的 CA 证书。
{{< /admonition >}}

### 1.1 证书配置

我们从调用关系来看，各个组件需要的证书配置：

{{< image src="img1.png" height=350 >}}

1. **Kubectl -> APIServer**
   
   Kubectl 作为客户端，需要配置 Client Cert + Client Key + Server CA Cert 用以被认证，相关参数位于 kubeconfig 配置中。

   ```yaml
   users:
   - name: kind-kind
     user:
       client-certificate-data: REDACTED  # Client Cert
       client-key-data: REDACTED          # Client Key
   clusters:
   - cluster:
       certificate-authority-data: DATA+OMITTED # CA Cert
     name: kind-kind
   ```

   当然，证书认证仅仅是其中一种方式，其他方式见 [**Kubernetes - 认证与鉴权机制**](../authentication-and-authorization/#2-身份认证)。

   APIServer 作为服务端，需要配置 Server Cert + Server Key + Client CA Cert 用以被认证，相关参数通过启动参数配置。

   ```bash
   kube-apiserver \ 
    --tls-cert-file=/var/lib/kubernetes/kube-apiserver.pem \            # Server Cert
    --tls-private-key-file=/var/lib/kubernetes/kube-apiserver-key.pem \ # Server Key
    --client-ca-file=/var/lib/kubernetes/cluster-root-ca.pem \          # Client CA Cert
    # ...
   ```
   
2. **APIServer -> ETCD**
   
   APIServer 作为客户端，需要配置 Client Cert + Client Key + Server CA Cert 用以被认证，相关参数通过启动参数配置。

   ```bash
   kube-apiserver \ 
    --etcd-certfile=/var/lib/kubernetes/kube-apiserver-etcd-client.pem \    # Client Cert
    --etcd-keyfile=/var/lib/kubernetes/kube-apiserver-etcd-client-key.pem \ # Client Key
    --etcd-cafile=/var/lib/kubernetes/cluster-root-ca.pem \                 # CA Cert
    # ...
   ```

   ETCD 作为服务端，需要配置 Server Cert + Server Key + Client CA Cert，相关参数通过启动参数配置。

   ```bash
   etcd \ 
    --cert-file=/etc/etcd/kube-etcd.pem \\                # Server Cert   
    --key-file=/etc/etcd/kube-etcd-key.pem \\             # Server Key   
    --peer-trusted-ca-file=/etc/etcd/cluster-root-ca.pem  # Client CA Cert
    # ...
   ```

3. **Scheduler -> APIServer**

   Scheduler 作为客户端，APIServer 作为服务端，与 **Kubectl -> APIServer** 中一样。

4. **Controller -> APIServer** - APIServer 作为服务端，Controller 作为客户端
   
   Controller 作为客户端，APIServer 作为服务端，与 **Kubectl -> APIServer** 中一样。

5. **KubeProxy -> APIServer** -  APIServer 作为服务端，KubeProxy 作为客户端
   
   KubeProxy 作为客户端，基于 ServiceAccount 方式进行认证。

   ```yaml
   apiVersion: v1
   kind: Pod
   spec:
     serviceAccount: kube-proxy
   ```

6. **Kubelet <-> APIServer**
   
   Kubelet 作为客户端，Client Cert + Client Key + Server CA Cert 通过 `kubeconfig` 配置传入。

   ```bash
   kubelet --kubeconfig=/etc/kubernetes/kubelet.conf
   ```
   
   Kubelet 也会作为服务端，Server Cert + Server Key + Client CA Cert 通过配置文件传入。

   ```yaml
   # kubelet --config=/var/lib/kubelet/config.yaml
   apiVersion: kubelet.config.k8s.io/v1beta1
   kind: KubeletConfiguration
   tlsPrivateKeyFile: /var/lib/kubelet/pki/kubelet.crt # Server Cert
   tlsCertFile: /var/lib/kubelet/pki/kubelet.key       # Server Key
   authentication:
     x509:
       clientCAFile: /etc/kubernetes/pki/ca.crt # Client CA Cert
   # ...
   ```

   {{< admonition note Note>}}
   在没有提供 `tlsPrivateKeyFile` 与 `tlsCertFile` 配置情况下，Kubelet 会自签名证书与私钥，并保存到 `--cert-dir` 参数（默认为 /var/lib/kubelet/pki）指定的目录下。
   {{< /admonition >}}

   APIServer 作为服务端，与 **Kubectl -> APIServer** 中一样。

   APIServer 作为客户端，Client Cert + Client Key + Server CA Cert 通过命令行参数指定。

   ```bash
   kube-apiserver \ 
    --kubelet-client-certificate=/var/lib/kubernetes/kube-apiserver-kubelet-client.pem \ # Client Cert
    --kubelet-client-key=/var/lib/kubernetes/kube-apiserver-kubelet-client-key.pem \     # Client Key
    --kubelet-certificate-authority=/var/lib/kubernetes/cluster-root-ca.pem \            # Server CA Cert
    # ...
   ```

### 1.2 证书目录

ControlPlane 相关组件证书都是先在 Node 生成，然后通过 `hostPath` 挂载到 Pod 中，目录位于 `/etc/kubernetes/pki` 下：

```bash
.
|-- apiserver-etcd-client.crt    # APIServer -> ETCD 的 Client Cert
|-- apiserver-etcd-client.key    # APIServer -> ETCD 的 Client Key
|-- apiserver-kubelet-client.crt # APIServer -> Kubelet 的 Client Cert
|-- apiserver-kubelet-client.key # APIServer -> Kubelet 的 Client Key
|-- apiserver.crt                # APIServer 的 Server Cert
|-- apiserver.key                # APIServer 的 Server Key
|-- ca.crt                       # APIServer 的 Server CA Cert
|-- ca.key                       # APIServer 的 Server CA Key
|-- etcd
|   |-- ca.crt                   # ETCD 的 Server CA Cert
|   |-- ca.key                   # ETCD 的 Server CA Key
|   |-- server.crt               # ETCD 的 Server Cert
|   `-- server.key               # ETCD 的 Server Key
|-- sa.key                       # 用于签名 ServerAccount Token 的私钥
`-- sa.pub                       # 用于验证 ServerAccount Token 的公钥
```

Scheduler 或者 Kubelet 等核心组件访问 APIServer 使用的 `kubeconfig` 也是预先在 Node 生成，然后通过 `hostPath` 挂载到 Pod 中，目录位于 `/etc/kubernetes` 下：

```bash
.
|-- admin.conf          
|-- controller-manager.conf # Controller Manager 使用的 kubeconfig 文件
|-- kubelet.conf            # Kubelet 使用的 kubeconfig 文件
`-- scheduler.conf          # Scheduler 使用的 kubeconfig 文件
```

Kubelet 作为客户端和服务端相关的证书默认位于 `/var/lib/kubelet/` 目录下（CA 证书使用的 `/etc/kubernetes/pki/ca.crt`）：

```bash
|-- kubelet-client-current.pem # 访问 APIServer 的 Client Cert + Client Key
|-- kubelet.crt                # Kubelet 的 Server Cert
`-- kubelet.key                # Kubelet 的 Server Key
```

## 2 Kubernete 提供的证书签名

Kubernete 内部提供了简单的证书签名机制，基于 API Group `certificates.k8s.io` 提供。用户通过创建资源 `CertificateSigningRequest` 来提供 CSR，由 Controller Manager 来进行证书签发。

Controller Manager 基于以下参数提供了一个默认的 CA Cert 与 CA Key，来进行证书签发（默认使用 APIServer 的 CA）：

```bash
kube-controller-manager \ 
  --cluster-signing-cert-file=/etc/kubernetes/pki/ca.crt \ 
  --cluster-signing-key-file=/etc/kubernetes/pki/ca.key  \ 
```

### 2.3 证书签发

使用 `CertificateSigningRequest` 大致流程如下：

1. 本地创建 Server Key 与 Server CSR
   
   ```bash
   cat <<EOF | cfssl genkey - | cfssljson -bare server
   {
     "hosts": [
       "my-svc.my-namespace.svc.cluster.local",
       "my-pod.my-namespace.pod.cluster.local",
       "192.0.2.24",
       "10.0.34.2"
     ],
     "CN": "my-pod.my-namespace.pod.cluster.local",
     "key": {
       "algo": "ecdsa",
       "size": 256
     }
   }
   EOF
   ```

2. 创建 `CertificateSigningRequest` 对象：
   
   ```yaml
   apiVersion: certificates.k8s.io/v1
   kind: CertificateSigningRequest
   metadata:
     name: my-svc.my-namespace
   spec:
     request: $(cat server.csr | base64 | tr -d '\n')
     usages:
     - digital signature
     - key encipherment
     - server auth
   ```

   创建后可以看到相关的 csr 对象：

   ```bash
   $ kubectl get csr
   NAME                  AGE   SIGNERNAME            REQUESTOR          REQUESTEDDURATION   CONDITION
   my-svc.my-namespace   12s   example.com/serving   kubernetes-admin   <none>              Pending
   ```

## 参考

* 官方文档：[**CertificateSigningRequest**](https://kubernetes.io/zh-cn/docs/reference/access-authn-authz/certificate-signing-requests/)
