# 容器启停原理总结

> **总结系列的文章**是自己的学习或使用后，对相关知识的一个总结，用于后续可以快速复习与回顾。

本文主要描述容器启停背后的步骤，但是不涉及源码。

示例的基于 ubuntu 20.04.1 LTS 虚拟机运行，docker 版本如下：
```bash
$ docker version
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

## 1 启动

### 1.1 Create
通过 `docker create` 命令可以创建一个容器，但是容器并不会真正的运行，其对应进程不会存在。

create 容器经过的具体流程为：
 1. 容器参数检查与调整；
 2. 容器对应的读写层（RWLayer）的创建；
 3. 容器元信息的记录（主要是配置信息）；

#### (1) 容器参数检查与调整
检查容器运行参数是否合法，并调整一些参数的值，这一步不具体描述。

#### (2) 读写层的创建
容器的 "层" 可以分为：

* **`读写层`** ：保存容器对 rootfs 写、修改结果的层，并在容器退出后被 docker 删除；

  例如，容器中对系统盘 root 目录中某个文件的修改，这个修改后的文件会被复制一份在系统盘上。
* **`init 层`** ：用于处理一些与镜像不绑定，但是与运行容器相关的文件修改，主要是 /etc/resolve.conf /etc/hosts 等；
{{< admonition note "为什么要有 init 层？">}}
我的理解是：镜像层（只读层）提供的是一个静态的环境，即所有容器看到的环境都是一样。

而有些东西并不是想让所有容器看到的相同，例如 hostname、nameserver，这些 docker 都提供了参数配置，所 docker   单独抽出了一个 "机器维度" 的只读层，init 层

置于为什么不是在读写层修改，个人觉得是因为读写层是可以被 "导出" 的（docker save），而这些 /etresolve.conf   的内容又是不应该被导出的，所以放在了 init 层。
{{< /admonition >}}

* **`只读层`** ：镜像包含的所有层，仅仅只读。对只读层任何修改都会以 COW 形式放在读写层。

具体读写层的概念这里不展开，可以阅读官方文档：[storagedriver](https://docs.docker.com/storage/storagedriver/)。

这里读写层的创建仅仅指的是创建了 init 层与读写层的目录，并没有做 union mount，毕竟容器没有运行嘛，没必要。

找个例子看一下：
```bash
$ docker create --rm -t  ubuntu
361c520da78f848d639d65f042fcf5d448c13cbc4ce8c251dcba2250162b48fe
inspect 可以看到对应的读写层的目录：
$ docker inspect 361c520da78f
…
        "GraphDriver": {
            "Data": {
                "LowerDir": "/var/lib/docker/overlay2/d063d1d9c81d0c72d7384ea999dbd77b33d04b942ef94a5aabc6fb6cf984194c-init/diff:/var/lib/docker/overlay2/0336c489d40e65588748265a95f18328ddb1f5bcb9ebf10909fbf3f5f35b9496/diff:/var/lib/docker/overlay2/77d3ac91877751678bfec0576dab39ccd4b73666f8040aef387ef47ff30b4cf1/diff:/var/lib/docker/overlay2/ec8326178c990b52970a65371fd375737fdf256db597aa821a2b0f7d79bcc6f3/diff:/var/lib/docker/overlay2/385038374d3d369e98724926d0e1c240dcb74e31b1663ec1cb434c43ca2826f1/diff",
                "MergedDir": "/var/lib/docker/overlay2/d063d1d9c81d0c72d7384ea999dbd77b33d04b942ef94a5aabc6fb6cf984194c/merged",
                "UpperDir": "/var/lib/docker/overlay2/d063d1d9c81d0c72d7384ea999dbd77b33d04b942ef94a5aabc6fb6cf984194c/diff",
                "WorkDir": "/var/lib/docker/overlay2/d063d1d9c81d0c72d7384ea999dbd77b33d04b942ef94a5aabc6fb6cf984194c/work"
            },
            "Name": "overlay2"
        },
…

$ ls  /var/lib/docker/overlay2/d063d1d9c81d0c72d7384ea999dbd77b33d04b942ef94a5aabc6fb6cf984194c/ 
diff  link  lower  work
```
  * 所有的层（包含容器、镜像）都位于 */var/lib/docker/[driver]* 目录下，不过不同的 driver 有着不同的目录结构；
  * 每个层的目录结构也和对应的 driver 有关，overlay2 中就会包含 diff、work 等子目录，而真正容器运行后看到的就是 diff 目录被挂载后的内容；

在 */var/lib/docker/overlay2/* 目录下，我们还可以看到一个同读写层类似名字的 "**xxx-init**" 目录，这就是 init 层目录，对应的 diff 子目录也是用于挂载的目录：
```bash
$ ls /var/lib/docker/overlay2/d063d1d9c81d0c72d7384ea999dbd77b33d04b942ef94a5aabc6fb6cf984194c-init
committed  diff  link  lower  work

