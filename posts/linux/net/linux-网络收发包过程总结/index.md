# Linux 网络收发包过程总结


## 1 背景知识
### 1.1 中断
中断终止 CPU 执行流，立即执行必要的处理程序。分为两个类型：
* **同步中断**和**异常**：由 CPU 自身产生。例如程序崩溃、缺页异常；
* **异步中断**：由 IO 设备产生，任意时间可能发生；

两种中断发生后，CPU 会切换至内核态，并执行 [中断处理程序]^(interrupt handler)。

对于异步中断，因为会停止当前执行的进程，所以内核要确保中断处理程序尽快完成，尽快返还 CPU。同样，也会阻塞 IO 设备的下一次中断，Linux 将异步中断分为了 **硬件中断** 与 **中断下半部**。

#### 1.1.1 硬件中断
[硬件中断]^(hardware interrupt) 指硬件设备发起信号中断 CPU 执行，CPU 立即进行中断的处理。

实际上，为了不长时间阻塞与中断处理，硬中断往往启动到的是一个通知的作用。例如网卡收到数据包，并通过 DMA 写入 ringbuffer 后，会通过硬中断通知 CPU 从 ringbuffer 读取并处理数据。

通过 `/proc/interrupts` 可以看到硬中断的触发次数：
```bash
$ cat /proc/interrupts
# …
      CPU0
100:  15      IR-PCI-MSI  35651584-edge p2p1-TxRx-0
101:  0       IR-PCI-MSI  35651585-edge p2p1-TxRx-1
102:  0       IR-PCI-MSI  35651586-edge p2p1-TxRx-2
103:  0       IR-PCI-MSI  35651587-edge p2p1-TxRx-3
104:  0       IR-PCI-MSI  35651588-edge p2p1-TxRx-4
105:  0       IR-PCI-MSI  35651589-edge p2p1-TxRx-5
106:  0       IR-PCI-MSI  35651590-edge p2p1-TxRx-6
```
* 第一行：IRQ 编号；
* 第二行：每个 CPU 的中断次数；
* 第三行：中断的类型；
* 第四行：？？；
* 第五行：中断发起的设备名；

平时可能只需要关注第五行设备名称就行，因为可能要过滤出网卡对应队列的中断，然后将其绑定触发的 CPU，见网卡多队列。

#### 1.1.2 中断下半部
[中断下半部]^(bottom half) 包含三种处理方式：
* **`软中断 softirq`**：固定的 32 个接口，只留给对时间要求最严格的下半部使用。<br>
  查看 `/proc/softirqs` 文件 可以看到目前支持的软中断：
命名     | 含义
-|- 
HI      |
TIMER   | 定时中断
NET_TX  | 网络发送
NET_RX  | 网络接收
BLOCK   |
IRQ_POLL|
TASKLET | tasklet 软中断扩展
SCHED   | 内核调度
HRTIMER |
RCU     | RCU 锁
* **`tasklet`**：因为软中断只有固定的 32 个，为了支持扩展，tasklet 基于软中断时间，在不同处理器上运行，并且支持通过代码动态注册；
* **`工作队列 work queue`**：将一个中断的部分工作推后，可以实现一些 tasklet 不能实现的工作（比如可以睡眠）。<br>
  一些内核线程会不断处理工作队列的数据，其运行在进程上下文中，并且可以睡眠以及被重新调度。目前，**`kworker 内核线程`** 负责处理这个工作。

对于软中断与 tasklet，如果大量出现时，为了不一直进行 CPU 中断，内核会唤醒 **`ksoftirqd`** 内核线程进行异步的处理。**每个处理器有一个 ksoftirqd/n 线程，n 为 CPU 编号**。

## 2 网卡层
### 2.1 网卡多队列
网卡与系统传输数据包通过两个环形队列：`TX ring buffer`、`RX ring buffer`，也称为 `DMA 环形队列`。平时所说的设置网卡多队列指的就是设置这个环形队列的数量。

当网卡收到帧时，会**通过哈希来决定将帧放在哪个 ring buffer 上，然后通过硬中断通知其对应的 CPU 处理**。

