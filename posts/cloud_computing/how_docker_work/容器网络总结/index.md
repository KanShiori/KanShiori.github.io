# 容器网络总结


## 1 概览
docker 容器网络目前包含 5 中模式，包括：
 * **`bridge`**：默认的网络模式，使用 bridge 虚拟网卡 + iptables 实现一个内地内网，所有容器都处于该内网内，并且可以相互访问；
 * **`host`**：与宿主机处于同一个 net namespace，使用宿主机网络环境；
 * **`overlay`**：在多个 docker daemon 之间建立 overlay 网络，使得不同 docker daemon 的容器之间可以相互通信；
 * **`macvlan`**：使用 macvlan 虚拟网卡，将容器物理地址暴露在宿主机局域网中，你可以认为就是一台同局域网的物理机；
 * **`none`**：不进行任何网络配置，通常与自定义网络 driver 配合使用；

除了上述模式之外，其实 docker 还支持使用自定义的网络插件，这块不了解，具体见官方文档。<br>
下面所有示例都在虚拟机 ubuntu 20.04 与内核 5.4.0-52-generic 中完成，docker 版本如下：
```bash
$docker version
Client:
 Version:           19.03.8
 API version:       1.40
 Go version:        go1.13.8
 Git commit:        afacb8b7f0
 Built:             Wed Oct 14 19:43:43 2020
 OS/Arch:           linux/amd64
 Experimental:      false

Server:
 Engine:
  Version:          19.03.8
  API version:      1.40 (minimum version 1.12)
  Go version:       go1.13.8
  Git commit:       afacb8b7f0
  Built:            Wed Oct 14 16:41:21 2020
  OS/Arch:          linux/amd64
  Experimental:     false
 containerd:
  Version:          1.3.3-0ubuntu2
  GitCommit:
 runc:
  Version:          spec: 1.0.1-dev
  GitCommit:
 docker-init:
  Version:          0.18.0
  GitCommit:
```


## 2 背景知识

## 3 Bridge 网络

### 3.1 创建/删除 Bridge 网络
#### (1) 创建网络
先试着创建一个自定义的 bridge 网络，观察 docker 会对应创建哪些东西。
```bash
$ docker network create --driver=bridge \
	--subnet=192.168.100.0/24 \
	--ip-range=192.168.100.0/26 \
	--gateway=192.168.100.1 \
	--opt com.docker.network.bridge.name=mybr0 \
	mybridge0
2e61a7dc333c1bc61d9cb86503ce4cd5a7435977ea2f9b7cc97fc71ae0e2bb93
```
* `--driver=bridge` 指定创建的网络 driver；
* `--subnet=192.168.100.0/24` 指定对应 bridge 网络的网段；
* `--ip-range=192.168.100.0/26` 指定运行分配给容器的 ip 范围，当然，这个是要在指定的网段内的；
* `--gateway=192.168.100.1` 指定该内网的网关 IP；
* `--opt com.docker.network.bridge.name=mybr0` 指定创建虚拟 bridge 网卡的命名；
* `mybridge0` 为创建的 docker network 的命名；

