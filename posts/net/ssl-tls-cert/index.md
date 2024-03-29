# SSL/TLS 总结


## 1 密码套件

SSL 在握手的第一步就是确认所使用的[密码套件]^(Cipher Suite)，使得首先 Client 与 Server 使用相同的算法进行通信。其中包含三个组件：

 * **`非对称加密算法`**：表明传输秘钥使用的非对称加密算法，例如 RSA、DHE_RSA 等；
 * **`对称加密算法`**：加密握手后，传输数据使用的加密算法，例如 AES_128_CBC 等；
 * **`数据摘要算法`**：数据校验使用的算法，例如 SHA256 等；

例如，Server 端传输 *TLS_RSA_WITH_AES_256_CBC_SHA256* 算法套件，表明其询问使用 RSA + AES_256_CBC + SHA256 三个算法组件。

### 1.1 对称加密算法

对称加密算法的作用很简单：<important>使用同一个秘钥，发送端进行数据加密，接收端进行数据解密</important>。

{{< image src="img0.png" height=200 >}}

相比于非对称加密，加解密速度快，因此用于**传输数据的加解密**。

但是，对称加密无法解决秘钥传输问题，一旦双方交换秘钥时泄露，那么数据的安全性就无法保障了。

### 1.2 非对称加密算法

> 这里不会说明非对称加密算法的算法原理，而是想弄清楚为什么使用非对称加密算法。

首先，我们明确非对称加密算法的作用。非对称加密算法包含两个密码：

 * **`公钥`**：可以公开的秘钥；
 * **`私钥`**：不可公开，自身持有的秘钥；
 
作用很简单，<important>对于非对称加密，用公钥加密的数据只能用私钥解开，用私钥加密的数据只能用公钥解开</important>。

{{< image src="img0-1.png" height=200 >}}

这样的作用在哪？因为对于对称加密，交换的秘钥一旦泄露，那么加密的数据就会被破解。而对于非对称加密，需要交换的往往只有公钥，而公钥被泄露后，那么使用公钥加密的数据还是无法被破解的，那么还是能保证了**单向的数据安全性**。

但是，非对称加密算法加解密速度比较慢，所以在 TLS 中仅仅用于“交换”对称密钥（实际上是交换生成密钥的参数），传输数据使用对称加密保护。

