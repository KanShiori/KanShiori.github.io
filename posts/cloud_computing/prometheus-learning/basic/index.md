# Prom 学习 - 基本概念


## 1 数据模型
Prometheus 保存的所有数据都是 [时间序列]^(time series) 数据，也就是每个数据会带有一个时间戳。

通过图表可以理解其模型：
{{< find_img "img1.png" >}}
* 图表中每个点都是一个时间序列数据，也称为 [样本]^(Sample)。
* 不同的 label 与 Metric name 组合标识了一个数据流，也就是上面图表的一条线。
* 所有的数据都表示 prometheus_http_request_total 项统计。

所以，其数据模型为：
* **`Metric`** - 统计项
* **`Metric Name`** 与 **`Labels`** - 统计项下的一个数据流
* **`Sample`** - 数据流中的某个数据

### 1.1 Metric
Metric 定义了某个监控指标，都由如下格式表示：
```
<metric name>{<label name>=<label value>, ...}
```

该格式标识也被称为 **`Notation`**。

Metric name 表示监控项的含义，Labels 反映了样本的特征维度：
```metric
node_cpu_seconds_total{cpu="0",mode="idle"} 2.01377426e+06
node_cpu_seconds_total{cpu="0",mode="iowait"} 1639.03
node_cpu_seconds_total{cpu="1",mode="idle"} 1.9973224e+06
node_cpu_seconds_total{cpu="1",mode="iowait"} 1535.02
```

可以看到，Labels 是不同值的组合，都是一个 Metric 实例化。因此 Metric name 与 Labels 的组合才代表一个唯一的数据。

其中，__ 作为前缀的 label 是系统保留的，只能在系统内部使用。例如 Prometheus 底层实现中指标名称实际上是以 `__name__=<metric name>` 形式保存的：
```metric
{__name__="node_cpu_seconds_total", cpu="0",mode="idle"}
```

### 1.3 Sample

Sample 为样本，可以理解为某个采集项某次采集的数据。按照时间排序的 Sample 形成了实际的数据序列数据列表。

每个 Sample 包含三个部分：
* **metric** - metric name 与当前样本的 labelsets
* **value** - 采集的数据，float64
* **timestamp** - 采集数据的时间，ms

Prometheus 会按照时间顺序来存储 sample，如果 sample 不是顺序收集的，会将其丢弃。

## 2 Metric 类型
目前，Prometheus 提供了 4 种 Metric 类型。
* Counter
* Gauge
* Histogram
* Summary

### 2.1 Counter
**`Counter`** 代表一个**累加数据**，通常用于跟踪事件次数，累计时间等。

其特点如下：
* **值只能从 0 开始增加，不能减少**。
* 重启进程后，只会重置为 0。

通常，Counter 类指标会命名为 "xxx_total"。
```metric
# TYPE node_cpu_seconds_total counter
node_cpu_seconds_total{cpu="0",mode="idle"} 2.01526517e+06
node_cpu_seconds_total{cpu="0",mode="iowait"} 1640.01
```

### 2.2 Gauge

**`Gauge`** 反映一个**瞬时测量值**，其值可能随着时间变化上下波动。

其特点如下：

* **测量值是瞬时值，可以任意变化**。
* 重启进程后，会被重置。

Gauge 是适合记录无规律变化的数据。

```metric
# HELP node_load1 1m load average.
# TYPE node_load1 gauge
node_load1 1.6
# HELP node_load15 15m load average.
# TYPE node_load15 gauge
node_load15 1.01
```

### 2.3 Histogram
**`Histogram`** 表示直方图，会在一段时间范围内对数据进行采样，并将其计入 bucket 中。

histogram 中有三类值：
* **bucket**：值小于 "le" 下，统计到的数据次数
* **sum**：统计值的累计和
* **count**：统计次数