$ tree /var/lib/docker/overlay2/d063d1d9c81d0c72d7384ea999dbd77b33d04b942ef94a5aabc6fb6cf984194c-init/diff
/var/lib/docker/overlay2/d063d1d9c81d0c72d7384ea999dbd77b33d04b942ef94a5aabc6fb6cf984194c-init/diff
|-- dev
|   |-- console
|   |-- pts
|   `-- shm
`-- etc
    |-- hostname
    |-- hosts
    |-- mtab -> /proc/mounts
    `-- resolv.conf
```

但是这两个目录仅仅是被创建，如果执行 `mount` 命令可以看到，这些目录是没有被挂载的。

#### (3) 元信息的记录
执行 `docker create` 后，通过 `docker ps -a` 可以看到对应的容器，并且 inspect 可以看到对应的配置信息，因此，create 之后是有元信息的记录的。

而这个元信息就保存在 */var/lib/docker/containers* 目录下，每个子目录的名字就是对应的容器 ID：
```bash
$ cd /var/lib/docker/containers && ls
361c520da78f848d639d65f042fcf5d448c13cbc4ce8c251dcba2250162b48fe

$ ls 361c520da78f848d639d65f042fcf5d448c13cbc4ce8c251dcba2250162b48fe
checkpoints  config.v2.json  hostconfig.json
```
  * *config.v2.json* 保存的就是对应的容器配置信息；

可以看到， */var/lib/docker/containers* 目录就是所有容器信息的 "数据库"，这与层的概念是解耦的。

如果，你设置了 docker daemon 退出后不停止所有容器（默认情况 docker daemon 退出前会停止并删除所有容器，通过配置可以改变这个行为），那么 docker daemon 重新启动后，就会依靠这个目录进行 container 信息的恢复。

### 1.2 Start
通过 `docker start` 命令运行一个容器，其主要的步骤如下：
  1. 状态检查，只有 Create 与 Stop 状态容器才可以被 Start；
  2. 进行容器 rootfs 的构建，这里会进行读写层、init 层、镜像层的 union mount；
  3. 容器网络构建；
  4. 调用 containerd 进行容器的启动；
`docker run` 等同于 docker create + docker run，所以不需要特别说明。

#### (2) rootfs 的构建
**`rootfs`** 指定是容器最后看到的根文件系统，也就是 读写层 + init 层 + 镜像层经过 union mount 后的读写层。

我感觉，union mount 主要由两个特点：
  * **统一视图** ：将各个层 "压扁"，最后得到一个层。而上下层之间相同的文件、目录，就会被上层的覆盖。
  * **写时复制** ：对于整个视图的操作，只会影响最顶层（读写层），不会影响其他层，并且是写时复制的。

看一下具体示例：
```bash
$ docker start 361c520da78f
361c520da78f
```

通过 `mount` 命令，可以看到对应容器的 union mount 已经出现：
```bash
$ mount
…
overlay on /var/lib/docker/overlay2/d063d1d9c81d0c72d7384ea999dbd77b33d04b942ef94a5aabc6fb6cf984194c/merged type overlay (rw,relatime,lowerdir=/var/lib/docker/overlay2/l/O2TO66S3K4MTADSAAX6VXGTWSJ:/var/lib/docker/overlay2/l/UHTTQ5AJKPR23Y3V7J4ZLOIFDR:/var/lib/docker/overlay2/l/VWIFLRAQOPMH7LBAQQ5DDGIYVM:/var/lib/docker/overlay2/l/LQBRTVETGGWVU2OHWC42443K7X:/var/lib/docker/overlay2/l/5PDNI5HSOH6UMUDNWF4VMR46TS,upperdir=/var/lib/docker/overlay2/d063d1d9c81d0c72d7384ea999dbd77b33d04b942ef94a5aabc6fb6cf984194c/diff,workdir=/var/lib/docker/overlay2/d063d1d9c81d0c72d7384ea999dbd77b33d04b942ef94a5aabc6fb6cf984194c/work,xino=off)
nsfs on /run/docker/netns/4500ea4f0025 type nsfs (rw)

$ ls -lh /var/lib/docker/overlay2/l/O2TO66S3K4MTADSAAX6VXGTWSJ
lrwxrwxrwx 1 root root 77 Nov 14 15:51 /var/lib/docker/overlay2/l/O2TO66S3K4MTADSAAX6VXGTWSJ -> ../d063d1d9c81d0c72d7384ea999dbd77b33d04b942ef94a5aabc6fb6cf984194c-init/diff
```
  * 第一个挂载信息看出，将 init 层与镜像层的目录挂载到了读写的 *merged* 目录；
  * 后面执行的命令看待，使用的 */var/lib/docker/overlay2/l/O2TO66S3K4MTADSAAX6VXGTWSJ* 这样的目录是软链，指向前面说的对应的各个层的目录，因为整个层的名字太长，所以使用软链别名让挂载属性短一些；