默认下，处理环形队列数据由 CPU0 负责，可以通过配置**中断亲和性**，或者通过开启 **irqbalance service** 将中断均衡到各个 CPU 上。
#### 2.1.1 配置网卡队列
通过 `ethool -l/-L <nic>` 命令查看与配置网卡的队列数，通常配置的与机器 CPU 个数一样（如果网卡支持的话）：
```bash
$ ethtool -l eth0
Channel parameters for eth0:
Pre-set maximums:
RX:             0
TX:             0
Other:          1
Combined:       8
Current hardware settings:
RX:             0
TX:             0
Other:          1
Combined:       7

$ ethtool -L eth0 combined 8
```

设置后，你在 `/sys/class/net/<nic>/queues/` 可以看到对应的收发队列目录：
```bash
$ ls /sys/class/net/eth0/queues/
rx-0  rx-1  rx-2  rx-3  rx-4  rx-5  rx-6  rx-7  tx-0  tx-1  tx-2  tx-3  tx-4  tx-5  tx-6  tx-7
```

在 `/proc/interrupts` 中也可以看到各个队列对各个 CPU 的中断次数：
```bash
$ cat /proc/interrupts | grep eth0
# … 这里只打印了一个 CPU 中断
      CPU0
62:  0     IR-PCI-MSI   524288-edge   eth0
63:  0     IR-PCI-MSI   524289-edge   eth0-TxRx-0
64:  0     IR-PCI-MSI   524290-edge   eth0-TxRx-1
65:  1766  IR-PCI-MSI   524291-edge   eth0-TxRx-2
66:  0     IR-PCI-MSI   524292-edge   eth0-TxRx-3
67:  0     IR-PCI-MSI   524293-edge   eth0-TxRx-4
68:  0     IR-PCI-MSI   524294-edge   eth0-TxRx-5
69:  0     IR-PCI-MSI   524295-edge   eth0-TxRx-6
70:  0     IR-PCI-MSI   524296-edge   eth0-TxRx-7
```

通过 `ethtool -g/-G <nic>` 可以查看与配置 ring buffer 的长度：
```bash
$ ethtool -g eth0
Ring parameters for eth0:
Pre-set maximums:
RX:             4096
RX Mini:        0
RX Jumbo:       0
TX:             4096
Current hardware settings:
RX:             256
RX Mini:        0
RX Jumbo:       0
TX:             256
```

#### 2.1.2 配置中断亲和性
如果你发现 CPU0 的中断很高，那么就很有可能所有网卡队列的中断都打到了 CPU0 上。可以通过 `/proc/irq/<id>/smp_affinity_list` 查看指定编号的中断的对应允许的 CPU。
```bash
$ for i in {62..70}; do echo -n "Interrupt $i is allowed on CPUs "; cat /proc/irq/$i/smp_affinity_list; done
Interrupt 62 is allowed on CPUs 4
Interrupt 63 is allowed on CPUs 8
Interrupt 64 is allowed on CPUs 13
Interrupt 65 is allowed on CPUs 21
Interrupt 66 is allowed on CPUs 6
Interrupt 67 is allowed on CPUs 31
Interrupt 68 is allowed on CPUs 5
Interrupt 69 is allowed on CPUs 2
Interrupt 70 is allowed on CPUs 31
```
* 62-70 的网卡队列中断都打到了不同的 CPU 上；

通过写入 `echo "<bitmark>" > /proc/irq/<id>/smp_affinity` 可以中断绑定的 CPU，bitmark 的每个位对应一个 CPU，例如 "0x1111" 表示 CPU 0-3 都可以处理这个中断。

当然，你也可以通过 irqbalance 来均衡各个 CPU 的中断，动态的改变中断与绑定的 CPU。不过，irqbalance 不仅仅针对网卡队列中断，还会调整其他的。

### 2.2 接收数据
先来看第一个阶段，网卡接收到数据是如何处理的。
1. packet 进入物理网卡，物理网卡会根据目的 mac **判断是否丢弃**（除非混杂模式）；
1. 网卡通过 DMA 方式将 packet **写入到 ringbuffer**。<br>
   ringbuffer 由网卡驱动程序分配并初始化。
1. 网卡通过**硬中断通知 CPU**，有数据来了。
1. CPU 根据**中断执行中断处理函数**，该函数会调用网卡驱动函数。
1. 驱动程序**先禁用网卡中断**（NAPI），表示网卡下次直接写到 ringbuffer 即可，不需要中断通知了。<br>
	这样避免 CPU 不断被中断。
