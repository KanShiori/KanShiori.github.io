# K8s 学习 - 1 - Pod


## 1 Pod 的进程模型

Pod 是一组容器的集合，而为什么实现上需要多个容器？据传，因为开发 Borg 项目的工程师发现，部署的应用往往都是一个进程组，而不是一个进程。换个角度说，部署的业务往往有着多个不同的进程，并且往往需要部署到同一个机器上。

容器是 “**单进程模型**”，在云上代表的是一个进程。而 Pod 是 “**进程组模型**”，在云上代表着就是进程组。
{{< admonition note "“单进程模型”">}}
“单进程模型” 不代表容器只有一个进程，而是指容器无法管理多进程的能力。

你可以将其理解为，Pod 实现了 Supervisor/goreman 这样的功能，将多个进程进行分组管理，这对于业务应用的部署是非常有用的。
{{< /admonition >}}

所以 Kubernetes 是一个操作系统，管理着 Pod “进程组”。

### 1.1 Pause 容器
在创建 Pod 时，会先创建一个 **`pause 容器`**，也称为 **`根容器`** 或者 **`infra 容器`**。

pause 容器使用 k8s.gcr.io/pause 镜像，永远处于暂停，占用着很少的资源。
{{< admonition note "为什么需要 pause 容器？">}}
因为容器共享资源的特殊性。容器在需要共享 namespace 时，需要先创建一个 namespace，然后让其他的容器加入这个 namespace。

这带来的一个问题就是：如果容器 A 先启动，容器 B 加入。那么容器 A 异常重启后，是容器 A 加入容器 B，还是重新启动容器 B 再次加入容器 A。

很难选择，实现起来也很复杂，因为两个容器有着前后顺序的依赖性，而不是“平等的”。
{{< /admonition >}}

Kubernetes 使用一个占用极少资源，非业务容器的 pause 容器首先启动。然后在其 pause 容器配置好 namespace。接着其他的业务容器加入到 pause 容器的 namespace。

因为 pause 容器占用极少资源，也没有任何逻辑，所以其他业务容器重启后，只需要加入 pause 容器的 namespace 即可，不需要担心 namespace 发生变化。
{{< find_img "img1.png" >}}

	
### 1.2 容器共享的资源
Pod 内容器**默认共享 network uts ipc namespace**，所以容器间的网络是相同的。

Pod 可以**配置开启共享 PID namespace**，使得容器之间可以看到对方的进程。
{{< admonition tip "共享 pid namespace">}}
设置 Pod 定义的 `spec.shareProcessNamespace` 为 true 来共享 pid namespace。
{{< /admonition >}}

Pod 通过 Volume 的抽象概念来进行共享存储，将其块设备或者目录提供到每个容器内的不同路径。也就是说，**每个容器看到的目录路径或者块设备是独立的，但是都对应同一个宿主机的目录或块设备。**


## 2 Dynamic Pod 与 Static Pod
### 2.1 Dynamic Pod
大多数情况，我们会通过 ReplicaSet 或者 Deployment 这样的副本控制器来创建 Pod，甚至通过编写 Pod 的定义来手动部署 Pod。这种情况下，**Pod 是由 Kubernetes 控制面管理的**，可以通过 APIServer 启停它。

这样的 Pod 被称为 [动态 Pod]^(Dynamic Pod)。

### 2.2 Static Pod
另一类是特殊的 Pod，被称为 [静态 Pod]^(Static Pod)。

静态 Pod 是由本地 kubelet 创建的，仅仅在 kubelet 的 Node 上运行。**Kubernetes 控制面可以看见，但是无法管理它。kubelet 也不会其进行健康检查。**

创建静态 Pod 有着两种方式：配置文件 和 HTTP 方式。

#### 2.2.1 配置文件方式
在 kubelet 的配置文件中社会中 `staticPodPath` 为一个目录，kubelet 会扫描该目录，读取目录下的 .yaml 或 .json 文件，创建对应的 Pod。
{{< admonition tip "另一种方式">}}
也可以使用 kubelet 启动参数 `--pod-manifest-path` 指定目录，但是这是即将废弃的参数。
{{< /admonition >}}

你在 APIServer 可以看到该 Pod，但是如果你进行删除操作，该 Pod 会一直进入 Pending 状态，但是不会真正被停止。

