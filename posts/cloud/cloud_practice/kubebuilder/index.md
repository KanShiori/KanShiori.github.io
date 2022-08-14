# 使用 KubeBuilder 编写 Controller


## 1 基本框架

执行 `kubebuilder init` 命令后，KubeBuilder 就会创建 Controller 的基本框架。

```go
var (
	scheme   = runtime.NewScheme()
	setupLog = ctrl.Log.WithName("setup")
)

func init() {
	utilruntime.Must(clientgoscheme.AddToScheme(scheme))

	//+kubebuilder:scaffold:scheme
}

func main() {
    // 命令行参数绑定
	var metricsAddr string
	var enableLeaderElection bool
	var probeAddr string
	flag.StringVar(&metricsAddr, "metrics-bind-address", ":8080", "The address the metric endpoint binds to.")
	flag.StringVar(&probeAddr, "health-probe-bind-address", ":8081", "The address the probe endpoint binds to.")
	flag.BoolVar(&enableLeaderElection, "leader-elect", false,
		"Enable leader election for controller manager. "+
			"Enabling this will ensure there is only one active controller manager.")
	opts := zap.Options{
		Development: true,
	}
	opts.BindFlags(flag.CommandLine)
	flag.Parse()

	ctrl.SetLogger(zap.New(zap.UseFlagOptions(&opts)))

    // 创建一个 Manager
	mgr, err := ctrl.NewManager(ctrl.GetConfigOrDie(), ctrl.Options{
		Scheme:                 scheme,
		MetricsBindAddress:     metricsAddr,
		Port:                   9443,
		HealthProbeBindAddress: probeAddr,
		LeaderElection:         enableLeaderElection,
		LeaderElectionID:       "67295132.shiori.cn",
	})
	if err != nil {
		setupLog.Error(err, "unable to start manager")
		os.Exit(1)
	}

	//+kubebuilder:scaffold:builder

    // 添加健康检查
	if err := mgr.AddHealthzCheck("healthz", healthz.Ping); err != nil {
		setupLog.Error(err, "unable to set up health check")
		os.Exit(1)
	}
	if err := mgr.AddReadyzCheck("readyz", healthz.Ping); err != nil {
		setupLog.Error(err, "unable to set up ready check")
		os.Exit(1)
	}

    // 启动
	setupLog.Info("starting manager")
	if err := mgr.Start(ctrl.SetupSignalHandler()); err != nil {
		setupLog.Error(err, "problem running manager")
		os.Exit(1)
	}
}
```

### 1.1 Scheme

Scheme 表示 CRD 的结构，`controller-runtime` 包的 Client 就是基于 Scheme 来解析消息的。

因此，我们需要将所有 Client 将用到的资源的 Scheme 注册到生成的 scheme 对象中。

注册方式很简单，当我们使用 KubeBuilder 或者原始 Kubernetes 方式生成 API Client 后，都会生成对应的 `AddToScheme()` 函数。

```go
var (
    scheme   = runtime.NewScheme()
    setupLog = ctrl.Log.WithName("setup")
)

func init() {
    utilruntime.Must(clientgoscheme.AddToScheme(scheme)) // Kubernete 资源
    utilruntime.Must(v1beta1.AddToScheme(scheme)) // 自定义资源

    //+kubebuilder:scaffold:scheme
}
```

### 1.2 Manager

Manager 包含了各个 Controller 的通用依赖，通过 Manager 来给各个 Controller 共享 Cache 或者 Client。

初始化后，会创建一个 Manager，后续实现自己的 Controller 后，就会注册到 Manager 中。

```go
    // ...
	mgr, err := ctrl.NewManager(ctrl.GetConfigOrDie(), ctrl.Options{
		Scheme:                 scheme,
		MetricsBindAddress:     metricsAddr,
		Port:                   9443,
		HealthProbeBindAddress: probeAddr,
		LeaderElection:         enableLeaderElection,
		LeaderElectionID:       "67295132.shiori.cn",
	})
	if err != nil {
		setupLog.Error(err, "unable to start manager")
		os.Exit(1)
	}
```

当所有的 Controller 注册到 Manager 中后，`main` 函数结尾会启动 Manager，也就是启动了所有的 Controller。

```go
	setupLog.Info("starting manager")
	if err := mgr.Start(ctx); err != nil {
		setupLog.Error(err, "problem running manager")
		os.Exit(1)
	}
```

## 2 核心对象

### 2.1 Reconciler

`Reconciler` 代表着我们自己实现的处理资源的逻辑，其 interface 定义如下：

```go
type Reconciler interface {
	// Reconciler performs a full reconciliation for the object referred to by the Request.
	// The Controller will requeue the Request to be processed again if an error is non-nil or
	// Result.Requeue is true, otherwise upon completion it will remove the work from the queue.
	Reconcile(context.Context, Request) (Result, error)
}
```