通过 `ifconfig` 可以看到，bridge 网络创建会对应创建一个 **`bridge 网络设备`**，作为整个内网的 '交换机'。其 IP 就是指定的 gateway IP。
{{< admonition tip 推荐阅读 >}}
bridge 网卡推荐阅读：[Linux虚拟网络设备之bridge(桥)](https://segmentfault.com/a/1190000009491002)
{{< /admonition >}}
```bash
$ ifconfig
…
mybr0: flags=4099<UP,BROADCAST,MULTICAST>  mtu 1500
        inet 192.168.100.1  netmask 255.255.255.0  broadcast 192.168.100.255
        ether 02:42:46:8a:cf:34  txqueuelen 0  (Ethernet)
        RX packets 0  bytes 0 (0.0 B)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 0  bytes 0 (0.0 B)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
…

$ brctl show
bridge name     bridge id               STP enabled     interfaces
mybr0           8000.0242efdb0984       no
```
但是与虚拟机网络中的 bridge 网卡不同，该 bridge 不会连接任何的物理网卡，仅仅是作为内网的 '交换机' 使用。但是，毕竟内网是虚拟的，没有实际与物理网络连接，如何访问外网呢？<br>
答案是，**通过内核 iptables 进行 NAT，然后将包从实际的物理网卡上发送与接受**。因此还有一部分的改变在于 iptables，主要会建立的是 nat 与 filter 表的规则。<br>
先看 nat 表的相关规则（下面输出中省略了不相关规则）：
```bash
$  iptables -t nat -L  -nv
Chain PREROUTING (policy ACCEPT 2 packets, 88 bytes)
target     prot opt in     out     source               destination
DOCKER     all  --  *      *       0.0.0.0/0            0.0.0.0/0            ADDRTYPE match dst-type LOCAL

Chain INPUT (policy ACCEPT 2 packets, 88 bytes)
target     prot opt in     out     source               destination

Chain OUTPUT (policy ACCEPT 124 packets, 8797 bytes)
target     prot opt in     out     source               destination
DOCKER     all  --  *      *       0.0.0.0/0           !127.0.0.0/8          ADDRTYPE match dst-type LOCAL

Chain POSTROUTING (policy ACCEPT 124 packets, 8797 bytes)
target                prot opt in     out     source               destination
MASQUERADE  all     --   *      !mybr0  192.168.100.0/24     0.0.0.0/0

Chain DOCKER (2 references)
target       prot opt in     out     source               destination
RETURN     all  --  mybr0  *       0.0.0.0/0            0.0.0.0/0
```
* PREROUTING 与 OUTPUT 链中规则，使得所有入和出的包都会经过 **DOCKER** 链；
* POSTROUTING 链中，将 mybridge0 网络（192.168.100.0/24）的内网 ip 通过 **MASQUERADE** 行为进行伪装（可以简单认为内网 ip 会变为当前网卡的 ip）；<br>
当然，如果包发往的是 mybr0 网卡，说明是 mybridge0 网络内部通信，就不需要进行 MASQUERADE 伪装（!mybr0）；

当容器发包时，会通过 mybr0 网卡转发进入内核栈，因此在 filter 表中，相关的规则都是针对于 "in=mybr0"。看一下 filter 表的规则：
```bash
$ iptables -t filter -L  -nv
Chain INPUT (policy ACCEPT 61774 packets, 79M bytes)

Chain FORWARD (policy ACCEPT 0 packets, 0 bytes)
target     prot opt in     out     source               destination
DOCKER-USER  all  --  *      *       0.0.0.0/0            0.0.0.0/0
DOCKER-ISOLATION-STAGE-1  all  --  *      *       0.0.0.0/0            0.0.0.0/0
ACCEPT     all  --  *      mybr0   0.0.0.0/0            0.0.0.0/0            ctstate RELATED,ESTABLISHED
DOCKER     all  --  *      mybr0   0.0.0.0/0            0.0.0.0/0
ACCEPT     all  --  mybr0  !mybr0  0.0.0.0/0            0.0.0.0/0
ACCEPT     all  --  mybr0  mybr0   0.0.0.0/0            0.0.0.0/0

Chain OUTPUT (policy ACCEPT 42290 packets, 55M bytes)
target     prot opt in     out     source               destination

Chain DOCKER (2 references)
target     prot opt in     out     source               destination

Chain DOCKER-ISOLATION-STAGE-1 (1 references)
target     prot opt in     out     source               destination
DOCKER-ISOLATION-STAGE-2  all  --  mybr0  !mybr0  0.0.0.0/0            0.0.0.0/0
RETURN     all  --  *      *       0.0.0.0/0            0.0.0.0/0

Chain DOCKER-ISOLATION-STAGE-2 (2 references)
target     prot opt in     out     source               destination
DROP       all  --  *      mybr0   0.0.0.0/0            0.0.0.0/0
RETURN     all  --  *      *       0.0.0.0/0            0.0.0.0/0

Chain DOCKER-USER (1 references)
target     prot opt in     out     source               destination
RETURN     all  --  *      *       0.0.0.0/0            0.0.0.0/0
```
* FORWARD -> DOCKER-ISOLATION-STAGE-1 -> DOCKER-ISOLATION-STAGE-2 表明允许包从 mybr0 进入并转发（即容器可以向外正常发包）；
* FORWARD 中对 mybr0 进入的包设置了 **conntrack**，使得能够收到连接建立后的正常的回包；
#### 删除网络
通过 `docker network remove` 删除网络时，会发现对应的 bridge 网卡与 iptables 规则都被删除。
```bash
$ docker network remove 5a17670afb6f
5a17670afb6f
```

### 3.2 启动容器后的网络
下面看下容器启停后，带来的网络变化。先启动最简单的一个容器，指定使用网络为上面创建的 mybridge0。
```bash
$ docker run -dt --rm --network=mybridge0 --name br0_container ubuntu
676f7f9eab12a20fb3a975fa99cc2c92433a9581b5774ea58e63d447d86aa5ad
$ docker inspect 676f7f9eab12
…
        "Networks": {
                "mybridge0": {
                    "IPAMConfig": null,
                    "Links": null,
                    "Aliases": [
                        "882cac3e472f"
                    ],
                    "NetworkID": "af1dbf619ac62be1ad8a6b63696d3e6edff77cceab6cd0ee78de4b51e0d33683",
                    "EndpointID": "56cd85c5121d7d14146fcacc75599f6c56034e758e81f405a51437276ac6ac9f",
                    "Gateway": "192.168.100.1",
                    "IPAddress": "192.168.100.2",
                    "IPPrefixLen": 24,
                    "IPv6Gateway": "",
                    "GlobalIPv6Address": "",
                    "GlobalIPv6PrefixLen": 0,
                    "MacAddress": "02:42:c0:a8:64:02",
                    "DriverOpts": null
                }
            }
…
```
* `--network=mybridge0` 表明以 mybridge0 网络启动容器；
* 观察容器具体参数，可以看到，容器被随机分配 mybridge0 设置的 ip-range 一个 ip，并且 gateway 就是 mybridge0 网络的网关地址；

观察网络设备，可以看到一个 **`veth-pair 设备`** 出现在宿主机上，并且连接到了 mybr0 网卡：
```bash
$ ifconfig
…
vethef6b174: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet6 fe80::e8ad:86ff:fefe:14ca  prefixlen 64  scopeid 0x20<link>
        ether ea:ad:86:fe:14:ca  txqueuelen 0  (Ethernet)
        RX packets 0  bytes 0 (0.0 B)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 23  bytes 1882 (1.8 KB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
…

$ brctl show
bridge name     bridge id               STP enabled     interfaces
mybr0           8000.0242efdb0984       no              vethef6b174
```
veth-pair 都是成对出现的，可以简单被看做一个通道，一端发入的包会从另一端发出，并进入内核协议栈。不过，在 bridge 网络环境下，veth5b480f8 连接到 mybr0，所以所有从 veth5b480f8 发出的包都会被 mybr0 接手转发（相当于就是一根网线插入了交换机）。
{{< admonition tip 推荐阅读 >}}
veth-pair 设备了解推荐文章：[Linux虚拟网络设备之veth](https://segmentfault.com/a/1190000009251098)
{{< /admonition >}}
可以进入容器 namespace，看一下容器内的 veth 设备。
```bash
$ docker exec -it br0_container bash

# 以下在容器 namesapce 环境执行
root@676f7f9eab12:/# ifconfig
eth0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 192.168.100.2  netmask 255.255.255.0  broadcast 192.168.100.255
        ether 02:42:c0:a8:64:02  txqueuelen 0  (Ethernet)
        RX packets 1928  bytes 21609413 (21.6 MB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 1817  bytes 102779 (102.7 KB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
root@676f7f9eab12:/sys/class/net/eth0# ethtool -i eth0
driver: veth
version: 1.0
firmware-version:
expansion-rom-version:
bus-info:
supports-statistics: yes
supports-test: no
supports-eeprom-access: no
supports-register-dump: no
supports-priv-flags: no
root@676f7f9eab12:~# cat /sys/class/net/eth0/iflink
15
```
* 在容器内的仅仅有一个 eth0 网卡，ip 设置为了容器的 ip。但是其实这个网卡就是 veth 设备改了名字；
* 通过 `ethtool -i eth0` 命令看到，其对应 driver 是 veth，并且 */sys/class/net/eth0/iflink* 文件表明了对端的 veth 网卡编号为 15（即宿主机看到的 veth 网卡设备）；

现在，我们试着启动容器并添加一个 tcp 端口映射。
```bash
$ docker run -dt --rm \ 
    --network=mybridge0 --publish 12211:8080 \
    --name br0_container ubuntu
2502f7397a37e2ab482f8a9152d1ed968dd2e2825c71eb2a6737e4900f7236c1
```
而这个端口映射，就是将宿主机的 12211 端口映射给容器的 8080，所以所有发往宿主机的 12211 端口的包，都会被修改端口并转发到容器内部。这也是通过 iptables 实现的：
```bash
iptables -t nat -L  -nv
Chain PREROUTING (policy ACCEPT 0 packets, 0 bytes)
target     prot opt in     out     source               destination
DOCKER     all  --  *      *       0.0.0.0/0            0.0.0.0/0            ADDRTYPE match dst-type LOCAL

Chain INPUT (policy ACCEPT 0 packets, 0 bytes)
target     prot opt in     out     source               destination

Chain OUTPUT (policy ACCEPT 0 packets, 0 bytes)
target     prot opt in     out     source               destination
DOCKER     all  --  *      *       0.0.0.0/0           !127.0.0.0/8          ADDRTYPE match dst-type LOCAL

Chain POSTROUTING (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination
   10   617 MASQUERADE  all  --  *      !mybr0  192.168.100.0/24     0.0.0.0/0
    0     0 MASQUERADE  tcp  --  *      *       192.168.100.2        192.168.100.2        tcp dpt:8080

Chain DOCKER (2 references)
 pkts bytes target     prot opt in     out     source               destination
    0     0 RETURN     all  --  mybr0  *       0.0.0.0/0            0.0.0.0/0
    0     0 DNAT       tcp  --  !mybr0 *       0.0.0.0/0            0.0.0.0/0            tcp dpt:12211 to:192.168.100.2:8080
```
* PREROUTING  -> DOCKER 链中，所有不是从 mybr0 进入的包，并且发往 tcp 12211 端口的包，都会被 **DNAT** 为发往 192.168.100.2:8080。这样就实现了端口映射的功能。

### 3.3 bridge 网络总结
中心思想：bridge 网络使用 bridge 网卡创建了一个本地的内网，而 bridge 网卡 + iptables 规则成为了这个内网的 '**路由器**'。其中:
* bridge 网卡作为二层的交换机，bridge 网卡 ip 作为路由器的网关 ip。
* iptables 规则实现了 brdige 网卡与物理网络的连接
* 宿主机内核栈实现了这个 '路由器' 的路由功能。

下图展示了整个 bridge 网络的模型（图片来自网络）：
{{< find_img "img1.png" >}}
其中比较关键的点：
1. veth pair 设备将容器 net namespace 连接到 bridge 网卡（可以看做将 veth pair 作为网线插到了 bridge 这个 '路由器' 上）。
2. iptables 实现了 bridge 网卡与物理网络的 '连接'。
bridge 网卡收到的包，经过 iptables 的 MASQUERADE 将包进行地址转换，并经过内核协议栈的路由通过物理网卡发送到物理网络。而回包通过 conntrack 机制正常接收与逆地址转换。
3. 容器与宿主机的端口映射，也是通过 iptables 的 DNAT 实现的。


## Host 网络







## 参考
* [Docker 容器网络官方文档](https://docs.docker.com/network/)
* [Linux虚拟网络设备之bridge(桥)](https://segmentfault.com/a/1190000009491002)
