# Certificate 总结


## 1 基本原理

签发证书本质上是：使用 CA 的私钥给申请的公钥进行签名

从签发证书的角度看，设计到两个对象：

* CA - 签发证书的机构
  * `root cert` - CA 证书（包含 CA 公钥）
  * `root key` - CA 私钥

* User - 请求签发证书的对象
  * `user pubkey` - User 公钥
  * `user key` - User 私钥
  * `user cert` - User 证书

简单来说，签发证书的过程就是：User 拿着 `user pubkey` 去请求 CA 签发，CA 使用 `root key` 进行数字签名，得到的就是 `user cert`。

{{< image src="img1.png" height=150 >}}

上面最关键的在于：<important>只要持有了 `root cert`，那么就可以解密 `user cert`，进而得到 `user pubkey`</important>。这个过程就是 **证书认证**。

{{< admonition note 第三方权威>}}
上述过程中 CA 是作为一个第三方权威的存在，持有了 `root cert` 表明了信任 CA 的权威性，进一步相信了由 CA 签发过的证书。
{{< /admonition >}}

我们以 Client 要认证 Server 的证书为例，看下各需要配置好的资源。

* Client - `root cert` 
* Server - `server key` + `server cert`

## 2 证书

### 2.1 证书格式

一般来说，大家提到证书指的都是 X.509 v3 证书，也是 HTTPs 使用的证书的格式。除了 X.509 之外，还有一些其他格式的证书，例如 PKCS。证书格式都使用名为 ASN.1 的抽象语法来描述。

以下都以 X.509 证书为例进行说明。

证书的结构如下：

{{< image src="img2.png" height=250 >}}

* **Version** - 版本号 
* **Serial Number** - 序列号
* **Signature Algorithm** - 签名算法
* **Issuer** - 颁发者信息
* **Validity** - 有效时间
  * **Not Before** - 生效时间
  * **Not After** - 过期时间
* **Subject** - 签发对象
* **Subject Public Key Info** - 签发的公钥信息
  * **Public Key Algorithm** - 公钥算法
  * **Subject Public Key** - 公钥
* Issuer Unique Identifier（可选）- 颁发者唯一标识
* Subject Unique Identifier（可选）- 签发对象唯一标识
* Extensions（可选）- 扩展信息
  * ...
* **Signature Algorithm** - 数字签名与算法