#### (3) 容器网络构建
首先明确，**先有网络环境，再有的容器运行**。也就是说，是容器中的进程加入一个特定的 net namespace。所以，网络环境的构建的目标就是：得到一个配好网络的 net namespace。

所以网络构建可以分为三个大概的步骤：
  1. 创建一个新的 net namespace；
  2. 在这个 net namespace 里操作，构建好网络；
  3. 保留这个 net namesapce；

如何 "配好网络的 net namespace" 就和具体的网络模式有关了，具体见：[容器网络总结](https://kanshiori.github.io/posts/cloud_computing/how_docker_work/%E5%AE%B9%E5%99%A8%E7%BD%91%E7%BB%9C%E6%80%BB%E7%BB%93/)。

默认下，当一个 namespace 中没有任何进程时，该 namespace 就自动被内核销毁了（垃圾回收），而要将一个 namespace 持久化，就需要将其挂载到一个具体文件，这样该 net namespace 就会保留。

因此，docker 会通过这种方式先保留 net namespace，并让容器运行后的进程可以加入。当容器停止是，docker 将其手动删除。

通过 `mount` 命令，你可以看到对应容器的 net namespace 的挂载会随着容器运行出现。
```bash
$ mount
…
nsfs on /run/docker/netns/4500ea4f0025 type nsfs (rw)
…
```

当然，上面说的 "构建网络"，"挂载文件"，包括 "namespace 文件如何映射到对应的容器" 这些行为与信息，都是由 `libnetwork` 库中负责的。
{{< admonition tip Tip >}}
看 libnetwork 源码时发现一点，当某个代码需要进入 namespace，一定需要将当前 **goroutine 与 thread 绑定**（runtime.LockOSThread()）接口。

Why ？举个例子，一段代码由 G1 groutine 绑定到了 T1 线程执行，创建了 N1 namespace，并且期望在 N1 namespace 下执行。但是代码运行中，可能由于 Go 的调度，变成了 G1 groutine 绑定到 T2 线程执行。这时，就切换了 namespace 了，代码也就不是在期望的 namespace 下执行，这可能带来很大的问题。
{{< /admonition >}}

#### (4) 调用 containerd 启动容器
docker daemon 在设计到容器进程的运行时，都是交给 containerd，然后 containerd 调用 shim，shim 调用 runc 库执行。

真正容器内进程怎么运行还包括很多内容，特别是 runc 如何调整进程 namespace，如何启动进程这些内容，这里不再深入说。

如果有空，等后面出个文章详述（挖坑。

## 2 停止
### 2.1 Stop
stop 的步骤就是对 run 操作的回滚，包括：
  1. 停止容器的运行；
  2. 销毁容器网络；
  3. 解除容器读写层的挂载；

其中第 2、3 步就是反着来操作，没啥好说的。

#### (1) 停止容器运行
停止容器运行，其实就是停止容器内所有进程的运行。

有趣的是停止容器运行的过程里，并没有使用 runc 库，而是在 shim 这一层中，对 shim 进程发送信号。这一块也是后面细说。

大致的停止流程就是下面两步：
  1. 发送 配置/默认 (SIGTERM) 的停止信号；
  2. 上一步停止失败/超时，那么就发送 SIGKILL 信号；

### 2.2 Remove
老样子，操作的逆向，这也没啥好说的。
 1. 删除对应读写层目录与 init 层目录；
 2. 删除对应的容器元信息；

## 3 暂停与恢复

### 3.1 Pause 与 Resume
Pause 操作的原理很简单，通过 **cgroup.freezer** 冻结进程的运行，也就是不让进程被内核调度运行。

在容器运行状态，读取对应 cgroup 的 freezer.state 文件可以看到是 THAWED 状态。
```bash
$ docker ps
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES
f4b92a61851a        ubuntu              "/bin/bash"         2 seconds ago       Up 1 second                             host_container
$ cat /sys/fs/cgroup/freezer/docker/f4b92a61851a7f5f85569318c830722c275796b36d34a506eae05b547b31cb7c/freezer.state
THAWED
```

执行 `docker pause` 后，看到对应的 freezer.state 变为 FROZEN，表明被冻结了。
```bash
$ docker pause host_container
host_container
$ cat /sys/fs/cgroup/freezer/docker/f4b92a61851a7f5f85569318c830722c275796b36d34a506eae05b547b31cb7c/freezer.state
FROZEN
```

执行 `docker unpause` 恢复运行后，对应状态又变为了 THAWED：
```bash
$ docker unpause host_container
$ cat /sys/fs/cgroup/freezer/docker/f4b92a61851a7f5f85569318c830722c275796b36d34a506eae05b547b31cb7c/freezer.state
THAWED
```

而这个状态的变化，其实就是通过 `echo "<state>" > freezer.state` 实现的。