删除该 Pod 只能通过删除目录下的对应 .yaml 文件。

#### 2.2.2 HTTP 方式
设置 kubelet 的启动参数 `--manifest-url`，kubelet 会定期从该 URL 地址下载文件，然后以 yaml 或 json 格式解析，然后创建对应的 Pod。

## 3 Pod 的生命周期
### 3.1 Pod phase
Pod 的 `status.phase` 属性表明了 Pod 的生命周期阶段：
```yaml
status:
  phase: Running
```
Pod 生命周期包含如下阶段：
状态 | 含义
-|-
**Pending** | Pod 被创建后到 Pod 内所有容器成功创建前，都属于 Pending 阶段，包括了下载镜像期间。
**Running** | Pod 内的所有容器被成功创建，至少一个容器还在运行（即使其他容器异常重启）。
**Succeeded** | Pod 内所有容器成功执行并退出（退出码为 0），并且 Pod 不再会被重新拉起。
**Failed** | Pod 内所有容器都已经停止，并且至少一个容器是异常退出的（退出码非 0）。
**Unknown** | 无法得知 Pod 的状态。大多数情况是 Node 失联导致。

{{< find_img "img2.png" >}}

{{< admonition note "STATUS of kubectl">}}
你会发现通过 `kubectl get pods` 输出的 STATUS 可能并不是上述的 phase，而是 Pod 中容器状态的一个原因，即 `status.containerStatuses[].state.<>.reason`，这是为了直观输出 Pod 的异常原因。

通过 `kubectl get pods -o yaml` 可以看到 `status.phase`。
{{< /admonition >}}

### 3.2 Container State
Kubernetes 会跟踪 Pod 下每个容器的状态，位于 `status.containerStatuses.state` 字段。

容器可能的状态有：
状态 | 含义
-|-
**Waiting** | 等待运行。可能由于正在拉取 image，或者等待 init container 运行等情况。
**Running** | 容器正在运行中。
**Terminated** | 容器运行结束，无论正常退出还是异常退出。

通过 `status.containerStatuses.state` 你可以看到为啥容器处于该状态。

#### 3.2.1 容器声明周期回调
Kubernetes 提供两个容器生命周期的回调：
* **postStart** ：在容器创建后立即异步执行，因此不保证在容器的 entrypoint 前执行。

  但是 kubelet 会等待 postStart 后执行完，才将容器的状态变为 Running。
* **preStop** ：在 kubelet 发送容器停止信号前，执行的操作。
{{< admonition note Note>}}
注意，preStop 仅仅在 kubelet 停止容器的情况下才会执行（因为这是 kubelet 执行的操作），而容器自己退出的情况下不会执行。
{{< /admonition >}}

使用回调的示例如下：
```yaml 
apiVersion: v1
kind: Pod
metadata:
  name: lifecycle-demo
spec:
  containers:
  - name: lifecycle-demo-container
    image: nginx
    lifecycle:
      postStart:
        httpGet:
          path: /
          port: 80
      preStop:
        exec:
          command: ["/bin/sh","-c","nginx -s quit; while killall -0 nginx; do sleep 1; done"]
```

可以看到，回调可以配置两种执行方式：
* `httpGet` ：执行一次 HTTP GET 请求；
* `exec` ：执行命令行；

### 3.3 Pod Condition
Pod 的 `status.conditions` 表明了 Pod 的一些细节情况。