1. 驱动程序启动软中断，让 CPU 执行软中断处理函数**不断从 ringbuffer 读取并处理 packet**。

网卡多队列就是在这里生效，网卡会将 packet 放置到不同的 ringbuffer，不同的 ringbuffer 会中断不同的 CPU（如果设置了中断亲和性），使得各个 CPU 的队列硬中断是均衡的。

### 2.3 发送数据
网络设备通过驱动函数发送数据后，就归网卡驱动管了，不同的驱动有着不同的处理方式。

大致的流程如下：
1. 将 **sk_buff 放入 TX ringbuff**。
1. **通知网卡**发送数据。
1. 网卡发送完数据后，通过**中断通知 CPU**。
1. 收到中断后，进行 sk_buff 的**清理工作**。

当然，网卡驱动还有一些与 net_device 打交道的地方，比如网卡的队列满了，需要告诉上层不要再发了，等队列有空闲的时候，再通知上层接着发数据。

## 3 网络访问层
### 3.1 net_device
每个网络设备都表示为 **`net_device`** 一个实例。不同类型的网络设备都会有 net_device 表示，而其相关操作函数的实现不同。

net_device 是针对于 namespace 的，通过 sysfs 你可以看到当前命名空间下所有的 net_device。
```bash
$ ls /sys/class/net
docker0  enp0s3  enp0s8  lo  lxcbr0  vethea62395
```

net_device 包含了设备相关的所有信息，定义很长， 下面经过简化：
```c
struct net_device {
	char			name[IFNAMSIZ];
	
	// IO 相关字段
	unsigned long		mem_end;
	unsigned long		mem_start;
	unsigned long		base_addr;
	int			        irq;

	unsigned long	state;
	struct list_head	dev_list;

	int			ifindex;
	
	const struct net_device_ops *netdev_ops;
	const struct ethtool_ops *ethtool_ops;

	unsigned int		flags;
	unsigned int		mtu;
	unsigned short		type;
	unsigned short		hard_header_len;

	unsigned char		perm_addr[MAX_ADDR_LEN];
	unsigned char		addr_len;

#if IS_ENABLED(CONFIG_VLAN_8021Q)
	struct vlan_info __rcu	*vlan_info;
#endif
	// … 协议相关的特殊指针

	/* Interface address info used in eth_type_trans() */
	unsigned char		*dev_addr;

	struct netdev_rx_queue	*_rx;
	unsigned int		num_rx_queues;
	unsigned int		real_num_rx_queues;

	struct bpf_prog __rcu	*xdp_prog;
	unsigned long		gro_flush_timeout;
	int			napi_defer_hard_irqs;
	rx_handler_func_t __rcu	*rx_handler;
	void __rcu		*rx_handler_data;
	
	struct netdev_queue __rcu *ingress_queue;

	truct netdev_queue	*_tx；
	unsigned int		num_tx_queues;
	unsigned int		real_num_tx_queues;
	struct Qdisc		*qdisc;
	unsigned int		tx_queue_len;
};
```
* name\[IFNAMSIZ] ：设备命名；
* irq ：irq 编号；
* ifindex ：设备编号；
* mtu ：设备的最大传输单元；
* netdev_ops ：设备的操作接口，不同类型的接口有着不同的操作实现；
* dev_addr ：网卡硬件地址；
* _rx ：数据包接收队列（ringbuffer）；
* _tx ：数据包发送队列；
* qdisc : 设备进入的 qdisc；
* ingress_queue ：ingress 队列；
* 其他包含一些 XDP 相关，统计相关等字段；

大多数的统计信息都可以在对应设备的 sysfs 目录中找到。
{{< admonition note 网卡命名>}}
linux 内核启动过程中，会默认给网卡以 ethx 方式命名，后面 systemd 回去 rename 网卡名称。
{{< /admonition >}}

### 3.2 GRO
[GRO]^(Generic Receive Offloading) 用于将 jumbo frame（超过 1500B）的多个分片合并，然后将给上层处理，以减少上层处理数据包的数量。

通过 `ethtool -k/K <nic>` 查和设置 GRO。