后面 [**TLS 握手过程**](#22-tls-握手过程)中可以看到，Client 与 Server 各自生成了一些 parameter，然后传输给对方，并根据组合后的 parameter 生成对称密钥。因为单向的数据安全性，Client 传输给 Server 的 parameter 是安全的，进而生成的对称密钥也是安全的。

### 1.3 摘要算法

摘要算法不是用于数据加密的，而是用于**数据的校验**。无论多大的数据，经过摘要算法最后得到固定长度的数据。而不同的数据得到的摘要结果是不同的。

因此，我们比较两个数据的摘要结果，就可以确定两个数据是否相同，也就是数据检验。

## 2 SSH

在 TLS 之前，先说一下 SSH，github 与 ssh 登录都使用了 SSH 协议。

[SSH]^(Secure Shell) 是建立在应用层上基础上的网络安全协议（注意，其没有使用 TLS 协议）。

SSH 使用非对称加密技术 RSA 加密了所有传输的数据。使用了两种验证方式：

 * 基于口令的安全验证：只要知道需要登录的机器的账号与密码，就可以登录。所有传输的数据都会被加密，但是无法防止“中间人”攻击。
  
 * 基于秘钥的安全验证：提前创建公钥与私钥，把公钥放在服务器上。在客户端要登录服务器时，会发送其本地保存的公钥。服务端在指定目录查找对应公钥是否匹配。

可以看到，**服务器使用公钥来进行客户端的验证**，因此 Github 与 SSH 登录时都需要将本地生成的公钥先提前配置。

## 2 SSL/TLS

### 2.1 SSL 与 TLS

[TLS]^(Transport Layer Security) 是 [SSL]^(Secure Sockets Layer) 的升级版，下面是其发展的历史：

| 协议    | 创建时间 | RFC      |
| ------- | -------- | -------- |
| SSL 1.0 | -        |
| SSL 2.0 | 1995     |
| SSL 3.0 | 1996     | RFC 6101 |
| TLS 1.0 | 1999     | RFC 2246 |
| TLS 1.1 | 2006     | RFC 4346 |
| TLS 1.2 | 2008     | RFC 5246 |
| TLS 1.3 | 2019     | RFC 8446 |

而 HTTPS 就是在 TCP 建立连接后，先进行 TLS 协议握手，然后使用加密传输。

### 2.2 TLS 握手过程
看一下具体的握手过程：

{{< image src="img1.png" height=350 >}}

1. **`ClientHello`**：客户端**发起建立 TLS 连接请求**（TCP 连接已经建立）

   Client 发送的消息中，会表明其自身支持的 TLS 以及密码套件，让 Server 可以选择合适的算法。

   即包括几个重要信息：
   * 支持的 TLS 最高版本；
   * 支持的密码套件，按照优先级会进行排序；
   * 支持的压缩算法；
   * （可选）session id，通过 session 重用可以跳过一些握手过程；
   * 随机串，后序生成密码使用，使得每次握手得到的密码都是不一样的； 
  
2. **`ServerHello`**：服务端**选出合适的密码套件**

    Server 根据 Client 的 hello 消息，在自己的证书中查找合适的证书（证书里包含了非对称加密算法和摘要算法），并挑选出合适的对称加密算法，也就得到了密码套件。同时，也会返回一个随机串。

	如果 TLS 版本，或者密码套件无法匹配，那么直接会连接失败。

3. **`Certificate`**：服务端**发送自己的证书，让 Client 检查**

4. **`ServerKeyExchange`**（可选）：发送非对接加密 premaster 生成所需的参数（如果算法需要的话）

	对于一些非对称加密算法（DHE_RSA），需要 Server 传递一些特殊的参数，用以生成【准密码（premaster）】，因此需要传递一些特殊参数。

5. **`CertificateRequest`**（可选）：如果服务端开启双向认证，那么就发送请求，**要求 Client 提供证书**

	在内部服务的 HTTPS 中，有时候 Server 需要去验证 Client 是否是内部的，也就是需要进行双向认证，这就需要 Server 去检查 Client 的证书。
	
	当然，大部分 Web 服务不需要，而 Client 认证是通过账号密码来决定的。

6. **`ServerHelloDone`**：服务端表明结束

7. **`Certificate`**（可选）：**Client 发送自己的证书**

	如果第 5 步发生，那么 Client 需要发送自己的证书。就算 Client 没有证书，也要发送空的消息，让 Server 决定是否继续。

8. **`ClientKeyExchange`**：**验证完毕证书后，生成 premaster，通过证书中公钥加密后发送**

   Client 验证完毕 Server 证书后，生成非对称加密算法的 premaster，然后通过证书的公钥发送。
	
   如果算法需要，会使用第 4 步接收的相关参数进行生成。

9. **`CertificateVerify`**（可选）：发送 Client 校验码 + 私钥加密校验码的数据，让 Server 尝试解密并验证，来验证 Client 证书确实是属性 Client 的

   如果第 7 步执行，那么为了防止 Client 发送一个不属于自身的证书，所以需要进行一次非对称加密通信，使得 Server 确保证书是属于 Client 的（client 拥有着私钥）。

10. **`Finished`** ：根据密码套件，双方算出 master 密码，并且使用对称加密算法进行一次通信，来验证密码无误。
   
    根据加密套件 + premaster，并且使用 Client 随机串 + Server 随机串，client Server 各独立生成 master 密码，并且是一样的。
	
    接着 Client 与 Server 都会使用缓存的校验码，使用 master 密码进行对称加密，然后发送给对方，让对方使用 master 密码进行解密。这样来验证 master 密码双方使用的是一样的。

TLS 完成后，client Server 就得到了相同的 master 密码，作为后续对称加密通信的密码。

### 2.3 为何安全

在上面的流程中可以看到，Client Server 使用了非对称加密算法来进行 premaster 的交换（第 8 步使用公钥加密后发送），而最终的 master 密码生成需要使用 premaster 作为一个参数。

Client 发送给 Server 的 premaster 是必须使用私钥才能解密得到的。而在整个 TLS 过程中，私钥是完全不会离开 Server 的，因此这个 premaster 是安全的。即使公钥泄漏，仅仅也只能得到 Server 传输给 Client 的 premaster，无法得到最终生成的对称密钥。

可以简单地理解，**双方生成密码的一个重要参数使用了非对称加密算法进行传输，保证了数据的安全，这样最后各自独立生成的 master 密码就不会泄露。**

## 3 证书

在 TLS 过程中，非对称加密保证了密钥交换的安全性，证书用于保证 Server 端的可信性。

### 3.1 原理

我们从申请证书的过程看：

1. 生成 **`申请（签名）文件`**，**申请文件中包含了要使用的公钥**，以及一些申请者的信息：地点、组织等。

2. 将申请文件通过安全的方式（私下）发送给 **`CA`**。

   CA 就是一个用于分发证书的权威机构。

3. CA 使用**自己的私钥，对申请文件进行签名**（加密），得到了证书。

可以看到，最后得到的证书中包含了：
 * 加密后的公钥
 * 申请者的信息
 * CA 信息
 * 证书年限
 * 等

**在浏览器与服务器中，会内置世界上所有 CA 的根证书，也就是包含了 CA 的公钥**。通过证书公钥对 Server 证书进行解密，并且判断其中的信息，就可以判断出该证书是否是 CA 签署的。

{{< admonition note Note>}}
xx.crt 是证书文件，xx.crs 是申请文件
{{< /admonition >}}

### 3.2 自签名
当我们内部服务，仅仅需要自签名时，其实就是自己生成了称为了一个 CA，得到了根证书与私钥。

而后使用私钥给 Server 签证书，将根证书内置给 Client 使用，就可以进行自签名的 TLS 通信。

下面进行一次自签名的过程：

1. 生成根证书与私钥

   ```bash
   openssl genrsa -out root.key 2048
   openssl req -new -x509 -days 3650 -key root.key -out ca.crt
   ```

1. 生成 Server 证书与私钥，并使用根证书私钥签名

   ```bash
   openssl req -newkey rsa:2048 -nodes -keyout server.key -out server.csr
   openssl x509 -req -extfile <(printf "subjectAltName=DNS:caliper.xycloud.com") -days 3650 -in server.csr -CA root.crt -CAkey root.key -CAcreateserial -out server.crt
   ```

1. （可选）如果需要双向认证，那么要生成 Client 的证书，并使用相同的根证书签名。

   ```bash
   openssl req -newkey rsa:2048 -nodes -keyout client.key -out client.csr
   openssl x509 -req  -days 3650 -in client.csr -CA root.crt -CAkey root.key -CAcreateserial -out client.crt
   ```

### 3.3 总结

换个角度看，为了实现自签名，双向 TLS，两边需要的文件

* Server 端需要
  
  * server.key - Server 端私钥
  * server.crt - 通过 server.csr，由 Server CA 签名后的证书
  * ca.crt - Client 端的 CA 证书

  ```go
   func main() {
      // 加载服务器证书和私钥
      serverCert, err := tls.LoadX509KeyPair("server.crt", "server.key")
      if err != nil {
         panic(err)
      }

      // 加载客户端 CA 证书
      caCert, err := ioutil.ReadFile("ca.crt")
      if err != nil {
         panic(err)
      }

      caCertPool := x509.NewCertPool()
      caCertPool.AppendCertsFromPEM(caCert)

      // 创建一个 HTTP 服务器，使用双向认证和客户端 CA 证书验证
      server := &http.Server{
         Addr: ":443",
         TLSConfig: &tls.Config{
            Certificates: []tls.Certificate{serverCert},
            ClientCAs:    caCertPool,
            ClientAuth:   tls.RequireAndVerifyClientCert,
         },
      }

      // 启动 HTTPS 服务器
      err = server.ListenAndServeTLS("", "")
      if err != nil {
         panic(err)
      }
   }
  ```

* Client 端需要

  * client.key - Client 端私钥
  * client.crt - 通过 client.csr，由 CA 签名后的证书
  * ca.crt - Server 端的 CA 证书

   ```go
   func main() {
      // 加载客户端证书和私钥
      clientCert, err := tls.LoadX509KeyPair("client.crt", "client.key")
      if err != nil {
         panic(err)
      }

      // 加载服务端 CA 证书
      caCert, err := ioutil.ReadFile("ca.crt")
      if err != nil {
         panic(err)
      }

      caCertPool := x509.NewCertPool()
      caCertPool.AppendCertsFromPEM(caCert)

      // 创建一个 HTTP 客户端，使用双向认证和服务端 CA 证书验证
      client := &http.Client{
         Transport: &http.Transport{
            TLSClientConfig: &tls.Config{
               Certificates: []tls.Certificate{clientCert},
               RootCAs:      caCertPool,
            },
         },
      }
   }
   ```

{{< admonition note Note>}}
如果是单向 TLS，那么就不需要 Client 私钥与证书，CA 证书还是需要内置。
{{< /admonition >}}

而验证就是 Client 使用公钥正常解密数据，然后检查下数据。

所以证书，**其实就是给 Server “盖了一个章”，表明其是权威机构认证的**。而 Client 内置的根证书使得 Client 可以认到这个权威机构。

## 参考

[**Blog：SSL/TLS 及证书概述**](https://segmentfault.com/a/1190000009002353)
