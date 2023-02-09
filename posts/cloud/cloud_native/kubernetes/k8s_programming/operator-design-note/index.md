# K8s 编程 - Operator Design Notes


> 下面简单介绍了各个 Operator 项目的设计框架与思路，作为参考。
> 

## Cockroach Operator

[**Cockroach Operator**](https://github.com/cockroachdb/cockroach-operator) 基于 Controller Runtime 实现的，因此逻辑的入口是 `Reconcile()` 函数。

大体的逻辑如下：

```go
func (r *ClusterReconciler) Reconcile(ctx context.Context, req reconcile.Request) (reconcile.Result, error) {
	// 读取 Cluster
	cr := resource.ClusterPlaceholder(req.Name)
	if err := fetcher.Fetch(cr); err != nil {
		log.Error(err, "failed to retrieve CrdbCluster resource")
		return requeueIfError(client.IgnoreNotFound(err))
	}

	// ClusterStatus 为空，说明是新创建的 Cluster
	if cluster.Status().ClusterStatus == "" {
		// 转换 Cluster Status 为 Starting，并更新 Cluster
		cluster.SetClusterStatusOnFirstReconcile()
	    r.updateClusterStatus(ctx, log, &cluster)
		return requeueImmediately()
	}

	// 根据 Cluster 判断出应该执行的操作 Actor
	actorToExecute, err := r.Director.GetActorToExecute(ctx, &cluster, log)
	if err != nil {
		return requeueAfter(30*time.Second, nil)
	} else if actorToExecute == nil {
		log.Info("No actor to run; not requeueing")
		return noRequeue()
	}

	// 执行 Actor
	log.Info(fmt.Sprintf("Running action with name: %s", actorToExecute.GetActionType()))
	if err := actorToExecute.Act(ctx, &cluster, log); err != nil {
		// 记录错误信息到 Status.OperatorActions，并更新 Cluster Status 为 Failed
		// Save the error on the Status for each action
		cluster.SetActionFailed(actorToExecute.GetActionType(), err.Error())
	    r.updateClusterStatus(ctx, log, &cluster)
		return requeueIfError(err)
	}

	r.updateClusterStatus(ctx, log, &cluster)
	return noRequeue()
}
```

其中，最核心的组件就是 `Director` 与 `Actor` ：

* `Director` - 根据 Cluster 判断出应该执行哪一个 `Actor`；
* `Actor` - 表示对 Cluster 的操作；

所以，整体的 `Reconcile()` 逻辑就是：

1. `Director` 根据 Cluster 判断出应该执行哪一个 `Actor`。
2. 选出的 `Actor` 执行操作。
3. 根据操作结果更新 Cluster Status 中。

所有操作执行的结果都会体现在 Cluster Status 中，例如：

```yaml
status:
  clusterStatus: Failed
  conditions:
  - lastTransitionTime: "2023-02-06T13:34:06Z"
    status: "True"
    type: Initialized
  - lastTransitionTime: "2023-02-06T13:32:58Z"
    status: "True"
    type: CrdbVersionChecked
  - lastTransitionTime: "2023-02-06T13:32:59Z"
    status: "True"
    type: CertificateGenerated
  crdbcontainerimage: cockroachdb/cockroach:v22.1.3
  operatorActions:
  - lastTransitionTime: "2023-02-06T13:32:22Z"
    message: job changed
    status: Failed
    type: VersionCheckerAction
  - lastTransitionTime: "2023-02-06T13:34:01Z"
    message: pod is not running
    status: Failed
    type: Initialize
  version: v22.1.3
```

* `clusterStatus` 为 Cluster 整体的状态
* `conditions` 为记录 Cluster 状态信息
* `operatorActions` 记录各个 Action 执行信息

`Director` 按照顺序判断一个个 Actor 是否需要执行，如果需要则返回对应的 `Actor`：

```go
func (cd *clusterDirector) GetActorToExecute(ctx context.Context, cluster *resource.Cluster, log logr.Logger) (Actor, error) {
	// Restart Actor
	if cd.needsRestart(cluster) {
		return cd.actors[api.ClusterRestartAction], nil
	}
	
    // ...

	// Deploy - 部署 Kubernetes 资源（Sts Service 等）
	needsDeploy, err := cd.needsDeploy(ctx, cluster, log)
	if err != nil {
		return nil, err
	} else if needsDeploy {
		return cd.actors[api.DeployAction], nil
	}

	return nil, nil
}
```

以部署操作为例，通过检查 SubResource 是否修改判断是否需要执行部署：

```go
func (cd *clusterDirector) needsDeploy(ctx context.Context, cluster *resource.Cluster, log logr.Logger) (bool, error) {
    // 前置条件依赖（通过 Status.Condition 读取）
	conditions := cluster.Status().Conditions
	featureVersionValidatorEnabled := utilfeature.DefaultMutableFeatureGate.Enabled(features.CrdbVersionValidator)
	conditionInitializedTrue := condition.True(api.CrdbInitializedCondition, conditions)
	conditionInitializedFalse := condition.False(api.CrdbInitializedCondition, conditions)
	conditionVersionCheckedTrue := condition.True(api.CrdbVersionChecked, conditions)
	if !conditionInitializedTrue && !conditionInitializedFalse {
		return false, nil
	}
	if featureVersionValidatorEnabled && !conditionVersionCheckedTrue {
		return false, nil
	}

    // 子资源判断是否修改
	builders := []resource.Builder{
		resource.DiscoveryServiceBuilder{Cluster: cluster, Selector: labelSelector},
		resource.PublicServiceBuilder{Cluster: cluster, Selector: labelSelector},
		resource.StatefulSetBuilder{Cluster: cluster, Selector: labelSelector, Telemetry: kubernetesDistro},
		resource.PdbBuilder{Cluster: cluster, Selector: labelSelector},
	}
	for _, b := range builders {
		hasChanged, err := resource.Reconciler{
			ManagedResource: r,
			Builder:         b,
			Owner:           cluster.Unwrap(),
			Scheme:          cd.scheme,
		}.HasChanged()

		if err != nil {
			return false, err
		} else if hasChanged {
			return true, nil
		}
	}
	return false, nil
}
```

对应的，`Actor` 执行各个 SubResource 的管理：

```go
func (d deploy) Act(ctx context.Context, cluster *resource.Cluster, log logr.Logger) error {
	builders := []resource.Builder{
		resource.DiscoveryServiceBuilder{Cluster: cluster, Selector: labelSelector},
		resource.PublicServiceBuilder{Cluster: cluster, Selector: labelSelector},
		resource.StatefulSetBuilder{Cluster: cluster, Selector: labelSelector, Telemetry: kubernetesDistro},
		resource.PdbBuilder{Cluster: cluster, Selector: labelSelector},
	}

	for _, b := range builders {
		changed, err := resource.Reconciler{
			ManagedResource: r,
			Builder:         b,
			Owner:           owner,
			Scheme:          d.scheme,
		}.Reconcile()

		if err != nil {
			return errors.Wrapf(err, "failed to reconcile %s", b.ResourceName())
		}
	}
	return nil
}
```

## ES Operator

[**ES Operator**](https://github.com/elastic/cloud-on-k8s) 用于在 Kubernetes 中部署 ES 一套。其也是基于 Controller Runtime 实现的。

`Reconcile()` 函数宏观上分为三步：

1. 读取当前 Cluster CR；
2. 执行资源的 Reconcile，也会更新 Cluster Status；
3. 更新 Cluster Status；

资源的 Reconcile 流程都是串行执行，没有设计自己的框架。以最核心的 Workload 管理为例，核心的操作就是按照顺序执行：Change / Scale up -> Scale down -> Upgrade 的操作。

```go
func (d *defaultDriver) reconcileNodeSpecs(/* ... */) *reconciler.Results {
	results := &reconciler.Results{}

	// ...

	// Phase 1: apply expected StatefulSets resources and scale up.
	upscaleCtx := upscaleCtx{
		parentCtx:            ctx,
		k8sClient:            d.K8sClient(),
		es:                   d.ES,
		esState:              esState,
		expectations:         d.Expectations,
		validateStorageClass: d.OperatorParameters.ValidateStorageClass,
		upscaleReporter:      reconcileState.UpscaleReporter,
	}
	upscaleResults, err := HandleUpscaleAndSpecChanges(upscaleCtx, actualStatefulSets, expectedResources)
	if err != nil {
		return results.WithError(err)
	}

	if upscaleResults.Requeue {
		return results.WithReconciliationState(defaultRequeue.WithReason("StatefulSet is scheduled for recreation"))
	}

	// ...

	// Phase 2: handle sset scale down.
	// We want to safely remove nodes from the cluster, either because the sset requires less replicas,
	// or because it should be removed entirely.
	downscaleCtx := newDownscaleContext(
		ctx,
		d.Client,
		esClient,
		resourcesState,
		reconcileState,
		d.Expectations,
		d.ES,
		nodeShutdowns,
	)
	downscaleRes := HandleDownscale(downscaleCtx, expectedResources.StatefulSets(), actualStatefulSets)
	results.WithResults(downscaleRes)
	if downscaleRes.HasError() {
		return results
	}

	// ...

	// Phase 3: handle rolling upgrades.
	rollingUpgradesRes := d.handleUpgrades(ctx, esClient, esState, expectedResources)
	results.WithResults(rollingUpgradesRes)
	if rollingUpgradesRes.HasError() {
		return results
	}

	// ...
	return results
}
```

因为 Spec change 与 Scale up 仅仅是改变 StatefulSet Spec，不需要进行细节的控制，所以优先级最高放在第一个。