### 3.3 RPS XPS RFS
#### 3.3.1 RPS
网卡多队列需要硬件的支持，而 [RPS]^(Receive Packet Steering) 这些则是软件实现多队列，将包让指定的 CPU 去处理。通常情况，当网卡队列数小于 CPU 个数时，可以让 RPS 进一步利用多余的 CPU，让其去处理中断。
{{< admonition note Note>}}
RPS 的设置是针对单个 ringbuffer 的，与网卡多队列处理不是同一个阶段，也不是冲突的。
{{< /admonition >}}

开启 RPS 后，数据会由经由被中断 CPU 转发，由其他 CPU 处理。流程如下：
{{< find_img "img1.png" >}}
1. 当网卡收到数据存入 ring buffer 后，还是通知指定 CPUx 从 ringbuffer 取出数据；
1. 不过接下来，CPUx 会为每个 packet 哈希放入其他 CPU 的 input_pkt_queue 中。
1. CPUx 通过 Inter-processor Interrupt (IPI)  中断告知其他 CPU，处理自己的 input_pkt_queue ；
1. 其他 CPU 从各个的 input_pkt_queue 中取出数据包，并处理之后的流程；

可以看到，**RPS 不是用于减少 CPU 软中断，而是用于将数据包处理时间均摊到各个 CPU 上**。
```bash
# 配置该 ringbuffer 使用 CPU0 CPU1 的队列
$ echo "0x11" > /sys/class/net/eth0/queues/rx-0/rps_cpus

# 配置每个 CPU 的 input_pkt_queue 的大小
$ sysctl -w net.core.netdev_max_backlog=1000
```

#### 3.3.2 RFS
[RFS]^(Receive Flow Steering) 一般和 RPS 配合工作。

RPS 将受到的 packet 经过哈希发配到不同的 CPU input_pkt_queue。而 RFS 会根据 packet 的数据流，发送到对应被处理的 CPU input_pkt_queue 上，即**同一个数据流的 packet 会被路由到处理当前数据流的 CPU 上**，从而提高 CPU cache 的命中率。

RFS 默认是关闭的，需要通过配置生效，一般推荐的配置如下：
```bash
# 配置系统期望的活跃连接数
$ sysctl -w net.core.rps_sock_flow_entries=32768

# 配置每个 rps 队列负责的 flow 最大数量，为 rps_sock_flow_entries / N
$ echo 2048 > /sys/class/net/eth0/queues/rx-0/rps_flow_cnt
```

#### 3.3.3 XPS
[XPS]^(Transmit Packet Steering) 则是发送时的多队列处理。

### 3.4 接收数据
在网卡层中，最后由异步的软中断处理函数来异步的从里 ringbuffer 的数据。而这由内核线程 ksoftirqd 来调用对应的网络软中断函数处理。

**所以，在数据包到达 socket buffer 前，数据处理都是由 ksoftirqd 线程执行的。**

1. ksoftirqd 调用**驱动程序的 poll 函数来一个个处理 packet**。<br>
   如果没有 packet 的话，就会重新启动网卡硬中断，等待下一次重新的流程。
1. poll 函数将读取的每个 packet，**转换为 sk_buff 数据格式，并分析其传输层协议**。
1. （可选）如果开启了 GRO，那么进行 **GRO 的处理**。
1. 如果开启了 RPS，那么进行 **RPS 的处理**，否则放入当前 CPU 的 input_pkt_queue。<br>
 RPS 处理的流程如下：
	1. 当前 CPU 将 sk_buff 放到其他 CPU 的 input_pkt_queue 中。如果 input_pkt_queue 满的话，packet 会被丢弃。
    2. CPU 通过 IPI 硬中断通知其他 CPU 处理自己的 input_pkt_queue，也就是走 11 步流程。

到这里，而无论是否开启 RPS，接下来就是 **CPU 从各自 input_pkt_queue 取出 sk_buff 并处理**。

1. CPU 查看 socket 是否是 AF_PACKET 类型的。如果是的话复制一份数据处理（例如 tcpdump 抓这里的包）。
1. 调用对应协议栈的函数，将数据包交给协议栈处理。

### 3.5 发送数据
接受到网络层的数据后，来到网络访问层会经过一个非常重要的系统：Traffic Controller。这是接收数据时不会经过。
1. 根据 sk_buff 中的设备信息，获取对应 net_device.qdisc。<br>
   如果 qdisc 存在的话，走流量控制系统，可能会丢弃包：
   TODO
1. 拷贝一份 sk_buff 给 "packet taps"。
1. 调用具体驱动的发送数据的函数发送。

