# Thinking in Operator


> 下面是作为 Operator 初学者自己总结的一些经验，纯属个人随意总结，没有任何正确性可言。

## 如何使用 Status 传递过程式操作状态

过程式操作指的是 Upgrade Scale 这样的操作，也就是触发后需要 Operator 进行持续性的操作。

过程式操作对于 Operator 就是：根据当前的状态，进行操作，以逐步变为期望状态。我们把更新的 Spec 中配置称为 Desired Config，当前已经存在的资源配置称为 Current Config。

因此在操作过程中，就涉及了 Current Config -> Desired Config 的逐步转移，所以往往会有以下字段：

* `Spec.DesiredConfig`
* `Status.CurrentConfig`
* `Status.DesiredConfig`
* `Status.CurrentConfigCount`
* `Status.DesiredConfigCount`
* **(optional)** `Spec.Generation`
* **(optional)** `Status.ObservedGeneration`

通过这些字段，我们可以判断出操作所处的阶段：

* 判断操作是否完全完成：Spec.DesiredConfig == Status.CurrentConfig

* 判断操作是否进行中：Status.CurrentConfig == Status.DesiredConfig

* 判断操作的进度：Status.CurrentConfigCount || Status.DesiredConfigCount

* 如果需要区别出 DesiredConfig A -> DesiredConfig B -> DesiredConfig A 这种回滚的情况，需要一个完全增量的 Generation 来进行判断。

我们会以 StatefulSet 为例，来看看 StatefulSet 对于 Upgrade 的状态是如何暴露的。

{{< admonition note Note>}}
为了理解 Upgrade 的状态字段含义，你需要先理解 StatefulSet 是如何进行 Upgrade 的，见 [**Kubernetes - Update 与 Rollback**](../../k8s_learning/rolling-upgrade)
{{< /admonition >}}

```yaml
status:
  # ...
  currentReplicas: 3
  updatedReplicas: 3
  currentRevision: main-cluster-pd-6584676774
  updateRevision: main-cluster-pd-6584676774
  observedGeneration: 4
```

显而易见地，这些字段对应着上面所说的字段。一些区别在于，Sts 中使用生成的 Revision，而没有存在 `Spec.DesiredConfig` 的配置。对应地，Sts 中使用 Generation 来判断操作是否已经完全完成：

* observedGeneration == generation && currentRevision == && updateRevision
  
潜在的含义是：当 Sts Spec 更新后，generation 就自动增长了，如果 observedGeneration 与 generation 相等，就表明 Sts Controller 处理过了最新的 Spec，那么就可以用 Revision 来判断操作是进行中还是完成。

因此，如果 `Spec.DesiredConfig` 是一个内部的概念，那么需要使用 Generation。

## 如何使用标志位互斥

在复杂的 Operator 编写时，往往有一些操作是需要互斥的，例如 Initialize 过程中不允许执行 Scale 或者 Upgrade 操作，或者 Scale 与 Upgrade 也是不能同时进行的。

互斥操作实现很简单，先获取 “锁”，在 Kubernetes 资源中，需要通过 Status 中的某个字段来实现锁。

代码示例：

```go
flagSeted := ReadFlagFromStatus(cluster)
shouldOperate := ShouldOpearte(cluster)

if !shouldOperate {
    if flagSeted {
        // unset flag and end operation
        return UnsetFlag(cluster)
    }

    // done
    return nil
}

if !flagSeted {
    // set flag and begin operatiron
    return SetFlag(cluster)
}

// operate
return Operate(cluster)
```

可以看到，上面通过 Flag 作为操作前的必要条件，并且分离的条件判断与操作。

## Read 与 Write 分离

Operator 编写中，往往需要先获取一些资源，然后根据资源进行操作。Read 与 Write 分离指的就是："读取资源" 与 "进行操作" 的分离。

```go
// observe all resource
resources := Observe(cluster)

// optional progress

// operate
OperateA(resources)

// operate
OperateB(resources)
```

Read 与 Write 分离的最大好处是：Resource 只需要获取一次，后面各个 Operate 都可以复用这一个 Resource，并且所有操作是看到的都是同一个份 Resource，不会有 Resource 差异导致不同行为的问题。

不仅仅是 "读取资源" 与 "进行操作" 的分离，在 Operate 时，可以实现 "判断" 与 "操作" 分离。

```go
func Operate(resources []Resources) {
    shouldOperate := ShouldOperate(resources)
    couldOperate := CouldOperate(resources)

    if !shouldOperate {
        return
    }

    if !couldOperate {
        return
    }

    // operate
}
```

"判断" 与 "操作" 分离使得整体的 Operate 分为了清晰的步骤，更容易阅读与扩展。如果 Operate 之间互斥，可以使用 [**如何使用标志位互斥**](#如何使用标志位互斥) 中描述的代码。

## 为什么需要 `last-applied-configuration`

在 [**kubectl apply**](../../k8s_learning/update-resource-mechanism/#31-kubectl-apply) 实现中，默认的 Client Side Apply 使用了 `last-applied-configuration` Annotation 记录上一次的 Apply Config，由此来进行 Diff，而不是直接使用 Object 来 Diff。

一开始感觉直接使用当前 Object 进行 Diff 就可以得出 “期望” 与 “实际” 状态之间的差异，为了需要一个看起来多余的 Annotation 来记录呢？

往往一个 Object 的不同字段可能会由不同的 Controller 更新，也会被手动更新一些字段。这时候你的部分字段不是由 Apply Config 控制的，如果使用 Object 直接比较会导致一些字段的丢失。

我们通过一个例子说明，如果直接使用 Object 来进行 Diff 会发生什么：

1. 第一次使用 Apply Config 更新下面的 Object。这时候 `fieldA` 和 `fieldB` 都是由 Apply 来更新的。

   ```yaml
   kind: Object
   spec:
     fieldA: aaa
     fieldB: bbb
   ```

2. 如果该 Object 另外字段也会由其他 Controller 管理，那么可能其他 Controller 更新了 `fieldC` 字段。

   ```yaml
   kind: Object
   spec:
     fieldA: aaa
     fieldB: bbb
     fieldC: ccc
   ```

3. 但是，`fieldC` 字段的更新并不会体现在你的 Apply Config 中，这时候你再次 Apply 就会导致 `fieldC` 字段的丢失。

   ```yaml
   kind: Object
   spec:
     fieldA: aaa
     fieldB: bbb
     # fieldC: ccc # deleted by apply
   ```

问题最关键地点在于，Apply 的语义是 Update（而不是 Patch），所以需要知道 Apply 管理的是哪些字段，从而不影响其他的字段。因此需要额外的 Annotation `last-applied-configuration` 来记录管理的哪些字段。

同样，这也是为啥后续 Server Side Apply 能够不需要额外 Annotation 的原因。因为 Server Side Apply 提供了字段所有权的机制，知道哪些字段是哪些 Controller 管理的，所以不需要 `last-applied-configuration` 记录了。


