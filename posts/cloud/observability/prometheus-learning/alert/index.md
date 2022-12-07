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

Alertmanager 的所有配置都是通过配置文件提供的，通过参数 `--config.file` 指定配置文件：

```command
./alertmanager --config.file=alertmanager.yml
```

Alertmanager 也支持动态重新加载配置文件，可以通过发生 `SIGHUP` 信号或者向 `/-/reload` HTTP 接口发送 POST 请求重新加载配置文件。

Alertmanager 配置项分为：

* global 全局配置
* templates 告警模板
* route 告警路由
* receivers 接收器
* inhibit_rules 抑制规则

完整的配置文件说明见官方文档：[**CONFIGURATION**](https://prometheus.io/docs/alerting/latest/configuration/)。

### 3.1 global 配置

`global` 配置项中包含所有配置项的公共配置，可以作为其他配置项的默认值，自动被其他配置项的配置覆盖。

### 3.2 templates 配置

当 Alertmanager 向各个 Receivers 发送告警通知时，会通过 template 来构建通知信息。

Alertmanager 带有默认的 template。当然，也可以在 `templates` 配置项中定义告警模板，例如：

```yaml
templates:
- '/data/alertmanager/template/*.tmpl'
```

#### 3.2.1 template 数据结构

template 会使用 Go 模板语言构建的，因此可以在 template 实现代码逻辑以及变量的使用。

当然，Alertmanager 提供了预定义数据与函数可以在模板文件中使用。

```
{{ .ExternalURL }}/#/alerts?receiver={{ .Receiver }}
```

完整的数据结构见 [**NOTIFICATION TEMPLATE REFERENCE**](https://prometheus.io/docs/alerting/latest/notifications/)。

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

参考 [**Alert Rule**](../prometheus-basic/#32-alert-rule)。
