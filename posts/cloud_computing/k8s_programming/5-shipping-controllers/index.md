# K8s 编程 - 发布 Operator


> **Kubernetes 编程系列** 主要记录一些开发 Controller 所相关的知识，大部分内容来自于[《Programming Kubernetes》](https://www.oreilly.com/library/view/programming-kubernetes/9781492047094/)（推荐直接阅读）。

## 1 打包

### 1.1 Helm

[**Helm**](https://helm.sh/zh/docs/) 已经成为 Kubernetes 包管理器的事实标准。通过引入一个 **`Chart`** 的概念，帮助安装与升级 Kubernetes 应用。

Chart 实质上是一个参数化的 YAML 文件，下面是 Chart 一个 template 文件的例子：
```yaml
apiVersion: apps/v1 
kind: Deployment 
metadata: 
  name: {{ include "flagger.fullname" . }} 
# ... 
spec: 
  replicas: 1 
  strategy: 
    type: Recreate 
  selector: 
    matchLabels: 
      app.kubernetes.io/name: {{ template "flagger.name" . }} 
      app.kubernetes.io/instance: {{ .Release.Name }} 
  template: 
    metadata: 
      labels: 
        app.kubernetes.io/name: {{ template "flagger.name" . }}         app.kubernetes.io/instance: {{ .Release.Name }} 
    spec: 
      serviceAccountName: {{ template "flagger.serviceAccountName" . }} 
      containers: 
        - name: flagger 
          securityContext: 
            readOnlyRootFilesystem: true 
            runAsUser: 10001 
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
```

可以看到，所有的变量都以 **{{ .Some.value.here }}** 的格式表示。

通过 `helm install` 命令，就可以安装一个 Chart。最简单的方式是安装官方发布的一些 Chart：
```shell
# get the latest list of charts:
$ helm repo update 
 
# install MySQL:
$ helm install stable/mysql 
Released smiling-penguin 
 
# list running apps:
$ helm ls 
NAME             VERSION   UPDATED                   STATUS    CHART 
smiling-penguin  1         Wed Sep 28 12:59:46 2016  DEPLOYED  mysql-0.1.0 
 
# remove it:
$ helm delete smiling-penguin 
Removed smiling-penguin
```

想要发布的你的 Controller，需要编写一个 Helm Chart 并上传到公共仓库。默认情况，你可以把它发布到 [**Artifact Hub**](https://artifacthub.io/) 提供的公共仓库。

### 1.2 Kustomize

[**Kustomize**](https://kustomize.io/) 提供了对资源文件的一种声明式的配置方法。

**Kustomize 可以帮助你自定义 YAML 文件，而不用修改原始的文件**。

例如，你先定义一个 **`kustomization.yaml`** 文件：
```yaml
imageTags: 
  - name: quay.io/programming-kubernetes/cnat-operator 
    newTag: 0.1.0 
resources: 
- cnat-controller.yaml
```

然后创建原始的文件 cnat-controller.yaml 上：
```yaml
apiVersion: apps/v1beta1 
kind: Deployment 
metadata: 
  name: cnat-controller 
spec: 
  replicas: 1 
  template: 
    metadata: 
      labels: 
        app: cnat 
    spec: 
      containers: 
      - name: custom-controller 
        image: quay.io/programming-kubernetes/cnat-operator
```

使用 `kustomize build` 命令，cat-controller.yaml 文件不变，但是会输出如下配置：
```yaml
apiVersion: apps/v1beta1 
kind: Deployment 
metadata: 
  name: cnat-controller 
spec:   replicas: 1 
  template: 
    metadata: 
      labels: 
        app: cnat 
    spec: 
      containers: 
      - name: custom-controller 
        image: quay.io/programming-kubernetes/cnat-operator:0.1.0l
```

### 1.3 最佳打包实践

无论你用什么机制，下面这些经验都很适用：
* 提供一个合适的权限控制设置：一个专门的 ServiceAccount，ServiceAccount 绑定最小权限的 RBAC 许可。
* 确认你的 Controller 的控制范围，一个命名空间还是多个命名空间。参考 [**Twitter**](https://twitter.com/alexellisuk/status/1116273213460426755) 中的讨论。
* 测试和分析你的 Controller 使得你对它运行所需要的资源有所概念。例如，Red Hat 在 [**OperatorHub**](https://operatorhub.io/contribute) 有着详细的规定与说明。
* 确保 CRD 和 Controller 有着齐全的文档。


## 2 生命周期管理

生命周期管理的基本思想：考虑从开发到发布到升级的整个链路，把这个过程的自动化做到极致。为了安装和升级 Operator，你需要一个**专门的 Operator 来处理其他的 Operator**。而这就是所谓的 **`Operator Lifecycle Manager`**，简称 **OLM**。

简而言之，**OLM 提供了一种声明式的方式来安装和升级 Operator 以及其依赖**。

## 3 部署

### 3.1 将权限设置正确

CRD Controller 需要一系列正确的权限，通过 RBAC 相关的设置来实现。

其重点是：你需要创建一个**专门的 ServiceAccount 来运行 Controller**，而不要使用 default ServiceAccount。

最佳实践是：**定义一个拥有必要的 RBAC 规则的 ClusterRole 和一个 RoleBinding，并将其绑定到指定的命名空间**。并且只赋予 Controller 工作的最小权限。

一些实践的经验是，Controller 不应该具有以下功能：
* 对代码里通常是只读资源的写权限。例如，如果你只需要 watch Service 和 Deployment，那么就不需要提供 create、update 等权限。
* 访问所有的 Secret，限制 Role 只能访问必要的 Secret。
* 写入 MutatingWebhookConfiguration 或 ValidatingWebhook 的权限。有了这两个原型都等于拥有了对集群里所有资源的访问权限一样。
* 写入 CustomResourceDefinition 的权限。CRD 的创建应该是由另外的进程创建，而不是控制器自身。
* 写入自己不负责管理的外部资源的 /status 子资源。

{{< admonition tip "audit2rbac">}}
[**audit2rbac**](https://github.com/liggitt/audit2rbac) 工具通过分析审计日志来自动生成合适的权限，这可以帮助你进行权限的控制。
{{< /admonition >}}

### 3.2 自动化构建与测试

当你的 Controller 开发处于稳定时，就可以且必须开始各种测试：
* Kubernetes 自身使用性能相关测试，以及 kboom 工具，都可以用于性能系相关测试。
* 浸泡测试，用于发现长期使用场景下是否出现资源泄漏问题。

作为最佳实践，这些测试应该是你的 CI 流程的一部分。建议看看这篇文章 [**Spawning Kubernetes Clusters
in CI for Integration and E2E tests**](https://xmudrii.com/posts/spawn-kubernetes-ci/)。

### 3.3 可观测性

#### 3.3.1 日志

基于容器的部署方案中，日志一般都打印在 stdout 上，通过 `kubectl logs` 命令查看。

在 Kubernetes 代码库中，有两个广泛使用的方法：
* **logger interface**，httplog.go 里提供了该 interface，以及一个具体的实现（respLogger），它可以捕捉状态或错误等信息。
* [**klog**](https://github.com/kubernetes/klog)，Google glog 的一个分支，是 Kubernetes 项目中使用的结构化日志。

## 参考
* [**《Programming Kubernetes》**](https://book.douban.com/subject/33393681/)