默认包含以下的 condition：
Conditon | 含义
-|-
**PodScheduled** | Pod 是否已经被调度到某个节点。
**Initialized** | 所有 Init Container 是否成功执行。
**ContainersReady** | 所有容器是否是就绪的（运行中）。
**Ready** | Pod 是否能够提供服务，即能否加入到 Service 的 Endpoints 中。<br> Ready 状态受到 [**Readiness Probe**](#42-readinessprobe) 的控制。

```yaml
status:
  conditions:
  - lastProbeTime: null
    lastTransitionTime: "2021-06-11T07:11:46Z"
    status: "True"
    type: Initialized
  - lastProbeTime: null
    lastTransitionTime: "2021-06-12T15:32:39Z"
    message: 'containers with unready status: [pd]'
    reason: ContainersNotReady
    status: "False"
    type: Ready
  - lastProbeTime: null
    lastTransitionTime: "2021-06-12T15:32:39Z"
    message: 'containers with unready status: [pd]'
    reason: ContainersNotReady
    status: "False"
    type: ContainersReady
  - lastProbeTime: null
    lastTransitionTime: "2021-06-11T07:11:46Z"
    status: "True"
    type: PodScheduled
```
可以看到每个 Condition 都包含了如下信息：
* `type` ：condition 类型；
* `status` ：condition 是否正常，包括 True False Unknown；
* `lastProbeTime` ：上一次检查该 condition 的时间；
* `lastTransitionTime` ：上一次 condition status 变化的时间；
* `reason` ：上一次 condition status 变化的原因；
* `message` ：上一次 conditon status 变化的原因信息；

#### 3.3.1 自定义 condition
除了默认的 condition 外，可以通过 `spec.readinessGates` 来自定义 condition。

自定义 condition 往往由一个 controller 控制，**通过自定义 controller 来主动将自定义 condition 设置为 true。**

自定义 condition 仅仅会影响到 Ready condition，**只有所有自定义 condition 为 true 后，Ready condition 才能变为 true。**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: centos
spec:
  readinessGates:
    - conditionType: "www.example.com/feature-1"
```

### 3.4 Pod 重启策略
通过 `spec.restartPolicy` 指定 Pod 的重启策略，其可能的值为：
* **Always** ：容器退出时，kubelet 自动重启该容器；
* **OnFailure** ：容器退出且退出码非 0 时，kubelet 自动重启容器；
* **Never** ：任何容器提出，kubelet 都不重启容器；

kubelet 重启容器的时间间隔以指数的方式增长（1n、2n、4n …），最长延迟 5min。如果容器运行了 10min 没有退出，则间隔时间会被重置。
{{< admonition tip "重启时间间隔单位">}}
重启间隔时间是以 kubelet handle 周期为单位。
{{< /admonition >}}

## 4 Probe
Probe 以旁路的方式，周期性地检测 Pod 容器的情况，检测失败是及时更新 Pod 的状态，使得上层感知后进行一些恢复操作。

目前包含三种探针：LivenessProbe、ReadinessProbe 和 StartupProbe。

{{< admonition note 容器维度>}}
要明确，Probe 是以容器为对象的，而不是 Pod。
{{< /admonition >}}

### 4.1 LivenessProbe
**`LivenessProbe`** 用于周期性判断一个容器是否是存活的，**如果 LivenessProbe 探测失败，那么 kubelet 将会 “杀掉” 该容器。**

容器停止后，后续的操作是由 [**重启策略**](#34-pod-重启策略) 控制的。

### 4.2 ReadinessProbe
**`ReadinessProbe`** 用于周期性判断一个容器是否是 Ready，**从而影响 Pod 的 Ready condition 是否为 True。**

只有 Pod Ready condition 为 True 时，Service 才可以将其 Pod 加入到 Endpoints，从而该 Pod 才能接受请求并处理。

同样，当 Pod Ready condition 从 True 变为 False 时，Service 会将其 Pod 从 Endpoints 移除。
{{< admonition note "为何还需要 Readiness Gate？">}}
因为 ReadinessProbe 是一种被动探测的方式，而 [**Readiness Gate**](#331-自定义-condition) 提供了一种主动修改的方式。
{{< /admonition >}}

### 4.3 StartupProbe
**`StartupProbe`** 仅仅在容器启动后会运行，并且**成功后不会再次运行，直到容器重新启动**。

StartupProbe 是为了解决有些容器的启动情况很慢的情况，这种情况只需要进行一次成功的探测，所以不适合 ReadyinessProbe 的周期性语义。

### 4.4 使用 Probe
三种探针都可以使用三种探测方式：
* `exec` ：执行一个 shell 命令，退出码为 0 表明容器健康；
  ```yaml
	spec:
	  containers:
	  - name: liveness
	    image: k8s.gcr.io/liveness
	    livenessProbe:
	      exec:
	        command:
	        - cat
	        - /tmp/healthy
  ```
* `tcpSocket` ：连接 TCP 的一个 IP 与 Port，连接成功表明容器健康；
  ```yaml
	spec:
	  containers:
	  - name: goproxy
	    image: k8s.gcr.io/goproxy:0.1
	    ports:
	    - containerPort: 8080
	    readinessProbe:
	      tcpSocket:
	        port: 8080
  ```
* `httpGet` ：发送一个 HTTP Get 请求，状态码 [200, 400) 表明容器健康；
  ```yaml
	spec:
	  containers:
	  - name: liveness
	    livenessProbe:
	      httpGet:
	        path: /healthz
	        port: 8080
	        httpHeaders:
	        - name: Custom-Header
	          value: Awesome
  ```

每种探测器也包含如下的配置参数，用以控制探测的行为：
* `initialDelaySeconds` ：容器启动后多久开始进行探测。

  仅仅适用于 LivenessProbe 与 ReadinessProbe。
* `periodSeconds` ：执行探测的时间间隔。默认 10s。
* `timeoutSeconds` ：每次探测的超时时间。默认 1s。
* `successThreshold` ：探测失败后，经过几次探测成功后，才将其变为健康状态，默认值为 1。

  仅仅适用于 StartupProbe。
* `failureThreshold` ：连续探测失败多少次后，才将其容器视为不健康状态，默认值为 3。


## 5 Init Container
大多数应用在启动前都需要进行一些初始化的操作。Kubernetes 提供了 init container 来进行 Pod 的初始化操作。

**`init container`** 也是普通的容器，但是仅仅只运行一次就结束。多个 init container 的情况下，**按照定义的顺序运行，并且只有前面的运行成功后才会运行后面的。**

某个 init container 运行失败后，其 Pod 的启动流程会终止。而其后的操作就是受到 [**重启策略**](#34-pod-重启策略) 的控制。
{{< find_img "img3.png" >}}
	
init container 在 `spec.initContainers` 中定义：
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
  labels:
    app: myapp
spec:
  containers:
  - name: myapp-container
    image: busybox:1.28
    command: ['sh', '-c', 'echo The app is running! && sleep 3600']
  initContainers:
  - name: init-myservice
    image: busybox:1.28
    command: ['sh', '-c', "until nslookup myservice.$(cat /var/run/secrets/kubernetes.io/serviceaccount/namespace).svc.cluster.local; do echo waiting for myservice; sleep 2; done"]
  - name: init-mydb
    image: busybox:1.28
    command: ['sh', '-c', "until nslookup mydb.$(cat /var/run/secrets/kubernetes.io/serviceaccount/namespace).svc.cluster.local; do echo waiting for mydb; sleep 2; done"]
```

init container 与应用容器的区别如下：
* 启动顺序。init container 全部启动结束后，Kubernetes 才会初始化 Pod 各种信息，然后启动应用容器
* 资源限制，见：[**Resource**](https://kubernetes.io/docs/concepts/workloads/pods/init-containers/#resources)。
* init container 不能设置 ReadinessProber。
* Pod 重新启动时，init container 将会重新运行。所以 init container 应该是幂等的。


## 6 Ephemeral Container
有时由于容器崩溃或者容器镜像不包含 debug 工具，使得通过 `kubectl exec` 方式不能很好的进行调试。v1.18 版本开始，新添加一个 `kubectl debug` 命令，用于创建 **`ephemeral container`** 来进行 debug。
{{< admonition tip "开启特性">}}
需要开启 EphemeralContainers feature gate
{{< /admonition >}}

使用 `kubectl debug` 的示例如下：
```bash
$ kubectl debug -it ephemeral-demo --image=busybox --target=ephemeral-demo
```
* --image ：临时容器使用的镜像；
* --target ：加入的 Pod 中特定容器的 namespace；

也可以复制一个 Pod，然后添加临时容器来进行调试：
```bash
$ kubectl debug myapp -it --copy-to=myapp-debug --container=myapp -- sh
```
* --copy-to ：将 pod 复制一份新命名的 pod；

你可以让临时容器和应用容器共享 pid namespace，这样可以进行类似 strace 这样的进程排查。
```bash
kubectl debug myapp -it --image=ubuntu --share-processes --copy-to=myapp-debug
```
* --share-processes ：使用 --copy-to 时，开启 pid namespace 共享；

