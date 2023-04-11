# DNS 域名解析系统


主机使用 IP 地址作为通信地址是很难记忆的，用户更愿意使用比较容易记忆的 Hostname。而 **`Domain Name System`**（DNS）就是用于将<important> Domain 转换为 IP 地址</important>。

## 1 Domain 体系

**`Domain`** 有着严格的层次划分，例如 `main.cctv.com.` 就由三个标号组成，其中 **.** 为根，com 为顶级域名，cctv 为二级域名，mail 为三级域名。

{{< image src="img1.png" height=250 >}}

{{< admonition note "被省略的 .">}}
因为域名的根永远是 . 号，因此用户在使用时常常会省略，域名解析程序会自动加上根 . 号。
{{< /admonition >}}

根据其层级结构，`main.cctv.com` 也被称为 `cctv.com` 的 **`SubDomain`**。

## 2 Domain Server

**`Domain Server`**（也称为 Name Server），是用于 <important>处理 DNS 解析请求的服务器</important>。

基于 Domain 的分层设置，用于处理 Domain 转换为 IP 的服务器也是按照不同层级分类。

* **根域名服务器** - 最顶级的域名服务器
  
  全球一共有 13 台根域名服务器，每台根域名服务器知晓所有的顶级域名服务器。

  根域名服务器通常不会直接对域名进行解析，而是根据其顶级域名，返回对应的顶级域名服务器的 IP 地址。

* **顶级域名服务器** - 负责所有顶级域名
  
  同样，顶级域名服务器收到 DNS 请求时，可能直接回复 IP 地址，也可能返回下一级权限服务器的 IP 地址。

* **权限域名服务器** - 负责管理某个区域的域名

* **本地域名服务器** - 域名解析的入口
  
  **本地域名服务器并不负责域名的某一层级**，而是作为域名解析的入口。主机发起的 DNS 解析请求都会发送到本地域名服务器，由<important>本地域名服务器来执行域名解析的过程</important>。

  每个因特网服务商 ISP，甚至一个大学里都可以有着本地域名服务器。

  本地域名服务器也被称为 **`DNS Resolver`**。

{{< image src="img2.png" height=250 >}}

## 3 域名解析过程

主机向本地域名服务器的查询一般是递归查询：如果本地服务器无法处理 DNS 请求，那么就由本地域名服务器向其他上级域名服务器发出 DNS 请求。

{{< image src="img3.png" height=300 >}}

**缓存机制**：为了提高 DNS 查询效率，域名服务器会缓存 Domain -> IP 的映射关系，并在设定的 TTL 时间后清理缓存记录。不仅仅是域名服务器，许多主机也会在本地维护自己的域名缓存。

## 4 DNS Record

域名服务器中保存着多个 **`DNS Record`**，接受到 DNS 解析请求后，会查询保存的 DNS Record 来进行回复。

一个 DNS Record 可以理解为 **Domain -> Content** 的映射。根据其 Content 的不同内容，DNS Record 分类不同的类型：

* **`A`** - Content 为 IP 地址
* **`AAAA`** - Content 为 IPv6 地址
* **`CNAME`** - Content 为另一个 [权威域名]^(canonical name)
* **`MX`** - Content 为 [邮件交换服务器]^(mail exchange)
* **`NS`** - Content 为另一个 [域名服务器]^(name server)
* **`TXT`** - Context 是对该 Domain 的描述信息
* **`ALIAS`** - （AWS Route 53 特有）Content 为另一个 Domain

{{< admonition note Note>}}
DNS Record 的类型就是 DNS 报文中的 Type。
{{< /admonition >}}

### 4.1 A 类型

A 类型 DNS Record 表示 **Domain 对应的 IP 地址**。当域名服务器接受到 DNS 请求时，查询到 Domain 对应的 DNS Record 是 A 类型，那么就可以直接返回 IP 地址。

```bash
$ dig www.google.com

; <<>> DiG 9.16.1-Ubuntu <<>> www.google.com
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 55926
;; flags: qr rd ad; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 0
;; WARNING: recursion requested but not available

;; QUESTION SECTION:
;www.google.com.                        IN      A

;; ANSWER SECTION:
www.google.com.         0       IN      A       172.217.26.228

;; Query time: 0 msec
;; SERVER: 172.23.64.1#53(172.23.64.1)
;; WHEN: Sat Dec 25 22:42:48 CST 2021
;; MSG SIZE  rcvd: 62
```

### 4.2 AAAA 类型