{{< admonition note Note>}}
注意，发送数据时并没有 CPU 的 "outputqueue"，因为流量控制系统已经进行了流量控制。
{{< /admonition >}}

{{< admonition tip 入口流量控制>}}
如果想对入口流量进行流量控制，可以使用 tc ingressqdisc + 虚拟设备 IFB 进行
{{< /admonition >}}

## 4 网络层
网络访问层在接受到数据后，调用各个网络层协议的处理函数，进行 sk_buff 的处理。当然，我们下面说的是 IP 协议。
### 4.1 sk_buff
网卡接受到的数据包，整个在内核中传递使用的都是 sk_buff 数据结构。网络的各个层都是使用的同一个 sk_buff 对象，而无需进行数据的复制，使得性能更高。

sk_buff 包含各个指针执行对应数据的内存区域，并且表示其协议的 head data 等区域。
```C
// <sk_buff.h>
struct sk_buff {
    // ...

    union {
        __be16  inner_protocol;
        __u8    inner_ipproto;
    };

    __u16   inner_transport_header;
    __u16   inner_network_header;
    __u16   inner_mac_header;

    __be16  protocol;
    __u16   transport_header;
    __u16   network_header;
    __u16   mac_header;

    /* These elements must be at the end, see alloc_skb() for details.  */
    sk_buff_data_t  tail;
    sk_buff_data_t  end;
    unsigned char   *head,
        *data;
};
```
* head，end 为整个数据包的头尾内存地址；
* data，tail 为当前层对应的数据的头尾内存地址；
* transport_header，network_header，mac_header 传输层、网络层、链路层 header 的内存地址；

看图可能更好理解，sk_buff 通过指针将数据包各个区域表示出来了，而在各个协议层之间移动则是移动 data 与 tail 指针。
{{< find_img "img4.png" >}}

sk_buff_head 表示 sk_buff 组成的链表，这个结构就是 RPS 各个 CPU 的队列，以及 socket buffer 的实现。
```C
// <sk_buff.h>
struct sk_buff_head {
    /* These two members must be first. */
    struct sk_buff  *next;
    struct sk_buff  *prev;

    __u32       qlen;  // 链表长度
    spinlock_t  lock;
};
```

### 4.2 接收数据
看一下网络层处理数据报的步骤：
1. 如果其数据包的 MAC 地址不是当前网卡，那么丢弃（可能由于网卡混杂模式进来的，还是无法经过协议栈处理）。
1. 经过 netfilter 的 PREROUTING 阶段回调。
1. 进行路由判断：如果目的 IP 是本机 IP，那么接受该包。如果不是本机 IP，判断是否要路由。
1. 经过 netfilter 的 INPUT 阶段回调。
1. 进入传输层。

可以看到，如果仅仅是简单的接收数据包很简单，只要经过 netfilter 的回调即可。

看一下路由对数据包的处理：
1. 检查是否开启 ip forward 功能，没有开启的话数据包会执行丢弃。<br>
   通过 `sysctl -w net.ipv4.ip_forward=1` 开启 ip forward。
1. 经过 netfilter 的 FORWARD 阶段回调。
1. 走正常的发包流程发送数据包（会经过 netfilter POSTROUTING 阶段回调）

还是看图比较直观：
{{< find_img "img5.png" >}}


### 4.3 发送数据
1. 在 sk_buff 指向的数据区设置好 IP 报文头。
1. 调用 netfilter 的 OUTPUT 阶段回调。
1. 调用相关协议的发送函数（IP 协议或其他），将出口网卡设备信息写入 sk_buff。
1. 调用 netfilter 的 POSTOUTPUT 阶段回调。
1. POSTOUTPUT 可能设置了 SNAT，从而导致路由信息变化，如果发生变化重新重新回到 3。
1. 根据目的 IP，从路由表中获取下一跳的地址，然后在 ARP 缓存表中找到下一跳的 neigh 信息。
1. 如果没有 neigh信息，那么会进行一个 ARP 请求，尝试得到下一跳的 mac 地址。

到这里，得到了下一跳设备的 mac 地址，将其填入 sk_buff 并调用下一层的接口。

## 5 传输层
### 5.1 sock
sock 是 socket 在内核的表示结构，每个 sock 对应于一个用户态使用的 socket。TCP UDP 都是基于该结构来实现的。

