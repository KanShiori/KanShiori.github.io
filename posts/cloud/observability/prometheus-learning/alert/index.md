# Prometheus Alert


## 1 基本概念

Prometheus 生态中的告警，有 Prometheus Server 和 Alertmanager 组件实现。

{{< image src="img1.png" height=260 >}}

* Prometheus Server 负责采集 Metrics，并根据 Prometheus Rule 配置的 PromQL 决定是否触发 Alert。
* Alertmanager 负责接收 Server 的 Alert 信息，然后经过处理将其路由到各个 Receiver。

## 2 Alert 处理

Alertmanager 提供一些功能对收到的 Alert 信息进行处理。

### 2.1 Grouping

Grouping 指的是对 Alert 进行分组，将多个 Alert 合并到一个 Alert。

例如，Server 宕机的情况下，可能会导致所有的 Client 都发出 Alert。通过 Grouping 可以将这些 Alert 合并会一个 Alert，避免瞬间触发大量告警通知。

当然，也可以一定时间的 Alert 进行分组打包，合并为一个 “大” 的 Alert 通知，避免受到频繁的告警轰炸。

Grouping 功能通过 [**route 配置项**](#33-route-配置) 中的相关字段使用：

```yaml
route:
  # ...
  routes:
    - receiver: customer
      group_by: ['product', 'environment']
      group_wait: 10s
      group_interval: 10m
      match:
        alerttype: 'customer'
```

* group_by 
  
  指定分组基于的 label name，不同 label 值会被设为单独的一组。
  
  例如按照上述配置，按照 `label["product"]` + `label["environment"]` 进行分组，以下告警都属于不同 group：

  * `{product="web", environment="dev}`
  * `{product="app", environment="dev}`
  * `{product="web", environment="prod}`

* group_wait（默认 30s）
  
  设置接收告警到发送的等待时间。收到某一个分组的告警后，在 `${group_wait}` 等待时间内接收同组的告警，这些告警会合并为一个告警发出。

* group_interval（默认 5m）

  相同 group 告警的间隔时间。在发出该 group 的告警后，间隔 `${group_interval}` 再能够再次发送该 group 告警。

  通过该配置来防止频繁发送相同 group 的告警。

### 2.2 Inhibition

Inhibition 指的是某个 Alert 已经通知后，停止重复发送此 Alert 引发的其他异常或者故障产生的 Alert。在实际的配置中，可以理解为：当 Source Alert 触发后，根据配置抑制 Target Alert 的告警。

例如灾备体系中，原集群宕机，流量切换到灾备集群后，那么原集群的其他告警就失去了意义，那么通过 Inhibition 就可以抑制原集群的其他告警。

Inhibition 功能通过配置文件中的 inhibit_rules 模块实现。

```yaml
inhibit_rules:
# 当 source alert 触发时，会抑制 target alert
- source_match:                # source alert 匹配规则，
    alertname: 'TORouterDown'  # k: label name, v: label value
  # source_match_re: {}
  target_match_re:             # target alert 匹配规则
    alertname: '.*Unreachable' # k: label name, v: label value regex
  # target_match: {}
  equal: ['dc', 'rack']        # source 与 target alert 这些 label 的值相同时，才会触发抑制
```

* source_match / source_match_re
  
  描述 Source Alert 的匹配条件。示例中，只有一个 Alert 包含 `alertname: TORouterDown` 的 Label 才会被视为 Source Alert。

* target_match / target_match_re
  
  描述需要被抑制的 Target Alert 的匹配条件。示例中，包含 `alertname: RedisUnreachable` 的 Alert 会被视为 Target Alert。

* equal
  
  Source Alert 与 Target Alert 具有 equal 中包含的 Label，并且值完全相同，才会触发抑制规则。
  
  该配置用于描述 Source 与 Target Alert 的是相关联的，避免误触发抑制。示例中的含义为，要求 Source 与 Target Alert 同属于一个数据中心（dc）的支架（rack）才会触发抑制规则。

### 2.3 Silence

Silence 用于将特定的告警在一定时间不在通知。

可以通过两种方式来设置 Silence：

* Alertmanager Web Console

* amtool 命令行工具

## 3 Alert 配置

Alertmanager 配置项分为：

* global 全局配置
* templates 告警模板
* route 告警路由
* receivers 接收器
* inhibit_rules 抑制规则

### 3.1 global 配置

global 中包含所有配置项的公共配置，可以作为其他配置项的默认值，自动被其他配置项的配置覆盖。

### 3.2 templates 配置

templates 中定义告警模板，自定义告警通知的外观格式以及包含的信息。

templates 的只为读取的模板文件的目录，例如：

```yaml
templates:
- '/data/alertmanager/template/*.tmpl'
```

### 3.3 route 配置

route 配置描述收到 Prometheus 的告警后，如果将告警发送到目标的 receivers。

路由规则以树型结构组合：第一个路由规则称为 **根路由节点**，后续称为 **子路由节点**。

每个告警从配置的根节点进入路由树，按照深度优先顺序遍历匹配。当匹配到某个路由规则时，就会发送给 receiver。如果没有匹配到任意的路由规则，那么从根据跟路由节点发出。

```yaml
route:
  # 根节点
  receiver: ‘admin-receiver' # 指向某个 receiver
  group_by: ['alertname', 'cluster']
  group_wait: 10s
  repeat_interval: 30m
  routes:
    # 子节点
    - receiver: customer
      repeat_interval: ${repeatInterval}
      group_by: ['alertname']
      continue: false
      match:
        alerttype: 'customer'
    - receiver: jira-${env}
      group_by: ['...']
      continue: true
      group_wait: 10s
      group_interval: 10m
      repeat_interval: 2h
      match_re:
        severity: critical|emergency|blocker|major|warning
      routes:
        # 子节点
        - receiver: jira-${env}-major
          match_re:
            severity: major
        - receiver: jira-${env}-warning
          match_re:
            severity: warning
```

可以看到，每一个路由节点包含以下配置：

* receiver - 配置发送告警的 receiver 名称
* repeat_interval -（默认 4h）设置告警成功后，再次发送完全相同的告警的间隔时间
* continue -（默认 false）为 false 时，那么告警匹配时就停止继续匹配其他路由节点。反之，会继续匹配
* match - 匹配规则，对应 label 的 value 完全匹配
* match_re - 基于正则的匹配规则，对应 label 的 value 匹配正则
* routes - 子路由节点
* group_by group_wait group_interval - 参考 [**Grouping**](#21-grouping)

{{< admonition note Note>}}
通过 [**Routing tree editor**](https://prometheus.io/webtools/alerting/routing-tree-editor/) 可以导入 Route 配置，然后测试路由规则。
{{< /admonition >}}

### 3.4 receivers

receivers 配置中定义能够接受告警通知的目标。每个 receiver 有着全局唯一的名称，对应一个或多个通知方式。

Alertmanager 支持多种的通知方式，包括 Email、微信、PagerDuty、Webhook 等。

```yaml
receivers:
- name: slack
  slack_configs:
    # ...
- name: pager_duty
  pagerduty_configs:
    # ...
```

各种 receiver 的配置方式参考官方文档 [**receiver**](https://prometheus.io/docs/alerting/latest/configuration/#receiver)。

### 3.5 inhibit_rules

参考 [**Inhibition**](#22-inhibition)。

## 4 Prometheus Rule

### 4.1 Alert Rule

Alert Rule 在 Prometheus 的配置文件中配置，Prometheus 会周期性的执行指定的 PromQL，满足条件后就会发送 Alert 给 Alertmanager。

默认情况下，Prometheus 进行告警计算的间隔是 1min，当然可以在配置文件中配置：

```yaml
global:
  evaluation_interval: 15s
```

Alert Rule 可以通过文件或者目录提供给 Prometheus，在配置文件中的 `rule_files` 项指定：

```yaml
rule_files:
  - "/etc/prometheus/rule/*_rules.yaml"
  - "/custom_rules.yaml"
```

一个 Alert Rule 文件如下所示：

```yaml
groups:
- name: Cluster # rule group name
  rules:
  - alert: UP
    annotations:
      message: Node {{ $labels.name }} has been down for more than 5 min
    expr: |
      up{job="node"} == 0
    for: 3m
    labels:
      severity: critical
      component: node
  # rules ...
```

Prometheus 所有的 Alert Rule 是按分组来管理的，每组包含多个相关的 Alert Rule。

一个 Alert Rule 包含如下配置：

* alert 告警的名称
* expr 告警计算使用的 PromQL
* for（可选）告警规则满足持续多长时间后，才会发送告警
* labels 添加到告警上的 Label
* annotations 设置告警的相关描述信息，告警发送时会作为参数一起发送

配置中，可以使用 Go 的模板语法，以动态获取告警实例的相关信息：

* `{{ $labels.<label_name> }}` - 用于获取某 Label 的值；
* `{{ $value }}` - 用于获取 PromQL 执行的结果值；

### 4.2 触发规则

配置 Alert Rules 后，Prometheus 就会周期性执行 PromQL 计算是否触发告警。

触发 Alert 会经过三种状态：

* Inactive - 没有满足条件，告警处于未激活状态 
* Pending - 首次触发告警规则时，但是并没有满足 `for` 配置项的持续时间，告警会处于 PENDING 状态
* Firing - 已经触发告警，并满足了持续时间。处于 Firing 状态下，Prometheus 会持续将告警发送

{{< admonition note Note>}}
因为 Prometheus 计算告警是周期性触发地，因此告警状态的变化与触发的时间间隔都是 N * 计算间隔，所以不是完全按照 for 配置持续来触发告警的。
{{< /admonition >}}