AAAA 类型 DNS Record 表示 **Domain 对应的 IPv6 地址**。

dig 命令默认仅仅查询 A 记录，需要通过命令行参数指定：

```bash
$ dig www.google.com AAAA

; <<>> DiG 9.16.1-Ubuntu <<>> www.google.com AAAA
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 57495
;; flags: qr rd ad; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 0
;; WARNING: recursion requested but not available

;; QUESTION SECTION:
;www.google.com.                        IN      AAAA

;; ANSWER SECTION:
www.google.com.         0       IN      AAAA    2404:6800:4004:801::2004

;; Query time: 40 msec
;; SERVER: 172.23.64.1#53(172.23.64.1)
;; WHEN: Sat Dec 25 22:44:47 CST 2021
;; MSG SIZE  rcvd: 74
```

### 4.3 CNAME 类型

CNAME 类型表示使用别名的 [权威域名]^(canonical name)。也就是说，**CNAME 是 Domain 的域名的别名**。

**当 Name Server 返回 CNAME 记录时，表明 DNS Resolver 应该通过返回的 CNAME 继续进行 DNS 解析**。

例如，当我们使用 CDN 服务时，用户提供源站后，通常 CDN 服务会提供一个 CNAME，例如 "xx.cdn.dnsv1.com"。而这时，CDN 服务器会在 Name Server 添加一条 DNS Record。

```
CNAME <source site> "xx.cdn.dnsv1.com"
```

这样，用户访问 source site 时后会返回 CNAME 地址，接着用户就会对 CNAME 地址的服务器发起 DNS 解析以及访问，从而用户就放回到了 CDN 的缓存服务器。

{{< image src="img4.png" height=250 >}}

### 4.4 MX 类型

MX 记录，表示[邮件交换]^(mail exchange)服务。每个邮件厂商都有一个自己的域名，查询该域名后，就会返回对应的邮件服务器的域名。
```bash
$ dig qq.com MX

; <<>> DiG 9.16.1-Ubuntu <<>> qq.com MX
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 1841
;; flags: qr rd ad; QUERY: 1, ANSWER: 3, AUTHORITY: 0, ADDITIONAL: 0
;; WARNING: recursion requested but not available

;; QUESTION SECTION:
;qq.com.                                IN      MX

;; ANSWER SECTION:
qq.com.                 0       IN      MX      20 mx2.qq.com.
qq.com.                 0       IN      MX      30 mx1.qq.com.
qq.com.                 0       IN      MX      10 mx3.qq.com.

;; Query time: 10 msec
;; SERVER: 172.23.64.1#53(172.23.64.1)
;; WHEN: Sat Dec 25 23:10:03 CST 2021
;; MSG SIZE  rcvd: 108
```

{{< admonition note Note>}}
MX 类型的出现是历史的原因，现在理论上可以使用 A 类型记录替代。
{{< /admonition >}}

### 4.5 NS 类型

NS 类型记录，表示该 Domain 需要继续访问其他的 Name Server 进行解析，即该 DNS Record 会**返回另一个 Name Server 的域名**。

{{< image src="img5.png" height=300 >}}

以访问 `qq.com`，当 DNS 请求发到 `com` Name Server 时，`com` 根据自身的 NS 记录告知应该去哪个 Name Server 继续进行 DNS 解析。

因此，NS 类型实际上实现了 Domain 的层级结构，并形成了迭代的 DNS 解析流程。

```bash
$ dig qq.com NS

; <<>> DiG 9.16.1-Ubuntu <<>> qq.com NS
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 25101
;; flags: qr rd ad; QUERY: 1, ANSWER: 4, AUTHORITY: 0, ADDITIONAL: 0
;; WARNING: recursion requested but not available

;; QUESTION SECTION:
;qq.com.                                IN      NS

;; ANSWER SECTION:
qq.com.                 0       IN      NS      ns4.qq.com.
qq.com.                 0       IN      NS      ns1.qq.com.
qq.com.                 0       IN      NS      ns2.qq.com.
qq.com.                 0       IN      NS      ns3.qq.com.

;; Query time: 60 msec
;; SERVER: 172.23.64.1#53(172.23.64.1)
;; WHEN: Sat Dec 25 23:15:17 CST 2021
;; MSG SIZE  rcvd: 126
```

### 4.6 TXT 类型

TXT 类型记录不是实际上影响 DNS 解析的记录，而是保存一些文本信息，其中一些信息是 DNS 解析的配置。

