# Kubernetes - Rolling Update 与 Rollback


## 1 Deployment

### 1.1 操作

#### 1.1.1 Update

当用户修改 Deployment 定义中的 `.spec.template` 任意字段时，就会触发 Deployment 的滚动升级。

```bash
$ kubectl set image deployment.v1.apps/nginx-deployment nginx=nginx:1.16.1
```

通过 `kubectl rollout status` 查看滚动升级的过程：

```bash
$ kubectl rollout status deployment/nginx-deployment

Waiting for deployment "nginx-deployment" rollout to finish: 1 out of 3 new replicas have been updated...
Waiting for deployment "nginx-deployment" rollout to finish: 1 out of 3 new replicas have been updated...
Waiting for deployment "nginx-deployment" rollout to finish: 2 out of 3 new replicas have been updated...
Waiting for deployment "nginx-deployment" rollout to finish: 2 out of 3 new replicas have been updated...
Waiting for deployment "nginx-deployment" rollout to finish: 2 out of 3 new replicas have been updated...
Waiting for deployment "nginx-deployment" rollout to finish: 1 old replicas are pending termination...
Waiting for deployment "nginx-deployment" rollout to finish: 1 old replicas are pending termination...
deployment "nginx-deployment" successfully rolled out
```

#### 1.1.2 Rollback

默认情况下，Deployment 的每次更新的版本都会保留在 Kubernetes 中，实际上是保留所有的 ReplicaSet。

```bash
$ kubectl get rs

NAME                                DESIRED   CURRENT   READY   AGE
nginx-deployment-9456bbbf9          3         3         3       47m
nginx-deployment-ff6655784          0         0         0       45m
```

使用 `kubectl rollout history` 来查看某个 Deployment 的修订历史。

```bash
$ kubectl rollout history deployment/nginx-deployment

deployment.apps/nginx-deployment
REVISION  CHANGE-CAUSE
1         <none>
2         <none>
```

> 更新时使用 `--record=true` 参考可以记录更新的命令，并记录在 Change Cause 中。

使用 `--revision` 参数具体查看某个版本的 Pod template。

```bash
$ kubectl rollout history deployment/nginx-deployment --revision 2

deployment.apps/nginx-deployment with revision #2
Pod Template:
  Labels:       app=nginx
        pod-template-hash=ff6655784
  Containers:
   nginx:
    Image:      nginx:1.16.1
    Port:       80/TCP
    Host Port:  0/TCP
    Environment:        <none>
    Mounts:     <none>
  Volumes:      <none>
```

使用 `kubectl rollout undo` 来回滚到上一个版本，或者使用 `--to-revision` 参数回滚到特定的版本。

```bash
$ kubectl rollout undo deployment/nginx-deployment

deployment.apps/nginx-deployment rolled back
```

#### 1.1.3 流程控制

Deployment 升级流程中，会交替更新旧的与新的 ReplicaSet。有一些参数可以控制滚动升级。

`spec.strategy.type` 策略用于指定如何进行 Pod 升级：

* `RollingUpdate` - 默认值，滚动升级；
* `Recreate` - 先删除所有旧 Pod，后启动新 Pod；

在使用 `RollingUpdate` 策略下，可以通过下面两个字段来控制滚动升级的速率：

```yaml
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 10%
      maxSurge: 10%

```

* `maxUnavailable` - 指定升级过程中，不可用的 Pod 的个数上限。支持数字或者百分比（向下取整），默认值 25%。
  
  例如 `maxUnavailable: 30%` 时，那么滚动升级期间，会确保可用的 Pod 保持为总数的 70%。

  {{< image src="img1.png" height=200 >}}  

* `maxSurge` -  指定升级过程中，允许创建的超出期望的 Pod 的个数。支持数字或者百分比（向上取整），默认值为 25%。
  
  例如 `maxSurge: 30%` 时，表明升级过程中可能所有的 Pod 数量会是期望的 130%，然后再会交替的进行滚动升级。

  {{< image src="img2.png" height=200 >}}  

{{< admonition note Note>}}
Deployment 设计上是用于管理无状态应用的，因此滚动升级中通过多创建一些 Pod 作为 Buffer，以获得更好的升级体验。而 StatefulSet 相反，因为运行的是有状态的，因此整个滚动升级过程中，Pod 总数是不变的。
{{< /admonition >}}

其他一些相关的参数包括：

```yaml
spec:
  revisionHistoryLimit: 3
  minReadySeconds: 30
  paused: false
```

* `revisionHistoryLimit` - 保留的历史版本数量（也就是 ReplicaSet 数量），默认为 10。
  
  如果为 0 则表示不保留任何版本，并且会立即清理。

* `minReadySeconds` - 滚动升级过程中，新创建的 Pod 就绪后的等待时间。
  
  每当新创建的 Pod 为就绪状态后，等待 `minReadySeconds` 秒后会继续进行下一个 Pod 的升级。如果为 0，那么表明 Pod 就绪后立即可用。

* `paused` - 暂停 Deployment 的升级，修复 Pod template 不会触发滚动升级。
  
  使用 `kubectl rollout pause/resume` 本质上就是修改 Deployment 的该字段。

### 1.2 实现

我们来看一下 Deployment 管理的背后实现，分三个阶段：

* Trigger - 如何判断需要进行 Update
* Update - 如何实现 Rolling Update
* Rollback - 如何实现 Rollback

Deployment 相关代码都位于 [**Deployment Controller**](https://github.com/kubernetes/kubernetes/blob/master/pkg/controller/deployment/deployment_controller.go)。

在说明各个阶段之前，先大致看下 Deployment 如果管理多个 ReplicaSet：

{{< image src="img3.png" height=350 >}}

* Deployment 与 ReplicaSet
  
  ReplicaSet 与 Deployment 通过 `ownerReferences` 来表明所属关系，通过 `revision` annotation 来表示 ReplicaSet 的修订版本。

* ReplicaSet 与 Pod
  
  基于 Pod Template 会生成一个 Hash 值，设置到 ReplicaSet 的 Labels 中，并该 Label 作为筛选 ReplicaSet 管理的 Pod。

{{< admonition note Note>}}
hash 值其实也是 ReplicaSet 命名的一部分:

```yaml
apiVersion: apps/v1
kind: ReplicaSet
  labels:
    pod-template-hash: ff6655784
  name: nginx-deployment-ff6655784
```
{{< /admonition >}}

#### 1.2.1 Trigger

当用户修改 Deployment 的 Pod Template 定义后，Controller 判断出当前 Deployment 与任何 ReplicaSet 的 Pod Template 不相同，那么就会创建新的 ReplicaSet。

{{< image src="img4.png" height=300 >}}

新的 ReplicaSet 的 revision 会加 1，使得整体有了版本的概念。

#### 1.2.2 Update

Rolling Update 流程就是一个交替缩扩容的流程：

1. 扩容 New RS
   
   满足当前所有 ReplicaSet 的 Pod 总数不能超过 `desired replica` + `max surge` 前提下，扩容 New RS。

2. 缩容 Old RS
   
   满足当前所有 ReplicaSet 的 Pod 总数不低于 `desired replica` - `max unavailable` 前提下，缩容 Old RS。

3. 循环执行，直到 New RS 满足 `desired replica`，那么剩余的 Old RS Pod 可以被删除。

{{< image src="img5.png" height=250 >}}

#### 1.2.3 Rollback



## DaemonSet

## StatefulSet

### Controller Revisions

### 原理

## PodDisruptionBudget 

## 参考

* 官方文档：[**Deployments**](https://kubernetes.io/zh/docs/concepts/workloads/controllers/deployment/)


