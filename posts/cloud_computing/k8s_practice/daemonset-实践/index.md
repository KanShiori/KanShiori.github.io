# DaemonSet 实践


DaemonSet 会为匹配的 Node 运行一个 Daemon Pod，与 Deployment 类最大不同，DaemonSet 没有副本的概念。

目的：部署 DaemonSet，试用 toleration 与 nodeAffinity，观察滚动升级流程。
1. 部署 DaemonSet。其中，是用 nodeAffinity 指定选择调度的节点，使用 toleration 容器 Node 的 taint:
    {{< find_img "img1.png"  >}}
创建后，可以看到，只选择调度到了 node-1 node-2 节点，和配置的 Affinity 匹配：
    {{< find_img "img2.png"  >}}
2. 改变 DaemonSet 使用的镜像版本，观察滚动升级流程:
    {{< find_img "img3.png"  >}}
通过 kubectl rollout status 可以看到，滚动升级流程与 Deployment 过程一致：
    {{< find_img "img4.png"  >}}
观察 Event，可以看到也是按照 delete -> create 的流程进行升级的。
    {{< find_img "img5.png"  >}}

