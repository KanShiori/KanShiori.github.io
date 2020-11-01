# 制作虚拟机镜像


中心思想：通过 libvirt 运行一个虚拟机（domain），并保存其对应的 domain 的镜像文件与配置文件，然后就可以在其他机器通过 `virsh define + start` 或者 `virt-install` 启动。<br>
说明：下面环境都是在 centos 上制作基于 KVM 的虚拟机镜像。

## 1 从 ISO 镜像安装
最基本的安装方式，通过安装并运行一个新的虚拟机，然后得到对应的配置文件与镜像文件。

### 1.1 手动安装
最基本的安装方式，通过 ISO 文件进行安装。
1. 下载 ISO 镜像文件，镜像文件在各个镜像站中就可以找到。（因为不使用图形界面，所以下载的是非图形化的 ISO）
2. 通过 `virt-install` 命令安装镜像（因为使用的是非图形化的安装，所以参数有一些不一样）
```bash
virt-install --name guest1_fromiso --memory 2048 --vcpus 2 \
    --disk size=8 --location CentOS-7-x86_64-DVD-2003.iso \
    --os-type Linux --os-variant=centos7.0  --virt-type kvm \
    --boot menu=on  --graphics none --console pty --extra-args 'console=ttyS0'
```
其中要注意几个参数：
* 因为我们是安装非图形化，所以需要 `--location` 参数指定 iso，并指定 `--boot menu=on` 打开安装菜单，最后还需要指定安装信息的输出 `--console pty --extra-args 'console=ttyS0'` 这样安装菜单才能正常展示出来
* `--graphics none` 指定非图形化；
* `--network bridge=virbr0`，指定网络模式，这里指定 libvirt 默认创建的 bridge 网卡，可以认为这是一个 libvirt 维护内网，安装时选择 dhcp 就可以获得一个可用的内网地址；<br>
具体 libvirt 的网络模型，后面在单独研究下。
* `-- disk size=8`，disk 参数用于指定系统盘，这里指定自动创建一个 8G 的 qcow2 文件，作为系统盘（默认镜像文件保存在 /var/lib/libvirt/images/）目录下；
这时就会进入虚拟机的安装步骤，具体安装步骤就不赘述了。
{{< find_img "img1.png" >}}
3. 安装成功后，可以看到 domain 就被创建了，这就可以得到它的配置文件与镜像文件了。
```bash
$ virsh list
Id    Name                           State
----------------------------------------------------
18    guest1_fromiso                 running
```

