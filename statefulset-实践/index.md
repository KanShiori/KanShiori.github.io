# StatefulSet 实践


Deployment 对于“无状态”的任务，已经能够做到副本控制，滚动升级功能了。但是 Deployment 无法适用于“有状态”的任务，因为其中 Pod 都是相同的，没有任何的对应关系。
而 StatefulSet 通过“固定命名、域名的 Pod，以及固定的创建顺序”作为基础，加上与命名对应的网络、存储，搭建一个“有状态”Pod 管理。

## 1 HeadlessService
StatefulSet 会使用 HeadlessService 的概念，首先来部署 HeadlessService。

1. 创建描述文件 headless_service
    {{< find_img "img1.png"  >}}
可以看到，Headless 与普通 Service 最大区别是 clusterIP 为 None。
2. 通过 kubectl create 部署 HeadlessService，成功后可以看到：
    {{< find_img "img2.png"  >}}
查看 Endpoint，可对应的 Endpoint 包含了 Node-1 Node-2 的 Pod 的地址了。并且 Endpoint 命名和Service 名字一样。
    {{< find_img "img3.png"  >}}

你按照这样的方式创建了一个 Headless Service 之后，它所代理的所有 Pod 的 IP 地址，都会被绑定一个这样格式的 DNS 记录：

  **\<pod-name>.\<svc-name>.\<namespace>.svc.cluster.local**

但是好像单独使用 Headless Service 是无法访问这些域名的。

## 2 StatfulSet

### 2.1 StaefulSet 的固定域名
下面开始部署 StatefulSet：

1. 先要创建对应的 Headless Service，如前面一样。
    {{< find_img "img4.png"  >}}
2. 创建对应的 StatefulSet 描述文件：
    {{< find_img "img5.png"  >}}
可以看到，StatefulSet 与 Deployment 最大的区别就是指定了 serviceName 字段，指定了使用的 HeadlessService。
3. 创建 StatefulSet 对象，类似于 Deployment，开始创建 Pod 了
    {{< find_img "img6.png"  >}}
但是 StatefulSet 并没有创建任何的 ReplicaSet，所以实现上与 Deployment 不一样：
    {{< find_img "img7.png"  >}}
4. 观察创建出的 Pod，可以看到，其 Pod 命名不是加上随机字符串了，而是有序的数字：
    {{< find_img "img8.png"  >}}
并且，其创建顺序也是有序的，先创建 web-0 ，web-0 运行后并 Ready 后，创建 web-1。
5. 运行一个 busybox 测试容器，执行 nslookup 访问 HeadlessService 为其绑定的域名，可以看到正常返回了。
    {{< find_img "img9.png"  >}}
6. 删除 web-0 Pod，可以看到 StatefulSet 会重新立刻创建同名的 Pod。所以，Pod 名字是固定的
    {{< find_img "img10.png"  >}}
再次通过 web-0.nginx.default.svc.cluster.local 进行域名解析，还是能够正确的解析：
    {{< find_img "img11.png"  >}}
**注意：虽然域名可以正确解析，但是其域名对应的 IP 不是保证固定的，所以不能保存 Pod 的 IP**

### 2.2 StatefulSet 的固定存储
目的：使用 StatefulSet 中的 Template PVC 自动构建持久化存储，并观察其 PVC 是否与 PodName 绑定。

1. 构建 nfs 的 2 个 PV，给 2 个 Pod 准备。（构建过程见\<PV、PVC 与 StorageClass>一节）
    {{< find_img "img12.png"  >}}
2. 创建 StatefulSet，其指定 PVC 模板，使得能够为每个 Pod 自动创建其对应的 PVC。
    {{< find_img "img13.png"  >}}
创建后可以看到，StatefulSet 为 2 个 Pod 创建了对应的 PVC：
    {{< find_img "img14.png"  >}}
可以看到，PVC 是和名字对应的，格式为 [volume name]-[pod name]，因此，PVC 与 Pod 名字的映射关系是固定的。<br>
所以这就实现了，当旧 Pod 删除，新 Pod 被创建后，因为其 Pod Name 没有变，所以就找到了旧 Pod 使用 PVC。