看个例子，下面数据表明了对于 / 的 HTTP 请求的
```metric
# HELP prometheus_http_request_duration_seconds Histogram of latencies for HTTP requests.
# TYPE prometheus_http_request_duration_seconds histogram
prometheus_http_request_duration_seconds_bucket{handler="/",le="0.1"} 1
prometheus_http_request_duration_seconds_bucket{handler="/",le="0.2"} 1
prometheus_http_request_duration_seconds_bucket{handler="/",le="0.4"} 1
prometheus_http_request_duration_seconds_bucket{handler="/",le="1"} 1
prometheus_http_request_duration_seconds_bucket{handler="/",le="3"} 1
prometheus_http_request_duration_seconds_bucket{handler="/",le="8"} 1
prometheus_http_request_duration_seconds_bucket{handler="/",le="20"} 1
prometheus_http_request_duration_seconds_bucket{handler="/",le="60"} 1
prometheus_http_request_duration_seconds_bucket{handler="/",le="120"} 1
prometheus_http_request_duration_seconds_bucket{handler="/",le="+Inf"} 1
prometheus_http_request_duration_seconds_sum{handler="/"} 0.000184525
prometheus_http_request_duration_seconds_count{handler="/"} 1
```
* bucket

    le="0.1" - 请求处理时间小于 0.1s 的次数为 1 次

    le="0.2" - 请求处理时间小于 0.2s 的次数为 1 次

    ...

    le="+Inf" - 请求处理时间小于 +Inf 的次数为 1 次
* sum

    所有请求的处理实际的总和为 0.000184525

* count

    统计的请求次数为 1 次

可以看到，**对于 bucket 的统计结果是累计的**。

### 2.4 Summary
**`summary`** 为概率图，包含了各百分比样本的值，也就是样本的值的分布区间。

summary 包含三类数据：
* **bucket**：百分比范围样本的值，"quantile" 表示百分比
* **sum**：统计值的累计和
* **counter**：统计次数

看个示例，下面数据表明了所有 wal_fsync 操作的耗时分布区间：
```metric
# HELP prometheus_tsdb_wal_fsync_duration_seconds Duration of WAL fsync.
# TYPE prometheus_tsdb_wal_fsync_duration_seconds summary
prometheus_tsdb_wal_fsync_duration_seconds{quantile="0.5"} 0.012352463
prometheus_tsdb_wal_fsync_duration_seconds{quantile="0.9"} 0.014458005
prometheus_tsdb_wal_fsync_duration_seconds{quantile="0.99"} 0.017316173
prometheus_tsdb_wal_fsync_duration_seconds_sum 2.888716127000002
prometheus_tsdb_wal_fsync_duration_seconds_count 216
```
* bucket

    quantile="0.5" - 50% 的 wal_fsync 操作耗时小于 0.012352463

    quantile="0.9" - 90% 的 wal_fsync 操作耗时小于 0.014458005

    quantile="0.99" - 99% 的 wal_fsync 操作耗时小于 0.017316173
* sum

    所有 wal_fsync 操作耗时总和 2.888716127000002
* count

    统计 wal_fsync 操作 216 次

可以看到，对于 bucket 的统计结果是累计的。

## 3 任务模型
在 Prometheus 中，任何被采集的目标（往往是一个 IP:Port）被称为 **`Intsance`**。而将相同采集项进行分组，每个组就称为 **`Job`**。

看下 prometheus.yml 配置文件来理解这个概念：
```yaml
scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']
  - job_name: 'node'
    static_configs:
      - targets: ['localhost:9100']
```
* `scrape_configs.job_name` 描述了一个脚本，这里有两个 job，分别表示采集 Prometheus 与 Node。
* `scrape_configs.static_configs.targets` 中就表明了需要采集的 Instance。

Prometheus 拉取一个采集样本时，会自动在时序的基础上添加 `"job"` 与 `"instance"` 两个 label，用于识别被采集的目标。
```metric
prometheus_http_requests_total{code="200", handler="/-/ready", instance="localhost:9090", job="prometheus"}
```
{{< admonition note Note>}}
如果这两个 label 已经存在了，那么会根据配置文件中的 `honor_label` 配置来决定如何处理。
{{< /admonition >}}

对于每个被采集的 Instance，Prometheus 也会有着一些统计数据。
* **up** - 如果 Instance 能够被正常采集，那么值为 1 ，否则为 0

    ```metric
    up{instance="localhost:9090", job="prometheus"} 1
    ```
* **scrape_duration_seconds** - 采集 Instance 累计的使用时间

    ```metric
    scrape_duration_seconds{instance="localhost:9090", job="prometheus"} 0.006962203
    ```
* **scrape_samples_post_metric_relabeling** - Instance 的采集数据被 relabel 后，剩余的采集数量累计

    ```metric
    scrape_samples_post_metric_relabeling{instance="localhost:9090", job="prometheus"} 723
    ```
* **scrape_samples_scraped** - 从该 Instance 采集数据的次数

    ```metric
    scrape_samples_scraped{instance="localhost:9090", job="prometheus"} 723
    ```

## 参考
* [**官方文档**](https://prometheus.io/docs/concepts)
* [**Blog: 一文搞懂 Prometheus 的直方图**](https://www.sohu.com/a/347228004_753508)