### 1.2 自动安装
可以看到，手动安装需要人为在菜单中选择、配置，这不适用于多个虚拟机的安装。而 RedHat 创建了 kickstart 安装方法，使得整个虚拟机安装流程变得自动化。<br>
这块不了解，具体见红帽官方文档：[KICKSTART INSTALLATIONS](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/6/html/installation_guide/ch-kickstart2#s1-kickstart2-whatis)



## 2 使用 Cloud Image
当然，上面制作过程中耗时都在安装系统上了，而各个云厂商的虚拟机数量那么多，肯定不会一台台去安装操作系统了。所以，目前最常见的都是直接下载已经安装过系统的虚拟机镜像文件。<br>
但是这样的虚拟机是没有特殊配置的，例如密码、hostname 都是一致的，所以 **cloud-init** 出现，用于在第一次启动虚拟机时进行系统的配置。<br>
所以，最快速的制作方法就是：虚拟机镜像文件 + cloud-init 配置。<br>

### 2.1 cloud-init
下面内容都来自于文档，这里仅为自己做个记录。<br>
首先明确 cloud init 的功能：系统第一次启动时，cloud init 相关的进程会根据配置信息去进行系统的配置，包括：设置 hostname、ssh key、password 等；
#### (1) 基本概念
* **metadata**：包含服务器信息，用于 cloud-init 配置；
* **userdata**：包含 cloud-init 系统配置信息，可以是 文件，脚本，yaml 文件等；
* **datasource**：cloud-init 读取配置数据的来源，包含大部分云厂商，当然，也可以来自本地的文件([NoCloud datasource](https://cloudinit.readthedocs.io/en/latest/topics/datasources/nocloud.html))；
#### (2) 运行过程
cloud init 设置包括五个阶段：
1. Generator
机器启动阶段，systemd 的一个 generator 将会决定是否将 **cloud-init.target**（target 可以简单认为特定事件下触发的一组 unit）包含在启动过程中。这就表示启动 cloud-init。<br>
默认情况下，generator 会启动 cloud-init，除非以下情况：
* `/etc/cloud/cloud-init.disabled` 文件存在；
* 内核启动命令行 `/proc/cmdline` 中包含 "cloud-init=disabled"。如果是容器中运行，会读取环境变量 `KERNEL_CMDLINE`；
而下面的步骤，就是由 target 包含的各个 unit 执行的。
2. Local
由 **cloud-init-local.service** 执行，主要目的：找到 "local" 的 datasource，根据配置网络。<br>
配置网络有三种情况：
   1. 首先，根据传入配置 "metadata" 配置网络；
   2. 当上面情况失败，直接配置 "dhcp on eth0"；
   3. 如果 /etc/cloud/cloud.cfg 配置文件中禁用了网络：network: {config: disabled}，那么就不进行网络配置；
3. Init、Config、Final 阶段
对应 service 为 **cloud-init.service**、**cloud-config.service**、 **cloud-final.service**。<br>
通过 local 阶段，网络已经配置好了，并且已经得到了 metadata。而 `/etc/cloud/cloud.cfg` 配置定义了剩下三个阶段对应的任务，也就是 module。
{{< find_img "img2.png" >}}
cloud init 通过一些缓存信息来判断机器是否经过初始化，通过 `cloud-init clean` 也可以手动清理缓存信息。<br>
`/var/log/cloud-init.log` 记录了 cloud-init 运行的完整过程。

### 2.2 制作镜像
下面就开始进行镜像制作：
1. 下载 cloud image，这里使用中科大提供的：
```bash
$ wget  https://mirrors.ustc.edu.cn/centos-cloud/centos/7/images/CentOS-7-x86_64-GenericCloud-2003.qcow2
```
{{< admonition tip 修改密码 >}}
当然，该镜像其实就可以直接进行 virt-install 来启动（因为我们没有配置文件，所以通过 virt-install 来启动并生成配置文件），但是不知道密码，也搜不到，无法登陆进入。不过你也可以使用下面命令来设置密码后进行登陆：
```bash
$ virt-customize -a CentOS-7-x86_64-GenericCloud-2003.qcow2  --root-password password:yourpassword
```
{{< /admonition >}}
2. 因为 cloud-init 需要一个 datasource，而我们没有使用云厂商，所以使用 NoCloud 形式，按照官方的s示例创建一个 disk 文件。
```bash
# 创建 user-data 与 meta-date 配置文件
$ cat meta-data
instance-id: guest1
local-hostname: guest1

$ cat user-data
#cloud-config
chpasswd:
    expire: false
    list: |
    root: password1
ssh_pwauth: True
	
# 生成 disk 文件，包含 userdata 与 metadata 配置数据
$ genisoimage  -output seed.iso -volid cidata -joliet -rock user-data meta-data
```
3. 创建并运行虚拟机。
```bash
$ virt-install --memory 2048 --vcpus 2 --name guest2 \
    --disk CentOS-7-x86_64-GenericCloud-2003.qcow2 --disk seed.iso \
    --os-type Linux --os-variant centos7.0 --virt-type kvm \
    --graphics none --network default \
    --import
```
几个比较重要的参数：
* `--disk CentOS-7-x86_64-GenericCloud-2003.qcow2`：制定系统盘；
* `--import`：跳过安装过程，因为已经安装好操作系统，不需要进行安装过程；
* `--disk seed.iso`：传递 cloud-init datasource 信息；
虚拟机启动过程中，可以看到 cloud-init 配置信息的一些打印：
{{< find_img "img3.png" >}}
最后根据配置的密码成功进入：
{{< find_img "img4.png" >}}
4. 而 **CentOS-7-x86_64-GenericCloud-2003.qcow2** 就是虚拟机经过配置的镜像文件，而 libvirt 启动所需的配置文件就是 `/etc/libvirt/qemu/guest1.xml`。


## 参考
* [CREATING GUESTS WITH VIRT-INSTALL](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/7/html/virtualization_deployment_and_administration_guide/sect-guest_virtual_machine_installation_overview-creating_guests_with_virt_install)
* [KICKSTART INSTALLATIONS](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/6/html/installation_guide/ch-kickstart2#s1-kickstart2-whatis)
* [cloud-init Documentation](https://cloudinit.readthedocs.io/en/latest/)