将 Reconciler 注册到 Controller 后，当监听的资源发生变化就会调用 `Reconcile()` 函数。

在我们使用 KubeBuilder 创建 API 资源后，会自动创建对应的 `Reconciler` 对象：

```go
type ClusterReconciler struct {
	client.Client
	Scheme *runtime.Scheme
}
```

* `Client` 提供了 Kubernetes Client，可用于 Get/List/Update/Patch 资源，也包括了 Status 更新接口。

无论是资源的增删改，都会触发 `Reconcile()` 函数，并通过 Result 来决定是否需要 requeue。 参数 Request 参数只提供了资源的 Namesapce/Name，我们需要自行通过 `Client` 来获取对应的资源对象。

```go
func (r *ClusterReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
	var cluster v1.Cluster
	err := r.Get(ctx, req.NamespacedName, &cluster)
}
```

### 2.2 Controller

当我们编写了一个 Reconciler 后，需要注入到 `Controller` 中，Controller 用于定义如何监听资源。Controller 最后注册到 Manager 中，由此来共享 Manager 的 Cache / Client 等资源。

Controller 的注册函数基本如下：

```go
import ctrl "sigs.k8s.io/controller-runtime"

ctrl.NewControllerManagedBy(mgr).
    For(&v1.Cluster{}, builder.WithPredicates(pdt)).
    WithOptions(controller.Options{RateLimiter: common.DefaultRateLimiter()}).
    WithEventFilter(p).
    Complete(r)
```

1. 调用 `NewControllerManagedBy` 基于 Manager 创建 Controller Builder。
2. 围绕如何触发 Reconcile 来配置 Builder。
3. 调用 Complete/Build 来完成注册。

Builder 支持的方法包括：

* 资源监听

  * For - 表明监听资源的 create / delete / update 事件。

  * Owns - 表明监听该资源的子资源，当子资源 create / delete / update 时，会触发其 Own 资源的 Reconcile。

  * Watches - 底层的 Watch 函数，自定义监听的事件以及过滤条件。

* 配置

  * WithEventFilter - 根据 Label 等条件过滤事件。

  * WithOptions - 覆盖 Manager 的配置以控制 Reconcile 的行为，例如 Rate Limit 和日志等配置。

* 完成 Build

  * Complete - 注入 Reconciler。

  * Build - 注入 Reconciler 并返回 Controller 对象。

### 2.3 Manager

Manager 用于包含各个 Controller 共享的对象，例如 Cache / Client / Scheme 等对象。

```go
// Cluster provides various methods to interact with a cluster.
type Cluster interface {
	// SetFields will set any dependencies on an object for which the object has implemented the inject
	// interface - e.g. inject.Client.
	// Deprecated: use the equivalent Options field to set a field. This method will be removed in v0.10.
	SetFields(interface{}) error

	// GetConfig returns an initialized Config
	GetConfig() *rest.Config

	// GetScheme returns an initialized Scheme
	GetScheme() *runtime.Scheme

	// GetClient returns a client configured with the Config. This client may
	// not be a fully "direct" client -- it may read from a cache, for
	// instance.  See Options.NewClient for more information on how the default
	// implementation works.
	GetClient() client.Client

	// GetFieldIndexer returns a client.FieldIndexer configured with the client
	GetFieldIndexer() client.FieldIndexer

	// GetCache returns a cache.Cache
	GetCache() cache.Cache

	// GetEventRecorderFor returns a new EventRecorder for the provided name
	GetEventRecorderFor(name string) record.EventRecorder

	// GetRESTMapper returns a RESTMapper
	GetRESTMapper() meta.RESTMapper

	// GetAPIReader returns a reader that will be configured to use the API server.
	// This should be used sparingly and only when the client does not fit your
	// use case.
	GetAPIReader() client.Reader

	// Start starts the cluster
	Start(ctx context.Context) error
}

// Manager initializes shared dependencies such as Caches and Clients, and provides them to Runnables.
// A Manager is required to create Controllers.
type Manager interface {
	// Cluster holds a variety of methods to interact with a cluster.
	cluster.Cluster

	// Add will set requested dependencies on the component, and cause the component to be
	// started when Start is called.  Add will inject any dependencies for which the argument
	// implements the inject interface - e.g. inject.Client.
	// Depending on if a Runnable implements LeaderElectionRunnable interface, a Runnable can be run in either
	// non-leaderelection mode (always running) or leader election mode (managed by leader election if enabled).
	Add(Runnable) error

	// Elected is closed when this manager is elected leader of a group of
	// managers, either because it won a leader election or because no leader
	// election was configured.
	Elected() <-chan struct{}

	// AddMetricsExtraHandler adds an extra handler served on path to the http server that serves metrics.
	// Might be useful to register some diagnostic endpoints e.g. pprof. Note that these endpoints meant to be
	// sensitive and shouldn't be exposed publicly.
	// If the simple path -> handler mapping offered here is not enough, a new http server/listener should be added as
	// Runnable to the manager via Add method.
	AddMetricsExtraHandler(path string, handler http.Handler) error

	// AddHealthzCheck allows you to add Healthz checker
	AddHealthzCheck(name string, check healthz.Checker) error

	// AddReadyzCheck allows you to add Readyz checker
	AddReadyzCheck(name string, check healthz.Checker) error

	// Start starts all registered Controllers and blocks until the context is cancelled.
	// Returns an error if there is an error starting any controller.
	//
	// If LeaderElection is used, the binary must be exited immediately after this returns,
	// otherwise components that need leader election might continue to run after the leader
	// lock was lost.
	Start(ctx context.Context) error

	// GetWebhookServer returns a webhook.Server
	GetWebhookServer() *webhook.Server

	// GetLogger returns this manager's logger.
	GetLogger() logr.Logger

	// GetControllerOptions returns controller global configuration options.
	GetControllerOptions() v1alpha1.ControllerConfigurationSpec
}
```

