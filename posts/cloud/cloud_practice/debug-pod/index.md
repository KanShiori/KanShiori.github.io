# Kubernetes 中调试 Pod


## 1 Ephemeral Container

可以使用 `kubectl debug` 来给一个运行中的 Pod 添加一个 Ephemeral Container 运行，并且不需要重启 Pod。

{{< admonition note 版本要求>}}
Ephemeral Container 需要 Kubernetes 1.23 版本以上，目前还是处于 beta 状态。
{{< /admonition >}}

```bash
kubectl debug -it ${pod_name} --image=busybox --target=${container_name}
```

* `-it` 开启 stdin 并支持交互式；
* `--image` 指定使用的 image；
* `--target` 指定容器加入的另一个容器的 process namespace，指定后就能看到目标容器的进程；

执行后，Kubernetes 会为该 Pod 添加一个 `debug-xxxx` 命名的 Ephemeral Container，在 Pod 的定义中就可以看到相关的信息。

```yaml
apiVersion: v1
kind: Pod
metadata:
  # ...
spec:
  containers:
    # source container ...
  ephemeralContainers:
  - image: busybox
    imagePullPolicy: Always
    name: debugger-p9dff
    resources: {}
    stdin: true
    targetContainerName: pump
    terminationMessagePath: /dev/termination-log
    terminationMessagePolicy: File
    tty: true
status:
  # ...
  containerStatuses:
    # ...
  ephemeralContainerStatuses:
  - containerID: containerd://0a2679a36b18f4738398176d2692b9d0c3760a9e9e767612a15a7f37c3bce45b
    image: docker.io/library/busybox:latest
    imageID: docker.io/library/busybox@sha256:caa382c432891547782ce7140fb3b7304613d3b0438834dce1cad68896ab110a
    lastState: {}
    name: debugger-p9dff
    ready: false
    restartCount: 0
    state:
      running:
        startedAt: "2022-03-29T06:25:49Z"
```

不过，Ephemeral Container 无法被动态的移除，你只能靠删除 Pod 来重建这个 Pod：

```bash
kubectl delete pod ${pod_name}
```

## 2 新建 Pod 复制来调试

Ephemeral Container 是比较新的特性，旧版本情况下可以选择为运行中的 Pod 创建一个复制 Pod，通过对复制的 Pod 定义的修改来进行期望的调试。

### 2.1 新建 Pod 复制并添加调试容器

下面命令会新建一个 `${pod_name}` Pod 的复制 Pod，名为 `${copy_pod_name}`。新的 Pod 中添加了一个调试用的容器。

```bash
kubectl debug ${pod_name} -it --image=ubuntu --share-processes --copy-to=${copy_pod_name}
```

* `-it` 开启 stdin 并支持交互式；
* `--image` 指定使用的 image；
* `--share-processes` 表明新的 Pod 中所有容器会共享一个 process namespace；

但是，这种方式不会改动 Pod 中原容器的运行命令，因此只能适用于无状态的容器调试。

### 2.2 新建 Pod 复制并修改原容器配置

下面命令会新建一个 `${pod_name}` Pod 的复制 Pod，名为 `${copy_pod_name}`，并且修改了 `${container_name}` 容器的启动命令。

```bash
kubectl debug ${pod_name} -it --container=${container_name} --copy-to=${copy_pod_name} -- sh
```

通过改变容器的启动命令，我们进入到了一个容器中，进而可以需要的调试。

进一步的，我们可以改变容器的 image，使之能够使用包含调试工具的容器启动。或者，你也可以不修改容器启动命令，只修改容器 image。

```bash
kubectl debug ${pod_name} -it --set-image=${container_name}=ubuntu
```

### 2.3 更多参数

上面各个方式都支持一些参数：

* `--replace=false` - 如果为 true，新建 Pod 复制时会删除原 Pod。
  
  这在 StatefulSet 或者使用 PVC 的一些场景很有用，我们可以直接使用新的 Pod 来替换旧的 Pod。

* `--same-node=flase` - 如果为 true，新建 Pod 会强制调度到原 Pod 相同的 Node。
  
  通过设置 `pod.spec.nodeName` 来实现的。

* `--env=[]` - 为新建的 Pod 设置环境变量。

## 3 Node 上新建调试容器

`kubectl debug` 支持在某个 Node 上新建一个宿主机命名空间的特权 Pod，以支持在 Node 上调试。

```bash
kubectl debug node/${node_name} -it --image=ubuntu
```

Pod 名字基于 Node 的名字生成，运行在宿主机的 IPC Net PID 命名空间，并且 Node 的 rootfs 会挂载到 Pod 的 /host 目录下。

## 参考

* 官方文档：[**Debug Running Pods**](https://kubernetes.io/docs/tasks/debug-application-cluster/debug-running-pod/#before-you-begin)
