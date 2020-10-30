# PV PVC 与 StorageClass


## 1 PV 与 PVC
目的：使用 NFS 做 PV，创建 Pod 使用该 PV
1. Node-1 构建 nfs 服务，位于 "/nfs" 目录。
2. 创建 PV，使用 nfs 类型。
<br>{{< find_img "img1.png" true >}}
{{< find_img "img2.png" >}}
3. 声明、创建 PVC，使用上述指定的 StorageClass，形成指定的绑定关系。
<br>{{< find_img "img3.png" true >}}
{{< find_img "img4.png" >}}
pvc 状态为 Bound，表明已经成功绑定到了 pvc。查询 pv，可以看到 pv 也是被绑定了。(**注意：一个 PV 只能绑定一个 PV**)
{{< find_img "img5.png" >}}
4. 创建 Pod Deployment，在 Pod 的 Volume 使用 PVC。
<br>{{< find_img "img6.png" true >}}
{{< find_img "img7.png" >}}
其中两个 Pod 分别调度到了 Node-2 Node-3，在 Node-2 Node-3 中可以看到对应的 nfs mount：
{{< find_img "img8.png" >}}
对应容器配置也可以看到指定的挂载：
{{< find_img "img9.png" >}}
5. 在 Node-3 节点的容器环境内写入挂载路径文件，可以看到同步到了主节点的 /nfs 目录上，因此，存储配置成功。
{{< find_img "img10.png" >}}

