# CRD 实践


CRD 是 Kubernetes 可扩展性的第一个体现，因为 Kubernetes 提供的是一个编排的框架，因此不止可以对 Pod 进行编排，也支持通过 CRD 对你自定义的类型的编排。

## 1 CRD 构建
目的：通过 CRD 构建一个自定义资源。
1. 编写 CRD manifest，完成自定义资源的定义。
   {{< find_img "img1.png" >}}

   可以看到，CRD 中只有类型的简单定义，没有该 CR 的元属性的定义，因为这些需要通过代码中定义。
2. 通过 kubectl apply 创建 CR 对应 CRD 对象，让 Kubernetes "认识" 这个自定义资源。
   {{< find_img "img2.png" >}}

3. 调用 kubectl apply 创建自定义资源：
   {{< find_img "img3.png" >}}

   这里其实 Kubernetes 不知道该资源具体的代码类型，它只是知道有 CRNetwork 这个资源，并支持创建删除，将其保存下来了。
   {{< find_img "img4.png" >}}
4. 编写 CR 相关定义代码。其实这里编写代码是为生成 kubectl 的 Client，使其在编写 Controller 时候能够正确的解析你的 CR 对象。<br>
整个的目录结构如下：
   {{< find_img "img5.png" >}}

具体代码见仓库：[k8s_practice](https://github.com/KanShiori/k8s_practice)

通过 k8s 生成代码，最终生成的目录结构如下：
{{< find_img "img6.png" >}}


## 2 编写自定义控制器
因为 Kubernetes 中是基于声明式 API 的业务实现，所以需要控制器来“监控”对象变化，执行对应的操作。

编写自动以控制器主要有三个过程：编写 main 函数、编写自定义控制器定义，编写控制器业务逻辑。

整个实践代码见：[k8s_practice](https://github.com/KanShiori/k8s_practice)

1. Controller 代码编写。主要逻辑：处理 Informer 通知的 Event，执行 CR Sync 操作。<br>
Controller 主要包含三个部分：
    * **Informer**：包含从 APIServer 同步的 CR 对象的 Cache，并且处理 CR Event，调用 Event Handler。
    * **Event Handler**：Informer 的各个 Event 调用的事件回调，一般都是放入 workQueue，延后处理。
    * **Workers**：根据各个 Event 进行真正的业务处理，例如真实资源的创建、删除、更新等。
2. main 函数代码编写。主要逻辑：创建 Informer、Controller，执行 Controller 的启动。
3. 编译后运行 Controller。
   {{< find_img "img7.png" >}}
   
   第一同步后，所有的 CR 对象都可以被任务是“新添加的”，因此会一个个调用 HandleAdd 接口。上图可以看到，因为集群中已经有了一个 CR 对象，因此 Controller 会进行该对象的 Sync。
   {{< find_img "img8.png" >}}
4. 创建一个新的 CR 对象 example-crnetwork-2，观察 Controller。
   {{< find_img "img9.png" >}}

   可以看到 Controller 处理完成。
   {{< find_img "img10.png" >}}
5. 删除刚创建的 example-crnetwork-2，观察 Controller。
   {{< find_img "img11.png" >}}

   可以看到 Controller 正确的执行删除的逻辑。
   {{< find_img "img12.png" >}}

## 3 将自定义资源控制封装为 Pod
TODO


