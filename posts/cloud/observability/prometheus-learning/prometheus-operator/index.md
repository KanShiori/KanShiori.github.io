# Prometheus Operator


Prometheus Operator 提供了 Kubernetes 上原生部署 Prometheus 以及其他监控组件的能力。

## 1 基本模型

为了方便的管理 Prometheus 以及监控，Prometheus Operator 提供了一下的 CR：

* Prometheus - 用于部署 Prometheus
* Alertmanager - 用于部署 Alertmanager
* ThanosRuler - 用于描述期望的 Thanos Ruler
* ServiceMonitor
* PodMonitor
* Probe
* PrometheusRule
* AlertmanagerConfig

## 2 监控

### 2.1 Prometheus

`Prometheus` CRD 定义了期望运行的 Prometheus。

对于每一个 Prometheus 实例，Operator 会部署一个或多个 `StatefulSet` 实例。`StatefulSet` 的数量等同于 Shard 的数量。

`Prometheus` 配置中，通过 Label/Namespace Selector 来关联到相关的 `ServiceMonitor` `PodMonitor` `Probe` `PrometheusRules`。

```yaml
apiVersion: monitoring.coreos.com/v1
kind: Prometheus
metadata:
  name: k8s
  namespace: monitoring
spec:
  # ...
  alerting:
    alertmanagers:
    - apiVersion: v2
      name: alertmanager-main
      namespace: monitoring
      port: web
  image: quay.io/prometheus/prometheus:v2.40.1
  nodeSelector:
    kubernetes.io/os: linux
  serviceMonitorNamespaceSelector: {}
  serviceMonitorSelector: {}
  podMonitorNamespaceSelector: {}
  podMonitorSelector: {}
  probeNamespaceSelector: {}
  probeSelector: {}
  ruleNamespaceSelector: {}
  ruleSelector: {}
  replicas: 2
  scrapeInterval: 30s
  evaluationInterval: 30s
  version: 2.40.1
```

* `serviceMonitorNamespaceSelector` 与 `serviceMonitorSelector` - 关联的 `ServiceMonitor` 的 Namespace/Label Selector

* `podMonitorNamespaceSelector` 与 `podMonitorSelector` - 关联的 `PodMonitor` 的 Namespace/Label Selector

* `probeNamespaceSelector` 与 `probeSelector` - 关联的 `Probe` 的 Namespace/Label Selector

* `ruleNamespaceSelector` 与 `ruleSelector` - 关联的 `PrometheusRules` 的 Namespace/Label Selector

#### 2.1.1 与 Alertmanager 集成

`Prometheus` CR 中通过 `alerting` 配置需要发送给的 Alertmanager：

```yaml
# ...
spec:
  alerting:
    alertmanagers:
    - apiVersion: v2
      name: alertmanager-main
      namespace: monitoring
      port: web
```

### 2.2 ServiceMonitor

`ServiceMonitor` CRD 依照 `Service` 的概念定义了采集哪些 Pod 的 Metric。

Prometheus 实例会遍历相关联的 `ServiceMonitor`，然后遍历出 `Service` 相关联的 `Endpoints` 对象得到 Pod IP，最后访问 Pod 的 HTTP Metric 接口采集信息。

{{< admonition note Note>}}
因为 Operator 使用 `Endpoints` 对象，所以意味着需要采集 Metric 必须需要对应的 `Service` 对象。
{{< /admonition >}}

通过这种方式，无论底层的 Pod 如何变化，Prometheus 按照约定自动采集新的 Pod 的 Metric 信息，而不需要修改配置。

示例配置如下：

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: quota-monitor
  namespace: monitoring
spec:
  endpoints:
  - interval: 15s
    path: /metrics
    port: metrics
    relabelings:
    - action: replace
      regex: (.*)
      replacement: $1
      sourceLabels:
      - __meta_kubernetes_pod_node_name
      targetLabel: instance
  jobLabel: app.kubernetes.io/name
  selector:
    matchLabels:
      app.kubernetes.io/component: quota-monitor
  namespaceSelector: {}
```

* `endpoints` - 定义采集 Metric 调用的接口，以及鉴权相关信息

* `jobLabel` - 指定 Metric 的 Job Label 使用的 `Service` Label
  
  为空时，Metric 的 Job Label 默认为 `Service` Name

* `namespaceSelector` 与 `selector` - 关联的 `Endpoints` 的 Namespace/Label Selector
  
  `namespaceSelector` 为空时，表明匹配与 `ServiceMonitor` 相同 Namespace 的 `Endpoints`。

  如果想要匹配所有的 Namespace，使用 `any` 配置：
  
  ```yaml
  namespaceSelector:
    any: true
  ```

对应的 `Endpoints` 为：

```yaml
# kubectl get endpoints -n monitoring --selector app.kubernetes.io/component=quota-monitor -o yaml
apiVersion: v1
kind: Endpoints
metadata:
  labels:
    app.kubernetes.io/component: quota-monitor  # match `selector`
  name: quota-monitor
  namespace: monitoring  # match `namespaceSelector`