其中，sock 最主要的属性就是常说的 socket buffer 了。
```c
// <sock.h>
struct sock {
    // …
	#define sk_state    __sk_common.skc_state
	// …
    struct sk_buff_head sk_error_queue;
    struct sk_buff_head sk_receive_queue;
    struct sk_buff_head sk_write_queue;

	int sk_rcvbuf;
	int sk_sndbuf;
	
	u32 sk_ack_backlog;
	u32 sk_max_ack_backlog;
	
	// 各个回调函数
    void    (*sk_state_change)(struct sock *sk);
    void    (*sk_data_ready)(struct sock *sk);
    void    (*sk_write_space)(struct sock *sk);
    void    (*sk_error_report)(struct sock *sk);
    int     (*sk_backlog_rcv)(struct sock *sk,
                struct sk_buff *skb);
    // …
}
```
* sk_state ：TCP 的状态；
* sk_receive_queue sk_write_queue ：接受/发送队列（buffer）；
* rcvbuf，sndbuf ：接受/发送队列的大小，单位 B；
* sk_ack_backlog ：经过三次握手后，等待 accept() 的全连接队列；

#### 5.1.1 配置 socket buffer 大小
通过 sysctl 可以配置 TCP、UDP 的接受与发送缓冲区大小：
```bash
sysctl -w net.ipv4.tcp_rmem="4096 503827 6291456"
sysctl -w net.ipv4.tcp_wmem="4096 503827 6291456"
sysctl -w net.ipv4.udp_mem="377868 503827  755736"
```
每个设置包含三个值，min、default、max。内核会根据当前的可用内存动态调节队列的大小；

### 5.2 UDP 层
#### 5.2.1 接收数据
先从简单的 UDP 处理开始看：
1. 对数据包进行一致性检查。
1. 根据目的 IP 与目的 port，在 udptable（包含机器所有的 udp sock）查找对应的 sock。没有找到则会丢弃数据包，否则继续。
1. 检查 receive buffer 是否满，如果满了则会直接丢弃数据包。
1. 检查数据包是否满足 BPF socket filter，如果不满足则直接丢弃数据包。
1. 将数据包放入 sock 接受队列。
1. 调用回调函数，以通知数据包已经准备好。这会将阻塞等待数据包到来的用户态程序唤醒。

UDP 的处理很简单，找到对应的 sock 结构，然后将其放入到队列中，唤醒用户态程序。

#### 5.2.2 发送数据
发送操作与接收不是对称的，因为在发送时就要确定出口的设备，来确认包是否会被直接丢弃。
1. 根据机器的路由表和目的 IP，决定数据包应该从哪个设备发送出去。如果根据路由表无法到达目的地址，直接丢弃包。
1. 如果 socket 没有绑定源 IP，那么就使用设备的源 IP。
1. 根据获取到路由信息，将 msg 构建为 sk_buff 结构体。
1. 向 sk_buff 的数据区填充 UDP 包头，然后调用 IP 层相关函数。


## 参考
总结
内核实现层次分明的比较清楚，因此可以一层层观察其对应负责的行为，所以整体要有一个框架的概念。
* 网卡层
    * [**网卡多队列的概念与位置**](#21-网卡多队列)；
    * [**CPU 如何接受网卡数据**](#22-接收数据)；
    * [**CPU 如何发送网卡数据**](#23-发送数据)；
* 网络访问层
    * [**网络访问层接受数据**](#34-接收数据)；
    * [**网络访问层发送数据**](#35-发送数据)；
    * [**ksoftiqrd 线程的任务**](#34-接收数据)；
    * [**RPS、XPS、RFS 的概念**](#33-rps-xps-rfs)；
    * [**TC 流量控制处理的位置**](#35-发送数据)；
    * XDP 处理的位置；
* 网络层
    * [**sk_buff 结构的概念**](#41-sk_buff)；
    * [**网络层接受数据**](#42-接收数据)；
    * [**网络层发送数据**](#43-发送数据)；
    * netfilter 各个阶段回调的位置；
    * [**如何进行路由判断**](#43-发送数据)；
* 传输层
    * [**sock 结构的概念**](#51-sock)；
    * [**UDP 的 recv socket buffer**](#521-接收数据)；
    * TCP 的半连接队列，全连接队列，r/w socket buffer；

