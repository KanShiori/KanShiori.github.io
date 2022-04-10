# 使用 Pulumi


[**Pulumi**](https://www.pulumi.com/) 是一个支持 IaC 的平台。与其他 IaC 工具最大的不同时，Pulumi 允许使用熟悉的编程语言来编写部署程序。开发者编写部署程序，Pulumi 解析后创建对应允许平台的资源。

## 1 基本概念

Pulumi 使用的主要结构如下：

{{< image src="img1.png" >}}

* Resource - 物理云资源的逻辑抽象，在 Program 会编写期望的 Resource；

* Program - 用户使用通用语言（NodeJS、Golang 等）编写的部署程序，其通过声明式的方式描述了期望的 Resource 架构；

* Project - Program 所属一个 Project，包含 Program 以及元信息的目录就是 Project；

* Stack - Program 会部署到多个 Stack，对应不同的运行环境。各个 Stack 之间 Resource 是独立隔离的；

大致的使用流程为：

1. 用户编写 Program，描述期望的 Resource。
2. 通过 Pulumi 将 Program 部署到不同的 Stack，例如 dev 环境和 prod 环境。
3. Pulumi 根据编写 Program 调用各个云的 API 来创建物理资源。

{{< admonition note 异步构建>}}
可以看到，代码中的逻辑与实际 Pulumi 运行是异步的，Pulumi 会识别出 Resource 之间的依赖关系，并最大化的并行构建资源。
{{< /admonition >}}

## 2 Project

执行 `pulumi new` 构建一个新的 Project，会一步步引导你来创建 Project。

任何包含 `Pulumi.yaml` 文件的目录都被认为是一个 Project。 `Pulumi.yaml` 中包含了 Project 的元信息。

```yaml
name: webserver
runtime: nodejs
description: Basic example of an AWS web server accessible over HTTP.
```

{{< admonition note Note>}}
不同的语言之间文件内容不同，例如 JavaScript 会包含 package.json 文件来管理依赖。
{{< /admonition >}}

在 Pulumi Program 中使用的所有本地路径都是当前目录的相对目录。

```ts
const myTask = new cloud.Task("myTask", {
    build: "./app", // subfolder of working directory
    ...
});
```

## 3 Stack

Stack 代表一个独立的环境，Pulumi 会跟踪每个 Stack 的当前状态，在部署 Program 到一个 Stack 时，对比期望状态（Program 编写的）和当前状态（Stack 状态）来进行声明式的部署。

{{< image src="img2.png" >}}

{{< admonition note Note>}}
可以理解，Program 是静态的，Stack 是动态的，类似于程序与进程之间的关系。
{{< /admonition >}}

不同 Stack 往往代表着不同的部署环境，Stack 之间通过 Stack 配置文件来描述环境的差异。当部署 Program 到一个 Stack 时，程序就会使用该 Stack 的配置文件来构建。Stack 配置文件格式为：`Pulumi.<stackname>.yaml`。

大多数的 `pulumi` 命令会默认使用当前的指定的 Stack，称为 Active Stack。可以使用 `pulumi stack select` 命令切换 Active Stack。

### 3.1 Stack 管理

执行 `pulumi init <stackname>` 来创建一个 Stack。如果要指定创建某个 Org 下的 Stack 需要使用 `<orgname>/<stackname>` 或者 Stack 完全名称 `<orgname>/<projectname>/<stackname>`。

```bash
$ pulumi stack init prod
Created stack 'prod'
```

执行 `pulumi up` 将 Program 部署到 Active Stack，之后 Pulumi 会使用当前 Stack 的配置文件来进行计算，并展示部署计划。

```bash
$ pulumi up
Previewing update (dev)

View Live: https://xxx

     Type                              Name            Plan
 +   pulumi:pulumi:Stack               quickstart-dev  create
 +   ├─ kubernetes:apps/v1:Deployment  nginx           create
 +   └─ kubernetes:core/v1:Service     nginx           create

Resources:
    + 3 to create
...
```

使用 `pulumi stack ls` 查看当前 Project 的 Stack，并可以使用 `pulumi stack select` 来切换 Active Stack。

```bash
$ pulumi stack ls
NAME   LAST UPDATE    RESOURCE COUNT  URL
dev    8 minutes ago  4               https://app.pulumi.com/KanShiori/quickstart/dev
prod*  n/a            n/a             https://app.pulumi.com/KanShiori/quickstart/prod

$ pulumi stack select dev
```

删除 Stack 分为两步：先需要使用 `pulumi destroy` 销毁 Stack 中的所有资源，才能使用 `pulumi stack rm` 删除 Stack。删除 Stack 意味着对应的配置文件以及所有的元信息都会被删除。

```bash
$ pulumi destroy
Previewing destroy (dev)

View Live: https://app.pulumi.com/KanShiori/quickstart/dev/previews/a4d21b5f-a0b6-4631-b4dd-2c1ac49b9e14

     Type                              Name            Plan
 -   pulumi:pulumi:Stack               quickstart-dev  delete                                                                                                              -   ├─ kubernetes:core/v1:Service     nginx           delete
 -   └─ kubernetes:apps/v1:Deployment  nginx           delete

Outputs:
  - ip: "10.96.122.118"

Resources:
    - 3 to delete

Do you want to perform this destroy? yes
...

$ pulumi stack rm dev
This will permanently remove the 'dev' stack!
Please confirm that this is what you'd like to do by typing ("dev"): dev
Stack 'dev' has been removed!
```

{{< admonition note 强制删除>}}
使用 `--force` 参数可以强制删除一个仍有资源的 Stack，但是不推荐这么做。
{{< /admonition >}}

### 3.2 Stack Tag

Stack 可以关联一组 tag（键值对）作为额外信息，用以 Pulumi Service 使用。例如可以使用 tag 来对 Stack 进行分组。

默认 Stack 会自动被加上一些 tag，例如 Key 为 pulumi:project pulumi:runtime gitHub:owner vcs:owner 等。

```bash
$ pulumi stack tag ls
Please choose a stack: prod
NAME                VALUE
gitHub:owner        KanShiori
gitHub:repo         Example
pulumi:description  A minimal Kubernetes TypeScript Pulumi program
pulumi:project      quickstart
pulumi:runtime      nodejs
vcs:kind            github.com
vcs:owner           KanShiori
vcs:repo            Example
```

当然，你也可以使用 `pulumi stack tag set <name> <value>` 来添加自定义 tag。约定俗成地，pulumi: gitHub: vcs: 前缀是内置使用的 Key，避免使用。

```bash
$ pulumi stack tag set myname shiori
```

### 3.3 Stack Output

Program 中可以定义一些 Output 来导出 Resource 的信息，Stack 元信息中会包含定义的 Output 信息。用户可以用来导出一些重要信息，或者导出信息来让 Program 使用。

使用 export 语句定义 Output：

```ts
export let url = resource.url;
```

Stack Output 本质上是一个 JSON：

```json
$ pulumi stack output --json
{
  "x": "hello",
  "o": {
      "num": 42
  }
}
```

{{< admonition note "Secret 信息">}}
如果 Stack Output 包含 Secret 信息，那么默认不会展示 Secret 信息与对应的值。你可以通过 `--show-secrets` 来展示。
{{< /admonition >}}

### 3.4 Stack Reference

使用 Stack Reference 可以在 Stack 中访问另一个 Stack 的 Output。代码中通过创建一个 `StackReference` 对象，并引用另一个 Stack 的**完全命名**来使用（支持跨 Org 或者跨 Project 使用）。

```ts
import * as pulumi from "@pulumi/pulumi";
const other = new pulumi.StackReference("acmecorp/infra/other");
const otherOutput = other.getOutput("x");
```

举一个适用场景：你的一些 Kubernetes 使用其他 Stack 进行部署，你在某个 Stack 想要在该 Kubernetes 中部署资源，那么你可以使用 `StackReference` 来引用目标 Kubernetes 的 kubeconfig。

```ts
import * as k8s from "@pulumi/kubernetes";
import * as pulumi from "@pulumi/pulumi";
const env = pulumi.getStack();
const infra = new pulumi.StackReference(`mycompany/infra/${env}`);  // 引用另一个 Stack
const provider = new k8s.Provider("k8s", { kubeconfig: infra.getOutput("kubeConfig") }); // 从 Output 中得到 kubeconfig
const service = new k8s.core.v1.Service(..., { provider: provider });  // 设置 Provider 以部署资源到目标 Kubernetes 中
```

### 3.5 Stack 的导入导出

Stack 支持使用原始数据的导入与导出，这适用于某些情况你必须手动修改 Stack 相关信息。

```bash
$ pulumi stack export --file stack.json

$ pulumi stack import --file stack.json
```

## 4 Backend

Pulumi 本质上是对比期望状态（Program 中定义）与实际状态（Stack 资源信息），来进行声明式的构建资源。Stack 元信息也被称为 Stack State，由 Backend 负责存储。

Pulumi 支持多种 Backend，大致上可以分为两类：

* Service
  
  Pulumi 原生的 Backend 服务，默认使用域名 app.pulumi.com 来作为 Backend。

  Pulumi Service 也支持线下自行部署并使用。
  
* Self-Managed
  
  因为 Stack State 本质上是一个 JSON 文件，因此支持大部分的对象存储来进行存储，包括 AWS S3、Azure Blob Storage 等兼容 S3 的对象存储，以及本地文件系统。

  Self-Managed Backend 的实现都是开源的，可以在任何环境免费使用。不过， Self-Managed Backend 仅仅负责 State 文件的存储，意味着需要用户自行管理 State 的访问安全、信息加密、History 等功能。

### 4.1 Login 与 Logout

执行 `pulumi login` 来登陆到一个 Backend，不提供 URL 情况下默认使用 Pulumi Service。

```bash
$ pulumi login <backend-url>
```

Login 成功后，Pulumi CLI 会将凭证信息保存在 ~/.pulumi/credentials.json 文件中。

{{< admonition note Note>}}
或者，你刻意设置 `PULUMI_BACKEND_URL` 环境变量来默认指定 Backend URL。
{{< /admonition >}}

对于不同的 Self-Managed Backend，体现在 URL 路径中。例如：`s3://<bucket-path>` `azblob://<container-path>`。URL 中指定了存储的根目录，Pulumi 会将文件存储在根目录的 .pulumi 目录下。

对于本地文件系统，可以指定 URL 为 `file://<fs-path>` 或者使用 `pulumi login --local`。区别在于，`--local` 固定保存文件在 ~/.pulumi 目录下，而指定 URL 可以指定本地文件的目录。

执行 `pulumi logout` 来登出 Backend，这将会删除  ~/.pulumi/credentials.json 文化中的所有凭证信息。

### 4.2 Pulumi Service 架构

Pulumi Service 提供两个域名：app.pulumi.com 是针对用户的 Web 控制台，api.pulumi.com 是用于 CLI 访问的 REST 服务。

Pulumi Service 整体的架构如下：

{{< image src="img3.png" >}}

可以看到，Pulumi Service 与 Cloud 之间没有任何关联，调用 Cloud API 完全是由 CLI 中发起的。也就是说，Pulumi Service 不需要任何 Cloud 的权限。

当线下部署 Pulumi Service 时架构与上图类似，区别在于使用的不是公共域名而是用户设置的私有域名。

### 4.3 State 的细节

参考官方文档：[**Advanced State**](https://www.pulumi.com/docs/intro/concepts/state/#advanced-state)。

## 5 Resource

Resource 代表将要部署的云资源，例如 Instance、Kubernetes 集群等。

Resource 在代码中由 `Resource` 类表示，有两个子类：

* CustomResource - 对应与云管理的原生的资源，由对应的云的 Resource Provider 管理，例如 AWS、Azure 等；
  
* ComponentResource - 由其他 Resource 组合成的逻辑资源；

期望部署的资源由创建对应的 `Resource` 对象来描述，大部分都类似如下格式：

```ts
let res = new Resource(name, args, options);
```

* name - Resource 的 Logical Name，每个 Stack 的每个 Resource 中唯一。
  
  Logical Name 用于 Pulumi 中引用该 Resource，并且对应的实际资源的 Physical Name 会基于 Logical Name 来生成。

* args - 为 Resource 的相关属性的配置。
  
  每种 Resource 有着不同的配置参数，具体参考 [**Registry**](https://www.pulumi.com/registry/) 。

* options - 可选参数，允许控制 Resource 的某些方面。
  
  例如，可以通过 options 指定 Resource 之间的依赖关系，或者指定 Resource Provider 等等。

### 5.1 Resource Name

每个 Resource 有着 Logical Name 与 Physical Name。

* Logical Name - 用于 Pulumi 中标识该资源，代码中创建时需要指定；
* Physical Name - Resource Provider 部署实际的云资源使用的名字；

```ts
let role = new aws.iam.Role("my-role");
```

#### 5.1.1 Physical Name 生成规则

默认下，Provider 会使用 Logical Name 作为前缀，后增加随机字符串来生成 Physical Name。随机字符串的意义在于：

* 随机字符串使得 Stack 之间 Resource 的命名不同，防止出现命名冲突；
* 允许 Pulumi 进行滚动升级：先创建新的 Resource，后删除旧的 Resource；

当然，也可以不自动生成，大多数 Resource 有着 `name` 参数，用以指定 Physical Name。

```ts
let role = new aws.iam.Role("my-role", {
    name: "my-role-001",
});
```

{{< admonition note Note>}}
一些资源可能不是使用 `name` 参数（例如 s3 bucket），也有一些资源无法指定名称（例如 kms key）。
{{< /admonition >}}

指定 Physical Name 后，需要用户自行防止 Stack 之间的命名冲突，并且对于 Resource 升级时，需要指定参数 `deleteBeforeReplace: true`。该参数会在升级资源时，先删除旧资源，后创建新资源。

```ts
let role = new aws.iam.Role("my-role", {
    name: `my-role-${pulumi.getProject()}-${pulumi.getStack()}`,
}, { deleteBeforeReplace: true });
```

#### 5.1.2 Resource URN

每个 Resource 有着全局唯一的名字，称为 Uniform Resource Name（URN）。不过大多数情况下，用户不会直接看到 URN。

URN 基于 Project、Stack、Resource、Resource 类型和父资源类型来创建：

```urn
urn:pulumi:production::acmecorp-website::custom:resources:Resource$aws:s3/bucket:Bucket::my-bucket
           ^^^^^^^^^^  ^^^^^^^^^^^^^^^^  ^^^^^^^^^^^^^^^^^^^^^^^^^ ^^^^^^^^^^^^^^^^^^^^  ^^^^^^^^^
           <stack-name> <project-name>   <parent-type>             <resource-type>       <resource-name>
```

不同 URN 的 Resource 之间是被认为没有任何关联的。例如当你改变了某个资源导致 URN 发生变化时，Pulumi 的行为是删除旧 URN 的资源，创建新 URN 的资源。这意味着，Pulumi 调用的是 create 和 delete 操作，而不是 update 和 replace 操作。

{{< image src="img4.png" >}}

### 5.2 Resource Options

所有的 Resource 创建时可以传入一些选项来控制一些行为，例如：

* additionalSecretOutputs - 指定一些属性作为 Secret 加密；
* aliases - 指定 Resource 的别名，以在重命名时保留物理资源；
* deleteBeforeReplace - 先删除旧资源再创建新资源，以覆盖默认行为；
* dependsOn - 指定 Resource 之间的依赖关系；
* import - 直接复用当前已有的云资源；
* parent - 在 Resource 之间构建父子关系；
* replaceOnChanges - 某些属性的更改应该直接使用 replace；
* provider - 指定管理资源使用的 Provider；

更多信息参考官方文档：[**Resource Options**](https://www.pulumi.com/docs/intro/concepts/resources/options/)。

### 5.3 ComponentResource

ComponentResource 是将一组 Resource 进行组合。Component 在构造函数中实例化使用到 Resource，并将其作为 Child Resource。

{{< admonition note Note>}}
实际上，Stack 本身就是一个 ComponentResource，里面包含了所有 Program 中构建的 Resource。
{{< /admonition >}}

使用时，我们需要创建一个继承于 `ComponentResource` 的子类。在该类的构造函数中，需要先调用父类的构造函数，以注册 ComponentResource 类型到 Pulumi 中。

在构造函数中，可以构建使用的 Child Resource，这与正常编写 Program 时一致。

最后，我们调用了 `registerOutputs()` 可以将构造函数中的 Output 注册到 Pulumi 中，这也意味着构造函数的结束。因此，无论是否需要注册 Output，都应该最后调用该函数。

```ts
const aws = require("@pulumi/aws");
const pulumi = require("@pulumi/pulumi");

// Define a component for serving a static website on S3
class S3Folder extends pulumi.ComponentResource {

    constructor(bucketName, path, opts) {
        // Register this component with name examples:S3Folder
        super("examples:S3Folder", bucketName, {}, opts);
        console.log(`Path where files would be uploaded: ${path}`);

        // Create a bucket and expose a website index document
        let siteBucket = new aws.s3.Bucket(bucketName, {},
            { parent: this } ); // specify resource parent

        // Create a property for the bucket name that was created
        this.bucketName = siteBucket.bucket,

        // Register that we are done constructing the component
        this.registerOutputs({bucketDnsName: bucket.bucketDomainName});
    }
}

module.exports.S3Folder = S3Folder;
```

创建 ComponentResource 对象时，支持一个特殊的参数 `providers`。因为 ComponentResource 可以包含多个 Resource，因此会使用多种 Provider 来构建。`providers` 参数就是用于指定 ComponentResource 使用的 Provider。

例如，下面示例中指定了 ComponentResource 使用的 AWS 与 Kubernetes 资源对应的 Provider。

```ts
let component = new MyComponent("...", {
    providers: {
        aws: useast1,
        kubernetes: myk8s,
    },
});
```

ComponentResource 中所有的 Child Resource 都默认会继承 Provider，除非创建时指定。

### 5.4 Resource Provider

Resource Provider 负责与云服务通信，以 create、read、update 和 delete 代码中定义的 Resource。不同的 Resource 会使用不同的 Provider，例如：AWS Resource Provider，Kubernetes Resource Provider 等。

默认下，所有的 Provider 都使用 Stack 配置文件作为配置。因此，你可以在 Stack 配置文件中设置 Provider 的相关配置。

例如，下面命令设置当前 Stack 默认将资源部署在 `us-west-2` Region。

```bash
$ pulumi config set aws:region us-west-2
```

{{< admonition note "关闭默认的 Provider">}}
默认 Provider 可以通过配置文件中设置关闭：

```bash
$ pulumi config set --path 'pulumi:disable-default-providers[0]' aws

$ pulumi config set --path 'pulumi:disable-default-providers[1]' kubernetes

$ pulumi config set --path 'pulumi:disable-default-providers[0]' '*'
```
{{< /admonition >}}

在 Program 中也可以通过创建 `Provider` 对象，然后通过创建 Resource 时指定 `provider`。这样，就可以使用定制的 Provider 来创建 Resource。

```ts
let pulumi = require("@pulumi/pulumi");
let aws = require("@pulumi/aws");

// Create an AWS provider for the us-east-1 region.
let useast1 = new aws.Provider("useast1", { region: "us-east-1" });

// Create an ACM certificate in us-east-1.
let cert = new aws.acm.Certificate("cert", {
    domainName: "foo.com",
    validationMethod: "EMAIL",
}, { provider: useast1 });

// Create an ALB listener in the default region that references the ACM certificate created above.
let listener = new aws.lb.Listener("listener", {
    loadBalancerArn: loadBalancerArn,
    port: 443,
    protocol: "HTTPS",
    sslPolicy: "ELBSecurityPolicy-2016-08",
    certificateArn: cert.arn,
    defaultAction: {
        targetGroupArn: targetGroupArn,
        type: "forward",
    },
});
```

如 [**ComponentResource**](#componentresource) 所说，通过 `providers` 参数来设置 ComponentResource 子资源使用的 Provider。

```ts
class MyResource extends pulumi.ComponentResource {
    constructor(name, opts) {
        let instance = new aws.ec2.Instance("instance", { ... }, { parent: this });
        let pod = new kubernetes.core.v1.Pod("pod", { ... }, { parent: this });
    }
}

let useast1 = new aws.Provider("useast1", { region: "us-east-1" });
let myk8s = new kubernetes.Provider("myk8s", { context: "test-ci" });
let myResource = new MyResource("myResource", { providers: { aws: useast1, kubernetes: myk8s } });
```

### 5.5 Dynamic Providers

Dynamic Providers 指的是用户自行编写的 Provider，类似于原生的 AWS Provider 与 Kubernetes Provider 等，可以通过 Dynamic Provider 来管理用户自定义的资源。

Dynamic Providers 实现基于一些框架，具体细节见官方文档：[**Dynamic Providers**](https://www.pulumi.com/docs/intro/concepts/resources/dynamic-providers)。

### 5.6 Getter 函数

所有的 Resource 资源都可以使用 `get` 函数来获取某个资源的信息。不同于 `import` 函数，`get` 函数取得的结果是同步的，Pulumi 永远不会更新或删除 `get` 函数读取的资源信息。

`get` 函数接受两个参数：Logical Name 和 对应资源的 Physical ID。

```ts
import * as aws from "@pulumi/aws";

let group = aws.ec2.SecurityGroup.get("group", "sg-0dfd33cdac25b1ec9");

let server = new aws.ec2.Instance("web-server", {
    ami: "ami-6869aa05",
    instanceType: "t2.micro",
    securityGroups: [ group.name ], // reference the security group resource above
});
```

## 6 Input 和 Output

Program 中定义的 Resource 是异步构建的，为了能够处理 Resource 之间的依赖关系，Pulumi 设计出了 Input 与 Output 类型来描述 Resource 之间的依赖关系。

每个 Resource 的构造函数都接收 `Input<T>` 类型，该类型可以是静态的值（例如 string、interger、boolean 类型的值等），或者异步计算的值（例如 Promise 或者 Task），或者从其他 Resource 属性读取的 Output。

所有 Resource 的属性以及本身都可以作为 Output。Output 是类型 `Output<T>` 的实例，其值会经过异步的计算。因为在实际的资源部署之前，Resource Output 都是未知的，所以 Pulumi 会在部署后异步填入 Output。

{{< admonition note Note>}}
不要纠结于 Input 与 Output 的概念，可以简单理解 Output 与 Input 都是描述依赖关系，而所有的值都是引用。
{{< /admonition >}}

根据 Input 与 Output，Pulumi 就可以计算出 Resource 之间引用的依赖关系，从而确保按顺序创建存在依赖关系的 Resource。

因为 Output 是异步的，因此其值不是立即使用的。如果你想从 Output 中派生出新的 Output（依赖关系会延续），有以下方法。

* Apply - 回调函数，接受 Output 处理后派生出新的 Output；
* Lifting - 直接读取 Resource 属性，该属性会被提升为一个 Output；
* Interpolation - 使用 Output 进行字符串拼接；

### 6.1 Apply

使用 `apply` 函数可以读取 Output 并派生出一个新的 Output。

```ts
let url = virtualmachine.dnsName.apply(dnsName => "https://" + dnsName);
```

上述示例中，`url` 也是一个 Output，并且依赖于 `dnsName` Output。也就是说，只有当 `dnsName` 的值可用后，才会计算出 `url` 的值。

如果需要从多个 Output 处理后派生出一个 Output，可以使用 `all` 函数。新的 Output 将会等待所有依赖的 Output 值可用后才计算出。

```ts
import * as pulumi from "@pulumi/pulumi";
// ...
let connectionString = pulumi.all([sqlServer.name, database.name])
    .apply(([server, db]) => `Server=tcp:${server}.database.windows.net;initial catalog=${db}...`);
```

### 6.2 Lifting

当你将某个 Resource 的属性传递给其他 Resource 用于初始化时，该属性会被 Lift 为一个 Output。也就是说，Resource 会自动依赖与另一个 Resource。

```ts
let certCertificate = new aws.acm.Certificate("cert", {
    domainName: "example.com",
    validationMethod: "DNS",
});
let certValidation = new aws.route53.Record("cert_validation", {
    records: [
        certCertificate.domainValidationOptions[0].resourceRecordValue
    ],
...
```

在 JavaScript 与 TypeScript 环境下，如果访问的 Resource 属性是计算后是未定义的，那么会认为值为 undefined，而不是报错。

### 6.3 Interpolation

即使 Output 的值为字符串，也不能直接用于字符串连接。直接使用 `apply` 函数拼接可能过滤繁琐，你可以使用 Pulumi 提供的 Interpolation 方法。

```ts
let hostname: Output<string>;
let port: Output<number>;

// 直接使用 apply 函数
let url = pulumi.all([hostname, port]).
    apply(([hostname, port]) => `http://${hostname}:${port}/`);

// 使用 concat 函数
const url1: Output<string> = pulumi.concat("http://", hostname, ":", port, "/");
// 使用 interpolate 函数
const url2: Output<string> = pulumi.interpolate `http://${hostname}:${port}/`;
```

### 6.4 Input 转换为 Output

Input 也可以转换为 Output，以便进一步处理它。使用 `output` 函数来转换 Input。

```ts
function split(input: pulumi.Input<string>): pulumi.Output<string[]> {
    let output = pulumi.output(input);
    return output.apply(v => v.split());
}
```

## 参考

* 官方文档：[**Architecture & Concepts**](https://www.pulumi.com/docs/intro/concepts/) 
* 官方文档：[**Registry**](https://www.pulumi.com/registry/)
