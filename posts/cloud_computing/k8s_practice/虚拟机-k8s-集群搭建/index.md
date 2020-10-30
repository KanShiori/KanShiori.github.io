# 虚拟机 k8s 集群搭建


## 1 虚拟机集群搭建
目标：创建 3 个虚拟机，用作一个 Master Node，两个 Work Node；当然，三个节点处于同一个网段。
具体步骤如下:
1. 构建节点 <br>
构建三个虚拟机，基于 centos 7、内存 2 GB，并通过虚拟机复制功能（其实就是 copy 系统盘），完全复制出 Node 1，Node 2，Node 3。
    {{< find_img "image1.png" >}}
2. 搭建网络 <br>
三个节点需要互相访问，所以将其位于 VirtualBox 创建的 Nat网络下，给予每个 Node  静态的 IP（10.0.2.10 - 10.0.2.12），为了方便访问，并设置 ssh 的 DNAT。
    {{< find_img "image2.png" >}}
设置每个虚拟机网卡加入其创建的 "NodeNatNetwork"。例如：
    {{< find_img "image3.png"  >}}
启动每个虚拟机，设置其 hostname，与网卡静态 IP。例如：
    {{< find_img "image5.png"  >}}

至此，三个虚拟机位于同一个网段，并且能够相互访问；对外，则通过 VirtualBox 的 Nat 网络能够访问。

## 2 部署 K8s
目标：通过 kubeadm 部署整个 k8s，用 Node-1 为 Master 节点，其他为工作节点。

### 2.1 安装 kubeadm、kubelet、kubectl
安装 kubeadm、kubelet、kubectl。这个官方文档写的很详细，见 [Installing kubeadm](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/) 。

### 2.2 kubeadm init 初始化 Master 节点
Node-1 节点执行 kubeadm init，将其作为 Master 节点初始化。执行成功后，kubeadm 生成了 Kubernetes 组件的各个配置，以及提供服务的各类证书，位于 /etc/kubernetes 目录下:
  {{< find_img "image7.png"  >}}
并且已经以 static pod 的形式启动了：apiserver、controller-manager、etcd、scheduler。
  {{< find_img "image8.png"  >}}
还有最重要的，kubeadm 为集群生成一个【bootstrap token】，需要加入集群的节点都需要通过这个 token 加入。
  {{< find_img "image9.png"  >}}
#### * 问题
1. kubeadm 检查 swap 打开着，kubeadm 推荐不使用 swap，通过 `swapoff -a` 关闭交换区。
  {{< find_img "image11.png"  >}}
2. kubectl 默认通过 8080 端口访问，无法执行。	设置 kubectl 的配置文件为 kubeadm 生成的 /etc/kubernetes/admin.conf。（其实就是配置公钥，或者将 admin.conf 移动到 ~/.kube/config 文件，作为默认配置。
  {{< find_img "image12.png"  >}}

### 2.3 kubeadm join [token] 设置工作节点
通过 kubead init 最后返回的提示信息，执行对应 `kubeadm join` 将 Node-1 Node-2 加入到集群中，作为工作节点。
```bash
[root@Node-2 kubeadm join 10.0.2.10:6443 --token mahrou.d3uodof21i3d6yxk 
--discovery-token-ca-cert-hash sha256:21dfe4ef6b3bbd89f803bf44ff6eda587874336d103d0e4a3b --v 5
```
可以看到，kubelet 启动后就通过 pod 方式启动了本节点上 kube-proxy 容器：
  {{< find_img "image14.png" >}}
#### * 问题
1. 无法访问到 Node-1 节点，nc ip 失败，但是可以 ping 通。通过在 Node-1 `tcpdump` 可以抓取到来自 Node-3 的包，因此应该是防火墙的问题，通过 `iptables` 对 Node-2 Node-3 IP 开放。
2. kubectl 无法访问问题，与上述问题一致。

### 2.4 结果
目前为止，就完成了集群的搭建，但是 通过 kubectl get nodes，可以看到所有节点都是 NotReady：
  {{< find_img "image17.png"  >}}
`kubectl describe node node-1` 可以看到，原因是因为没有设置正确的 Network Plugin：
  {{< find_img "image18.png"  >}}


## 3 部署网络插件
目的：部署网络插件，使各个节点为 Ready 状态，并其内部 Pod 能够相互通信。<br>
以 Weave 部署为例，部署网络插件：
```bash
kubectl apply -f "https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version | base64 | tr -d '\n')"
```
其描述文件中定义所有 Weave 需要的 BAAC 权限组件，以及最重要的网络插件 Pod 对应的 DaemonSet:
  {{< find_img "image19.png"  >}}
应用成功后，可以看到对应的 DaemonSet 就运行起来，并开始给三个 Node 部署 Pod:
  {{< find_img "image20.png"  >}}
在节点上，可以看到 weave-net 对应的 pod ，包括两个容器：
  {{< find_img "image21.png"  >}}


## 4 部署容器存储插件
目的：为了能够让容器使用网络存储，使得容器数据持久化，需要部署存储插件。 <br>
以 Rook 项目为例，部署存储插件：
```bash
$ kubectl apply -f https://raw.githubusercontent.com/rook/rook/master/cluster/exampleskubernetes/ceph/common.yaml
$ kubectl apply -f https://raw.githubusercontent.com/rook/rook/master/cluster/exampleskubernetes/ceph/operator.yaml
$ kubectl apply -f https://raw.githubusercontent.com/rook/rook/master/cluster/examples/kubernetes/ceph/cluster.yaml
```
安装成功后，可以看到，rook 有着自己的 namespace，并且已经部署了 DaemonSet：
  {{< find_img "image22.png"  >}}
可以看到，Pod 也部署成功了：
  {{< find_img "image23.png"  >}}
















  
