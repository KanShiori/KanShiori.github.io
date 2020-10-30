# Deployment 实践


## 1 ReplicaSet
Deploment 管理的是 ReplicaSet，所以先运行 ReplicaSet 观察。
ReplicaSet 只包含副本控制功能，没有滚动升级等高级的功能。

### 1.1 部署 ReplicaSet
1. 创建 manifest 文件。
    {{< find_img "image1.png"  >}}
2. 调用 `kubectl create` 创建资源。
    {{< find_img "image3.png"  >}}
    {{< find_img "image3.5.png"  >}}
观察下 ReplicaSet 的事件，可以看到各个 Pod 的创建流程。
    {{< find_img "image4.png"  >}}

### 1.2 副本保证
目前，3 个 Pod 都运行在了 Node-3 上：
    {{< find_img "image5.png"  >}}
1. 下线 Node-3 ，可以看到 node-3 变为 NotReady:
    {{< find_img "image6.png"  >}}
2. 过了一段时间后，原来三个 Pod 变为 Terminating 状态，而新的 Pod 被创建被调度。
    {{< find_img "image7.png"  >}}
    可以看到，新创建 3 个 Pod 被调度到了 Node-2 上:
    {{< find_img "image8.png"  >}}
3. 恢复 Node-3 上线，kubelet 会同步任务，因此不会再次运行旧的三个 Pod。Pod 的状态也从 Terminating 变为被删除。
    {{< find_img "image10.png"  >}}

### 1.3 水平缩扩
1. 通过 `kubectl scale` 进行副本扩展:
{{< find_img "image11.5.png"  >}}
可以看到，新的 Pod 在 Node-3 被运行：
{{< find_img "image12.png"  >}}
2. 依旧 `kubectl scale` 进行副本缩容，可以看到，两个 Pod 被停止: 
{{< find_img "image13.5.png"  >}}
{{< find_img "image14.png"  >}}
可以看到，ReplicaSet 启动和停止任务都是由 Scheduler 选择的，而不能认为的控制选择指定的 Pod，也就是说，所有的 Pod 应该被认为是“无状态的”，随时可能被停止。


## 2 Deployment
Deployment 操作与管理的是 ReplicaSet，而不是 Pod。

### 2.1 部署
1. 构建 manfiest 文件:
    {{< find_img "image15.png"  >}}
2. 通过 `kubectl create` 创建 Deployment。
    {{< find_img "image16.png"  >}}
    可以看到，Event 中打印的是对应的ReplicaSet 的自动被创建，所以Deployment 是创建 ReplicaSet 的。
    {{< find_img "image18.png"  >}}

### 2.2 水平扩展
Deployment 水平扩展方式与 ReplicaSet 一致，并且就是操作 ReplicaSet 的 replica 的值来实现，跳过。

### 2.3 滚动升级
#### (1) 正常流程
Deployment 在 ReplicaSet 基础上添加了“滚动升级”的功能。

1. 依旧创建 Deployment，并直接扩展到副本数为 3。
    {{< find_img "image20.png"  >}}
2. 修改 Deployment 的配置文件，将 image 版本升级。
    {{< find_img "image21.png"  >}}
可以看到，Deployment 新建了一个 ReplicaSet （部署新版本 Pod），而旧的 ReplicaSet 副本变为了 0。
    {{< find_img "image22.png"  >}}
通过 Deployment 的 Event 可以看到，旧版 ReplicaSet 的副本数逐渐减少，而新版本 ReplicaSet 副本数逐渐增加。这样使得集群中 Pod 会维持在一个最低数量（示例中为 3）
    {{< find_img "image23.png"  >}}

#### (2) 错误流程
观察下当升级出现错误时，Deployment 会处于怎样的状态。
1. 通过 kubectl set image 将 Deployment 使用镜像变为一个不存在的镜像。
    {{< find_img "image24.png"  >}}
    通过 Event 看到，滚动升级停止在了最新版本的 Replicaset 的第一个副本部署。
    {{< find_img "image25.png"  >}}
    因为新旧版本是交替部署的，所以当第一个副本部署失败时，也就不会继续进行旧版本 Pod 的停止了。

### 2.4 回滚
在错误流程看到，当发布错误的版本后，Deployment 会停止新版本的发布，而这时，就需要通过 `kubectl rollout` 进行 Deployment 的回滚。

1. 执行 `kubectl rollout history`，查看每次 Deployment 变更对应的版本。（因为 -record，所有的 kubectl 命令都会被记录）。<br>
可以看到，两次的版本变更都被记录了下来。
    {{< find_img "image26.png"  >}}
通过 --revision 参数，查看对应的命令细节。
    {{< find_img "image27.png"  >}}
2. 通过 `kubectl rollout undo` 进行版本的回退，默认为上一次版本，通过 --to-revision 可以执行回退的版本。
    {{< find_img "image28.png"  >}}
事实上，回退也是一次“升级”，通过 history 可以看到一个新的部署记录：
    {{< find_img "image29.png"  >}}
这个 4，就是最新的一次回滚执行的命令了。