如 [**基本原理**](#1-基本原理) 中所述，Client 会使用 `root cert` 解密证书，然后可以得到 Subject 公钥，该公钥就是 **Subject Public Key** 字段的值。

**Signature Algorithm** 是对证书的数字签名，Client 在本地对证书计算出数字签名，通过对比该字段来验证证书的完整性。

**Validity** 指定了证书的有效期，当时间超过 **Not After** 证书就会过期并失效。

证书中还包含了许多信息，用于做一些额外的检查。例如， **X509v3 Subject Alternative Name**（简称 SAN）用于添加证书的额外名字，也就是认证 Server 会做额外的检查：

{{< image src="img3.png" height=200 >}}

* DNS - 检查 Server Domain 是否和 SAN 中指定的匹配
* IP - 检查 Server IP 是否和 SAN 中指定的匹配
* Email
* URI 

### 2.2 证书文件扩展名

X.509 有多种常用的扩展名：

* .pem - Privacy-enhanced Electronic Mail 格式，通常是 Base64 格式

* .cer .crt .der - DER 二进制格式
  
* .p12 - PKCS#12 格式，包含证书也包含私钥

### 2.3 证书链

如果 CA 使用 `root cert` 来颁发所有的证书，那么一旦 `root key` 泄漏，所有颁发的证书都会受到影响。

通常 CA 使用 `root cert` 颁发多个 `intermediate cert`，并使用 `intermediate cert` 来颁发其他的 `intermediate cert` 或者 `leaf cert`。而 `leaf cert` 就是各个 Server 使用的实体证书。

在证书验证时，会循着**证书链**向上的不断验证，直到验证到了受信任的 `root cert`。

## 3 颁发证书

### 3.1 使用 cfssl 颁发证书

使用 [**cfssl**](https://github.com/cloudflare/cfssl) 颁发证书大致流程如下：

1. 创建 CA 证书的签名请求 CSR。
   
   ```bash
   cfssl print-defaults csr > ca-csr.json
   ```

   CSR 中提供 CA 相关信息：

   ```json
    {
        "CN": "My Server",
        "CA": {
            "expiry": "87600h"
        },
        "key": {
            "algo": "rsa",
            "size": 2048
        },
        "names": [
            {
                "C": "US",
                "L": "CA",
                "O": "Shori",
                "ST": "Beijing",
                "OU": "Shiori"
            }
        ]
    }
   ```

2. 基于 CSR 文件，生成 CA 证书与私钥。
   
   ```bash
    cfssl gencert -initca ca-csr.json | cfssljson -bare ca -
   ```

3. 创建 Server 端的 CSR 文件与配置文件。
   
   ```bash
   cfssl print-defaults config > server-config.json
   cfssl print-defaults csr > server-csr.json
   ```

    Server 端的 CSR 中可以使用 `hosts` 字段配置支持的域名与 IP。

    ```json
    {
        "CN": "example.net",
        "hosts": [
            "127.0.0.1"
            "::1",
            "*.example.net",
            "www.example.net",
        ],
        // ...
    }
    ```

    配置文件中配置签名相关选项：

    ```json
      {
          "signing": {
              "default": {
                  "expiry": "8760h"
              },
              "profiles": {
                  "server": {
                      "expiry": "8760h",
                      "usages": [
                          "signing",
                          "key encipherment",
                          "server auth"
                      ]
                  },
                  "client": {
                      "expiry": "8760h",
                      "usages": [
                          "signing",
                          "key encipherment",
                          "client auth"
                      ]
                  }
              }
          }
      }
    ```

4. 基于 CSR 与 配置文件，生成 Server 端证书。
   
   ```bash
    cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=server server.json \
     | cfssljson -bare server
   ```

### 3.2 使用 cert-manager 颁发证书

[**cert-manager**](https://cert-manager.io/docs/) 是在 Kubernetes 集群实现了证书的签发。与 cfssl 的过程式对比，cert-manager 基于独立的 CRD 进行声明式的证书的签发。

cert-manager 提供的核心 CRD 包括：

* Issuer 和 ClusterIssuer
* Certificate

#### 3.2.1 Issuer 和 ClusterIssuer

Issuer 和 ClusterIssuer 对象代表 CA 机构，保管着 CA 证书与私钥，“负责” 签发证书（本质上签发证书是由 cert-manager 执行的）。

Issuer 和 ClusterIssuer 字段相同，区别在于 Issuer 是 namespaced 资源，仅仅负责同 namespace 的证书签发，而 ClusterIssuer 负责整个集群的。

例如，使用已有的证书与公钥来签发证书的 Issuer：

```yaml
apiVersion: cert-manager.io/v1
kind: Issuer
metadata:
  name: ca-issuer
  namespace: mesh-system
spec:
  ca:
    secretName: ca-key-pair
```

#### 3.2.2 Certificate

Certificate 表示一个证书：

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: acme-crt
spec:
  secretName: acme-crt-secret
  dnsNames:
  - example.com
  - foo.example.com
  issuerRef:
    name: letsencrypt-prod
    kind: Issuer
    group: cert-manager.io
```

* `secretName` - 保证证书的 Secert
* `issuserRef` - 请求的 Issuer 或者 ClusterIssuer

#### 3.2.3 自签名

如果是自签名的场景，需要先创建一个 SlefSigned Issuer 来创建 CA 证书与私钥，然后再创建 Issuer 负责颁发证书。

1. 创建一个 SlefSigned Issuer，配置 `slefSigned` 字段表明不使用已有的证书与私钥。

    ```yaml
    apiVersion: cert-manager.io/v1
    kind: Issuer
    metadata:
      name: selfsigned-issuer
      namespace: sandbox
    spec:
      selfSigned: {}
    ```

2. 定义一个 Certificate 资源来生成 CA 的证书与私钥。

    ```yaml
    apiVersion: cert-manager.io/v1
    kind: Certificate
    metadata:
      name: my-selfsigned-ca
      namespace: sandbox
    spec:
      isCA: true
      commonName: my-selfsigned-ca
      secretName: root-secret
      privateKey:
        algorithm: ECDSA
        size: 256
      issuerRef:
        name: selfsigned-issuer
        kind: ClusterIssuer
        group: cert-manager.io
    ```

3. 创建 Issuer 来使用刚刚生成的 CA 证书与私钥，后续就使用该 Issuer 颁发证书。

    ```yaml
    apiVersion: cert-manager.io/v1
    kind: Issuer
    metadata:
      name: my-ca-issuer
      namespace: sandbox
    spec:
      ca:
        secretName: root-secret
    ```

## 参考 

* Blog：[**关于证书和公钥基础设施的一切**](http://arthurchiao.art/blog/everything-about-pki-zh/)
* Wiki：[**X.509**](https://zh.m.wikipedia.org/zh-hans/X.509)
