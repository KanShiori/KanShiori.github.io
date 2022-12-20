# Kustomize


## 1 概述

Kustomize 基于 `kustomization.yaml` 文件来生成一系列的 Kubernetes 资源的 YAML 文件。

{{< image src="img1.png" height=200 >}}

基于这种模式，Kustomize 能够支持的功能大致分为三类：

* **生成 Resource** - 基于一些来源数据，生成出 Resource 定义，例如生成 ConfigMap 与 Secret
* **Cross-Cutting 字段** - 为一批 Resource 设置相同的字段，例如相同的 Namespace，Label 等
* **Patch Resource** - 设置 Patch 字段，以定义 Resource

Kustomize 所有的功能都是在 `kustomization.yaml` 文件中定义的，所有特性见文档：[**Kustomize 功能特性列表**](https://kubernetes.io/zh/docs/tasks/manage-kubernetes-objects/kustomization/#kustomize-%E5%8A%9F%E8%83%BD%E7%89%B9%E6%80%A7%E5%88%97%E8%A1%A8)。

## 2 Resource Template

Resource 生成是 Kustomize 最基本的功能。在 `kustomization.yaml` 中使用 `resource` 字段指定 Resource Template 文件。

```yaml
# kustomization.yaml
resources:
  # file rel path
- deployment.yaml

# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
  labels:
    app: my-app
spec:
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
      - name: app
        image: my-app
```

上面实例中，表明读取 Resource Template 生成一个 Deployment 定义。当然，这里没有使用任何 Kustomize 功能来对 Template 进行修改。因此生成的就是 Template 本身。

`resources` 字段是一个数组，支持读取多个 Resource，生成时将其组合在一个输出中。因此，你可以使用 Kustomize 来组合多个 Resource 定义。

## 3 Generator

与基于 Resource Template 生成资源不同，Kustomize 支持从源数据生成 ConfigMap 或者 Secret，不需要 Template。

{{< image src="img1.png" height=200 >}}

下面的示例中都以 ConfigMap 为例说明。Secret 用法与 ConfigMap 一致，区别在于生成 Secret 会自动为内容进行 Base64 编码。

### 3.1 文件作为数据源

`configMapGenerator` 中的 `files` 用于提供一组文件列表。Kustomize 会读取这些文件的内容，生成 ConfigMap。其中，Key 为文件名，Value 为文件内容。

举个例子，`kustomization.yaml` 表明读取两个文件来生成 ConfigMap。

```yaml
configMapGenerator:
- name: example-configmap-1
  files: # file relative path
  - myname
  - config/myage
```

文件内容如下：

```bash
$ cat myname config/myage
shiori
26
```

执行 `kubectl kustomize ./`，可以得到如下 ConfigMap:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: example-configmap-1-759hdc87fb
data:
  # Key: file name, Value: file content
  myage: |
    26
  myname: |
    shiori
```

### 3.2 环境变量作为数据源

`configMapGenerator` 中的 `envs` 支持读取文件中的环境变量来生成 ConfigMap，并且直接读取本地环境变量的值。

生成的 ConfigMap 中，Key 为环境变量的名字，Value 为环境变量的值。

举个例子，下面 `kustomization.yaml` 表明读取两个文件中的环境变量值。

```yaml
# kustomization.yaml
configMapGenerator:
- name: example-configmap-1
  envs:
  - my-age.env
  - my-name.env
```

文件内容如下，其中 `MY_AGE` 省略了赋值，表明读取本地的环境变量。

```bash
$ cat my-age.env my-name.env
MY_AGE
MY_NAME=shiori
```

临时覆盖其中的一个环境变量，执行 `MY_AGE=27 kubectl kustomize ./`，可以得到如下 ConfigMap:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: example-configmap-1-9276cd85k9
data:
  # Key: env name, Value: env value
  MY_AGE: "27"
  MY_NAME: shiori
```

### 3.3 字面值作为源数据

`configMapGenerator` 的 `literals` 用于直接给出 ConfigMap 的内容。

```yaml
# kustomization.yaml
configMapGenerator:
- name: example-configmap-2
  literals:
  - FOO=Bar
```

生成的 ConfigMap 如下：

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: example-configmap-2-42cfbf598f
data:
  FOO: Bar
```

### 3.4 使用生成的 ConfigMap

当使用 Kustomize 生成其他 Resource 时，可以直接引用 `configMapGenerator` 生成的 ConfigMap。Kustomize 会自动替换为实际生成的 ConfigMap 的名字。

举个例子，我们期望生成 Deployment 的定义，并且直接使用生成的 ConfigMap：

```yaml
# kustomization.yaml
resources:
- deployment.yaml
configMapGenerator:
- name: example-configmap-2
  literals:
  - FOO=Bar

# deployment.yaml
apiVersion: apps/v1
kind: Deployment
# ...
spec:
  template:
    spec:
      # ...
      volumes:
      - name: config
        configMap:
          name: example-configmap-2 # use `configMapGenerator.[].name`
```

执行 `kubectl kustomize ./` 生成，可以看到 Deployment 定义中使用的是生成的 ConfigMap 名字：

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: example-configmap-2-42cfbf598f
data:
  FOO: Bar
---
apiVersion: apps/v1
kind: Deployment
spec:
  template:
    spec:
      # ...
      volumes:
      - configMap:
          name: example-configmap-2-42cfbf598f # name is replaced
        name: config
```

### 3.5 generatorOptions

`generatorOptions` 提供生成的一些选项，包括：

* 不添加后缀。上面看到，生成的 ConfigMap 默认为会包含内容哈希值的作为后缀，通过 `disableNameSuffixHash` 可以关闭该功能。

* cross-cutting 字段。为所有生成的 ConfigMap 添加 Labels 与 Annotations。

```yaml
configMapGenerator:
- name: example-configmap-3
  literals:
  - FOO=Bar
generatorOptions:
  disableNameSuffixHash: true
  labels:
    type: generated
  annotations:
    note: generated
```

## 4 Patch Resource

Kustomize 支持使用 Patch 方式来定义化的修改 Resource，这样可以支持修改 Resource 的任意字段。

### 4.1 自定义 Patch

自定义 Patch 指的是能够支持 Resource 的任意字段。包括两种方式：

* [**Strategic Merge Patch**](../../cloud_native/kubernetes/k8s_learning/update-resource-mechanism/#23-strategic-merge-patch)
  
  使用 `patchesStrategicMerge` 字段指定文件列表，Kustomize 会使用文件内容来执行 Strategic Merge Patch。

  例如，修改 Deployment 定义中的 Replica：

  ```yaml
  # kustomization.yaml
  resources:
  - deployment.yaml
  patchesStrategicMerge:
  - increase_replicas.yaml
  
  # increase_replicas.yaml
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    name: my-nginx
  spec:
    replicas: 3
  ```

* [**JSON Patch**](../../cloud_native/kubernetes/k8s_learning/update-resource-mechanism/#21-json-patch)
  
  使用 `patchesJson6902` 字段指定文件列表与 Patch 对象，Kustomize 会使用文件内容来执行 JSON Patch。

  ```yaml
  # kustomization.yaml
  resources:
  - deployment.yaml
  patchesJson6902:
  - target:
      group: apps
      version: v1
      kind: Deployment
      name: my-nginx
    path: patch.yaml

  # patch.yaml
  - op: replace
    path: /spec/replicas
    value: 3
  ```

无论是哪种 Patch 方式，都需要明确指定修改的 Resource 的名字，以对应生成的 Resource。

### 4.2 特定字段 Patch

#### 4.2.1 Patch Image

`images` 支持为各个 Resource 修改 Image。

例如下面例子，将所有 `image:nginx` 的镜像替换为新的镜像，并使用新的 Tag。

```yaml
# kustomization.yaml
resources:
- deployment.yaml
images:
- name: nginx
  newName: my.image.registry/nginx
  newTag: 1.4.0
```

#### 4.2.2 公共 Labels 与 Annotations

`commonLabels` 为 所有 Resource 添加相同的 Labels，`commonAnnotations` 为 所有 Resource 添加相同的 Annotations。

```yaml
# kustomization.yaml
commonLabels:
  app: bingo
commonAnnotations:
  oncallPager: 800-555-1212
```

#### 4.2.3 公共 Namespace

`namespace` 为所有 Resource 添加相同的 Namespace。

```yaml
# kustomization.yaml
namespace: my-namespace
```

#### 4.2.4 Patch Name

`namePrefix` 和 `nameSuffix` 为所有 Resource 命名添加相同的前缀或者后缀

```yaml
# kustomization.yaml
nameSuffix: "-001"
```

#### 4.2.5 Patch Replica

`replicas` 修改特定的资源的副本数量。通过命名来指定需要修改的资源。

例如，以下配置修改 Deployment 的 Replica：

```yaml
# kustomization.yaml
replicas:
- name: deployment-name
  count: 5
```

## 5 变量传递

Kustomize 提供了一种多个 Resource 定义之间传递变量的方式。

使用 `vars` 字段定义好变量名称，以及获取值的方式：

```yaml
# kustomization.yaml
namePrefix: dev-
nameSuffix: "-001"
resources:
- deployment.yaml
- service.yaml
vars:
- name: MY_SERVICE_NAME
  objref:
    # 表明 MY_SERVICE_NAME 为 my-nginx Service 的 Resource Template 
    kind: Service
    name: my-nginx
    apiVersion: v1
```

在 Resource Template 中，使用 `$(var)` 方式来使用变量：

```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
# ...
spec:
  template:
    spec:
      containers:
      - name: my-nginx
        image: nginx
        command: ["start", "--host", "$(MY_SERVICE_NAME)"]

# service.yaml
apiVersion: v1
kind: Service
metadata:
  name: my-nginx
  labels:
    run: my-nginx
spec:
  ports:
  - port: 80
    protocol: TCP
  selector:
    run: my-nginx
```

这样，即使生成的 Service 命名会添加前缀和后缀，Kustomize 能够替换为生成后的 Service。

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: dev-my-nginx-001
spec:
  template:
    spec:
      containers:
      - command:
        - start
        - --host
        - dev-my-nginx-001 # var is replaced
```

## 6 Base 与 Overlay

Kustomize 支持层级的 `kustomization.yaml` 管理：Base 包含基础的 Resource Template 与 `kustomization.yaml` 文件，Overlay 引用 Base 并添加或覆盖字段。

Overlay 中使用 `resources` 字段来引用 Base 目录，并增加修改。

例如，我们使用如下的层级：

```bash
$ tree
.
├── base
│   ├── deployment.yaml
│   ├── kustomization.yaml
│   └── service.yaml
├── dev
│   └── kustomization.yaml
└── prod
    └── kustomization.yaml
```

* base 目录 - 包含基础 Resource Template 与 `kustomization.yaml` 文件
* dev 与 prod 目录 - 根据不同环境配置的 `kustomization.yaml` 文件

在 dev 与 prod 目录的中 `kustomization.yaml` 文件中，使用 `base` 字段引用基础目录，并使用 `namePrefix` 字段添加环境相关的前缀：

```yaml
# base/kustomization.yaml
resources:
- deployment.yaml
- service.yaml

# dev/kustomization.yaml
resources:
- ../base
namePrefix: dev-

# prod/kustomization.yaml:
resources:
- ../base
namePrefix: prod-
```

这样，执行 `kubectl kustomize <dev/prod>` 会得到不同的结果。

## 7 Patch CRD

为了支持 Patch CRD，需要额外的 Kustomize Config 来定义需要 Patch 的字段。

例如，有着 `MyKind` 与 `Bee` 两个 CR，以下的 Kustomize Config 文件为例：

```yaml
# kustomizeconfig/mykind.yaml
commonLabels:
- path: spec/selectors
  create: true
  kind: MyKind

nameReference:
- kind: Bee
  fieldSpecs:
  - path: spec/beeRef/name    # MyKind 的 `spec.beeRef.name` 字段值为生成的 Bee 资源的 name
    kind: MyKind
- kind: Secret
  fieldSpecs:
  - path: spec/secretRef/name # MyKind 的 `spec.secretRef.name` 字段值为生成的 Secret 资源的 name
    kind: MyKind

varReference:
- path: spec/containers/command # MyKind 的 `spec.containers.command` 字段可以使用 Vars
  kind: MyKind
- path: spec/beeRef/action      # MyKind 的 `spec.beeRef.action` 字段可以使用 Vars
  kind: MyKind
```

* `commonLabels` 定义了配置公共 Labels 也会影响 `MyKind` 的 `spec.selectors` 字段。

* `nameReference` 定义使用 `MyKind` 的子字段会使用其他 Resource 的命名。

* `varReference` 定义了 `MyKind` 的子字段需要进行 Vars 的解析。

对应的 Kustomize 文件与 Resource 文件如下：

```yaml
# kustomization.yaml
resources:
- resources.yaml
namePrefix: test-
commonLabels:       # 会影响 `MyKind` 的 `spec.selectors` 字段
  foo: bar
vars:              # vars 对应于 Kustomize Config 文件的 `varReference` 配置
- name: BEE_ACTION
  objref:
    kind: Bee
    name: bee
    apiVersion: v1beta1
  fieldref:
    fieldpath: spec.action
configurations:
- kustomizeconfig/mykind.yaml

# resources.yaml
apiVersion: v1
kind: Secret
metadata:
  name: crdsecret
data:
  PATH: YmJiYmJiYmIK
---
apiVersion: v1beta1
kind: Bee
metadata:
  name: bee
spec:
  action: fly
---
apiVersion: jingfang.example.com/v1beta1
kind: MyKind
metadata:
  name: mykind
spec:
  secretRef:
    name: crdsecret # 映射到 crdsecret Secret 生成后的 name
  beeRef:
    name: bee       # 映射到 bee Secret 生成后的 name
    action: $(BEE_ACTION)
  containers:
  - command:
    - "echo"
    - "$(BEE_ACTION)"
    image: myapp
```

## 8 kubectl

除了本身的 `kubectl kustomize` 来使用 Kustomize 之外，kubectl 基本的命令都已经支持直接使用 Kustomize。

与 `kubectl -f <file>` 参数类似，大部分子命令都支持 `kubectl -k <dir>`。与使用 `-f` 参数类似，执行之前先用 Kustomize 生成输出，然后使用实处。

例如，使用 `kubectl apply -k` 部署 Kustomize 输出结果的 Resource。

## 参考

* 官方文档：[**使用 Kustomize**](https://kubernetes.io/zh/docs/tasks/manage-kubernetes-objects/kustomization/#generating-resources)