subsets:
- addresses:
  - ip: 10.0.105.82
    nodeName: node1
    targetRef:
      kind: Pod
      name: quota-monitor-695c477f58-hjzzw
      namespace: monitoring
  ports:
  - name: metrics  # match `endpoints[].port`
    port: 8080
    protocol: TCP
```

可以看到，对应的 Prometheus 会访问 Endpoints 中每个 Pod 的 `http://<ip>:8080/metrics` 端口采集 Metric 信息。

### 2.3 PodMonitor

`PodMonitor` CRD 定义了采集哪些 Pod 的 Metric。

Prometheus 实例会遍历相关联的 `PodMonitor`，然后遍历出相关联的 `Pod`，最后访问 Pod 的 HTTP Metric 接口采集信息。

例如，以下配置：

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PodMonitor
metadata:
  name: example-app
  labels:
    team: frontend
spec:
  podMetricsEndpoints:
  - interval: 15s
    path: /metrics
    port: metrics
  namespaceSelector: {}
  selector:
    matchLabels:
      app: example-app
```

示例配置如下：

* `podMetricsEndpoints` - 定义采集 Metric 调用的接口

* `namespaceSelector` 与 `selector` - 关联的 `Pod` 的 Namespace/Label Selector

### 2.4 Probe

`Probe` CRD 提供了一种黑盒监控的功能。通过 `Probe` 可以支持对 HTTP 接口的探测，具体支持 Ingress 与 Static Target 两种目标。

不过 `Probe` 只是描述了监控目标，具体执行探测需要部署 [**Blackbox Exporter**](https://prometheus-operator.dev/docs/kube/blackbox-exporter/) 组件。

```yaml
kind: Probe
apiVersion: monitoring.coreos.com/v1
metadata:
  name: example-com-website
  namespace: monitoring
spec:
  interval: 60s
  module: http_2xx
  prober:
    url: blackbox-exporter.monitoring.svc.cluster.local:19115
  targets:
    staticConfig:
      static:
      - http://example.com
      - https://example.com
```

## 3 告警

### 3.1 Alertmanager

`Alertmanager` CRD 定义了期望运行的 Alertmanager。

对于每一个 `Alertmanager` 实例，Operator 会部署在同 Namespace 部署一个 `StatefulSet`。`Alertmanager` 使用的配置文件保存在名为 `alertmanager-<name>` 的 `Secret` 中。

```yaml
apiVersion: monitoring.coreos.com/v1
kind: Alertmanager
metadata:
  name: main
  namespace: monitoring
spec:
  configMaps:
  - alertmanager-templates-tmpl
  secrets:
  - alertmanager-main
  externalUrl: https://alertmanager.com
  image: prom/alertmanager:v0.24.0
  replicas: 1
  storage:
    volumeClaimTemplate: {}
  version: 0.24.0
```

* configMaps - 提供给 Alertmanager 的 `ConfigMap`，挂载到 `/etc/alertmanager/configmaps/<name>` 文件
* secrets - 提供给 Alertmanager 的 `Secret`，挂载到 `/etc/alertmanager/secrets/<name>` 文件
* externalUrl - Alertmanager 控制台的 URL

### 3.2 PrometheusRule

`PrometheusRule` CRD 定义了提供给 Prometheus 或者 Thanos Ruler 的告警规则。

告警规则会由 Operator 管理，自动重新加载，不需要重启 Prometheus 或 Thanos Ruler。

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: service
  namespace: monitoring
spec:
  groups:
  - name: service
    rules:
    - alert: Down
      annotations:
        message: The Service down.
      expr: absent(up{job="service"} == 1)
      for: 10m
      labels:
        severity: critical
```

具体的配置字段可以参考：[**Alert Rule**](../prometheus-basic/#3-prometheus-rule)。

### 3.3 AlertmanagerConfig

`AlertmanagerConfig` 定义了 Alertmanager 的配置子项，支持配置 Alert Route 与 Inhibition。多个 `AlertmanagerConfig` 会聚合为一个配置，并提供给 Alertmanager。

```yaml
apiVersion: monitoring.coreos.com/v1alpha1
kind: AlertmanagerConfig
metadata:
  name: config-example
  labels:
    alertmanagerConfig: example
spec:
  route:
    groupBy: ['job']
    groupWait: 30s
    groupInterval: 5m
    repeatInterval: 12h
    receiver: 'webhook'
  receivers:
  - name: 'webhook'
    webhookConfigs:
    - url: 'http://example.com/'
```

具体的配置字段可以参考：[**Alert 配置**](../alert/#3-alert-配置)。

## 参考

* 官方文档：[**Prometheus Operator**](https://prometheus-operator.dev/docs/prologue/introduction/)