因此，各个 Reconciler 中的 Client 或者 Cache 都可以从 Manager 中获取。

创建 Manager 时必须传入 kubeconfig，并提供一些参数控制行为。

```go
	mgr, err := ctrl.NewManager(ctrl.GetConfigOrDie(), ctrl.Options{
		Scheme:                 scheme,
		MetricsBindAddress:     metricsAddr,
		Port:                   9443,
		HealthProbeBindAddress: probeAddr,
		LeaderElection:         enableLeaderElection,
		LeaderElectionID:       "2100d6c4.infra.pingcap.com",
	})
```

将各个 Controller 注册到 Manager 后，由 Manager 来统一启动与停止。当 Manager 启动后，就会进行 Cache 的第一次填充，并且启动资源的监听。

```go
	if err := mgr.Start(ctrl.SetupSignalHandler()); err != nil {
		setupLog.Error(err, "problem running manager")
		os.Exit(1)
	}
```

具体的启动操作包括：

1. 启动 Metric 与 Health 接口。
2. 启动 Webhook。
3. 填充 Cache。
4. 获取 leader election，成功后启动各个 Controller 的监听。

## 3 Webhook

使用 KubeBuilder 可以生成为自定义类型创建 Webhook，通过 `kubebuilder create webhook` 命令。

```bash
kubebuilder create webhook --group batch --version v1 --kind CronJob --conversion
```

KubeBuilder 会自动为自定义资源创建 Webhook 相关的函数，并且在 `main` 函数中注册 Webhook。

```go
func (r *CronJob) SetupWebhookWithManager(mgr ctrl.Manager) error {
    return ctrl.NewWebhookManagedBy(mgr).
        For(r).
        Complete()
}

func main() {
    if err = (&batchv1.CronJob{}).SetupWebhookWithManager(mgr); err != nil {
        setupLog.Error(err, "unable to create webhook", "webhook", "CronJob")
        os.Exit(1)
    }
	// ...
}
```

### 3.1 Mutate

KubeBuilder 自动创建的 `Default()` 函数用于在 CR 写入 ETCD 前为 CR 设定一些默认值，在 Mutating Webhook 阶段被执行。

```go
func (r *CronJob) Default() {
    if r.Spec.ConcurrencyPolicy == "" {
        r.Spec.ConcurrencyPolicy = AllowConcurrent
    }
	// ...
}
```

### 3.2 Validate

KubeBuilder 也会自动创建 Validate 相关的函数，可以实现对 Create / Update / Delete 操作的。在 Validate Webhook 阶段被执行。

```go
// ValidateCreate 实现 webhook.Validator 使这类型的 webhook 被成功注册
func (r *CronJob) ValidateCreate() error {
    return r.validateCronJob()
}

// ValidateUpdate 实现 webhook.Validator 使这类型的 webhook 被成功注册
func (r *CronJob) ValidateUpdate(old runtime.Object) error {
    return r.validateCronJob()
}

// ValidateDelete 实现 webhook.Validator 使这类型的 webhook 被成功注册
func (r *CronJob) ValidateDelete() error {
    return nil
}
```

{{< admonition note Note>}}
ValidateDelete 是在调用 APIServer Delete 接口时触发的，因此是 CR 的 delete stamp 设置之前触发。
{{< /admonition >}}

## 4 controller-gen

## 参考

* [**KubeBuilder Book**](https://book.kubebuilder.io/introduction.html)
