---
date: 2024-07-07
categories:
    - :rainbow:Topics
    - :writing_hand_tone1:Observability
    - :writing_hand_tone1:Prometheus
---

# Prometheus

[Prometheus](https://prometheus.io/) 采集系统指标 (metrics)，并将其存储为时间序列 (time series)；并提供了一套强大的查询语言 PromQL，用于查询和分析这些指标。通常，我们会使用 Grafana 来把这些指标可视化。

本文主要介绍 Prometheus 的基本概念和查询方法。

<!-- more -->

!!! info
    如果暂时从未实际接触过 Prometheus，您可以在 [这里](https://grafana.com/blog/2019/12/04/how-to-explore-prometheus-with-easy-hello-world-projects/) 找到一个 minimum 的 end-to-end 示例。

## 1 Concepts | 基本概念

### 1.1 Data Model | 数据模型

Prometheus 存储的内容是一系列 `(metric_name, labels, timestamp, value)`；其中：

- `metric_name` 是指标名，用于表明指标的含义 (e.g. `http_requests_total`)；
- `labels` 是可选的，包含一组任意键值对，用于区分指标的不同维度 (e.g. `method="GET"`)；
    - 例如 `http_requests_total{method="GET", status="200"}` 表示 `GET` 请求返回 `200` 的总次数；
- `(timestamp, value)` 被合并称为 `sample`，表示在某个时间点下 `metric_name{labels}` 的值。
    - `timestamp` 的精度是毫秒，`value` 是一个 float64 数值。

### 1.2 Prometheus 如何采集指标

常见语言都有 Prometheus 的 client library，用于在应用程序中采集指标并暴露出来。例如，服务可以在 `/metrics` 路径下暴露指标，指标可能形如：

```hl_lines="11-15"
# HELP python_gc_objects_collected_total Objects collected during gc
# TYPE python_gc_objects_collected_total counter
python_gc_objects_collected_total{generation="0"} 4.6086284e+07
python_gc_objects_collected_total{generation="1"} 2.1484146e+07
python_gc_objects_collected_total{generation="2"} 6.290886e+06
# HELP python_gc_objects_uncollectable_total Uncollectable objects found during GC
# TYPE python_gc_objects_uncollectable_total counter
python_gc_objects_uncollectable_total{generation="0"} 0.0
python_gc_objects_uncollectable_total{generation="1"} 0.0
python_gc_objects_uncollectable_total{generation="2"} 0.0
# HELP http_requests_total Total number of HTTP requests
# TYPE http_requests_total counter
http_requests_total{method="post",code="200"} 1027.0
http_requests_total{method="post",code="400"} 3.0
http_requests_total{method="get",code="200"} 10000.0
```

在服务中，每次接受到 http 请求时，会根据请求的 method 和 code 修改一次 `http_requests_total` 指标。Prometheus 每隔指定的一段时间会去 `/metrics` 拉取这些指标，获得上面代码块所示的数据，从而获取到 `(metric_name, labels, timestamp, value)` 的 data model。

!!! note
    在 Prometheus 看来，一个 `metric_name{labels}` 的值是一个时间序列，即一系列 `(timestamp, value)` 的集合。如果 `labels` 的取值发生变化，Prometheus 会认为这是一个新的时间序列。

另外，在每次抓取数据时，Prometheus 还会额外记录以下指标：

- `up`，1 表示抓取成功，0 表示失败；这对于检查实例是否可用非常有用；
- `scrape_duration_seconds`，抓取数据的耗时；
- 还有更多，参见 [Automatically generated labels and time series](https://prometheus.io/docs/concepts/jobs_instances/#automatically-generated-labels-and-time-series)。

!!! info
    有时，需要被监控的组件可能不支持被 Prometheus 直接抓取，此时可以在组件中 [主动 Push](https://prometheus.io/docs/instrumenting/pushing/)；但这不会被包含在本文的内容中。。

### 1.3 Metric Types | 指标类型

Prometheus 的 client library 支持以下 4 种指标类型；但这只是一个便于使用的封装，在 Prometheus 中的指标仍然只有 `(metric_name, labels, timestamp, value)`

- `Counter`：只增不减的计数器。在 client 中，支持 `c.inc()` 或 `c.inc(1.6)` 的操作来增加其值。
- `Gauge`：可增可减的值。在 client 中，支持 `g.set(3.2)`, `g.inc()` 或 `g.dec(10)` 的操作来修改其值。
- `Histogram`：对观察的结果进行采样，记录到预先指定好的若干范围 (bucket) 中，用于统计数据的分布情况。在 client 中，支持 `h.observe(3.2)` 的操作来记录数据。参见下面的例子。
- `Summary`：对观察的结果进行采样，记录到预先指定好的若干分位数 (quantile) 中，用于统计数据的分布情况。在 client 中，支持 `s.observe(3.2)` 的操作来记录数据。参见下面的例子。

!!! warning
    请注意：并非所有的 client library 都支持所有的指标类型。例如，[截至目前](https://github.com/prometheus/client_python/issues/888) Python 的 client library `prometheus_client` 不支持 `Summary` 的 quantile 配置。

??? example "Histogram"
    `Histogram` 会将观察到的数据分桶 (bucket)，然后记录每个分桶的数据量。在创建一个 `Histogram` 指标时，需要指定各个 bucket 的上限值；`Histogram` 实际上会创建一系列不同的 metric 来记录观察得到的数据。下面是一个例子。

    举例而言，我们使用名为 `http_request_duration_seconds` 的 histogram 指标来记录请求的响应时间；它包含 `api` 和 `method` 两个 label；我们定义了如下的几个 bucket: `(0.1, 0.5, 1.0, Inf)`。那么，假如有 `#!python http_request_duration_seconds(api="/foo", method="GET").observe(0.8)`，那么 client 会记录如下的数据：

    ```linenums="1"
    http_request_duration_seconds_bucket{le="0.1",api="/foo",method="GET"} 0
    http_request_duration_seconds_bucket{le="0.5",api="/foo",method="GET"} 0
    http_request_duration_seconds_bucket{le="1.0",api="/foo",method="GET"} 1
    http_request_duration_seconds_bucket{le="+Inf",api="/foo",method="GET"} 1

    http_request_duration_seconds_sum{api="/foo",method="GET"} 0.8

    http_request_duration_seconds_count{api="/foo",method="GET"} 1
    ```

    1-4 行的几个 `<metricName>_bucket` 中记录了落到各个 bucket 的数据量；`le` (less than or equal to) 表示不大于该值的 observe 次数。第 6 行的 `<metricName>_sum` 记录了所有 observe 的数据和，第 8 行的 `<metricName>_count` 记录了 observe 的总量。

    在此基础上，如果有 `#!python http_request_duration_seconds(api="/foo", method="GET").observe(0.4)`，那么数据会变为：

    ```linenums="1"
    http_request_duration_seconds_bucket{le="0.1",api="/foo",method="GET"} 0
    http_request_duration_seconds_bucket{le="0.5",api="/foo",method="GET"} 1
    http_request_duration_seconds_bucket{le="1.0",api="/foo",method="GET"} 2
    http_request_duration_seconds_bucket{le="+Inf",api="/foo",method="GET"} 2

    http_request_duration_seconds_sum{api="/foo",method="GET"} 1.2

    http_request_duration_seconds_count{api="/foo",method="GET"} 2
    ```

    如果还有 `#!python http_request_duration_seconds(api="/bar", method="GET").observe(1.2)`，那么会新增与上述类似的 6 个 time series。

??? example "Summary"
    与 `Histogram` 类似，`Summary` 也会创建一系列不同的 metric 来记录观察得到的数据；但它会记录不同的分位数 (quantile) 而非预先定义的 bucket。例如：

    ```
    http_request_duration_seconds{quantile="0.5"} 0.232227334
    http_request_duration_seconds{quantile="0.90"} 0.821139321
    http_request_duration_seconds{quantile="0.95"} 1.528948804
    http_request_duration_seconds{quantile="0.99"} 2.829188272
    http_request_duration_seconds{quantile="1"} 34.283829292

    http_request_duration_seconds_sum 8953.332
    http_request_duration_seconds_count 27892
    ```

    由于 `Summary` 需要使用流式算法来计算分位数，因此它的计算量会比 `Histogram` 大得多。

关于如何选择 `Histogram` 和 `Summary`，请参考 [Histograms and Summaries | Prometheus](https://prometheus.io/docs/practices/histograms/)。

## 2 Querying | 查询

!!! note "Thanos"
    本文主要介绍 Prometheus；但在实际使用中，可能会使用 [Thanos](https://thanos.io/) 来扩展 Prometheus 的功能。Thanos 提供了一些额外的功能，例如长期存储、查询优化、数据压缩等。

    例如，每个集群的 Prometheus 负责收集所在集群各服务的 metrics 并提供一定的查询能力；而 Thanos 以 sidecar 的形式整合各个集群的 Prometheus 采集的指标并长期保存，**并同样适用 PromQL 查询**。

### 2.1 Data Types | 数据类型

在 PromQL 中，大多数表达式归属于以下 3 种类型之一：

- Scalar：单个浮点数数值；
- Instant vector：一组时间序列，每个时间序列包含它在特定时间点的值，例如 Query `http_requests_total` 或者 `http_requests_total{method="GET"}` 返回在 query timestamp 这个时间点中每个时间序列的值；
    - 请注意：即使 `http_requests_total{method="GET"}` 得到的 instant vector 只包含 1 个元素，它也不能被视为 scalar。
- Range vector：一组时间序列，每个时间序列包含它在一个时间段内每个时间点的值，例如 `http_requests_total[5m]` 返回在 5 分钟内每个时间序列的值。

如果我们用 Python 做类比的话，上面的三个类型可以以如下方式表示：

```Python linenums="1" hl_lines="1 5 10"
type Scalar = float

type Labels = Dict[str, str]
type LabeledScalar = Tuple[Labels, Scalar]
type InstantVector = List[LabeledScalar]

type TimeStamp = int
type TimeSeries = List[Tuple[TimeStamp, Scalar]]
type LabeledTimeSeries = Tuple[Labels, TimeSeries]
type RangeVector = List[LabeledTimeSeries]
```

### 2.2 Time Series Selectors | 时间序列选择器

以下表达式具有类型 `InstantVector`：

- `http_requests_total`
- `http_requests_total{method="GET"}`
    - 对于没有被 matchers 提及的 label，则任何值全部接受。
    - 对于同一个 label，可以有多个 matchers，它们之间是 AND 关系。
- `{__name__="http_requests_total", method="GET"}`
    - `__name__` 是一个特殊的 label，用于匹配指标名。
    - 由此可见，**selector 中不必包含 metric_name**。
    - 因此，该表达式和前一个例子等价。
- `http_requests_total{api=~"v1.*", code!~"20[0-9]", method!="POST"}`
    - `=` 表示精确匹配，`=~` 表示正则匹配，`!=` 和 `!~` 表示不匹配，分别适用于字符串和正则表达式。
    - 正则表达式是 fully anchored 的，即 `api=~"v1.*"` 等价于 `api=~"^v1.*$"`。
- `http_requests_total offset 5m` / `http_requests_total{method="GET"} offset 5m`
    - `offset` 用于指定时间偏移量，例如 `offset 5m` 表示向前偏移 5 分钟。
- `http_requests_total @ 1609746000` / `http_requests_total{method="GET"} @ 1609746000`
    - `@` 用于指定时间戳，例如 `@ 1609746000` 表示在时间戳 1609746000 的时候的值。

以下表达式具有类型 `RangeVector`：

- `http_requests_total[5m]`
- `http_requests_total{method="GET"}[5m]`
- `http_requests_total{method="GET"}[5m] offset 5m`
- `http_requests_total{method="GET"}[5m] @ 1609746000`

### 2.3 Operators | 运算符

#### 2.3.1 算术二元运算符

包括 `+`, `-`, `*`, `/`, `%` (取模), `^` (幂)。其行为是：

- `Scalar <op> Scalar`：和正常算术一致；
- `InstantVector <op> Scalar`：返回对 InstantVector 中的每个元素应用 `<op>` 的结果；**metric_name (`__name__`) 被删除**；
- `InstantVector1 <op> InstantVector2`：对于 InstantVector1 中的每个元素，在 InstantVector2 中找到标签完全相同的元素，将其应用 `<op>` 的结果返回，**metric_name (`__name__`) 被删除**；如果没找到，则结果中不包含该 label；
- 不接受 RangeVector。

#### 2.3.2 比较二元运算符

包括 `==`, `!=`, `>`, `<`, `>=`, `<=`。其行为是：

- `bool(Scalar <op> Scalar)`：返回 0 或者 1；
- `InstantVector <op> Scalar`：返回结果中比较结果为真的元素保留，其余删除；
- `bool(InstantVector <op> Scalar)`：返回结果中将比较结果为真的元素置为 1，其余置为 0；**metric_name (`__name__`) 被删除**；
- `InstantVector1 <op> InstantVector2`：InstantVector2 作为过滤器：InstantVector1 在其中找不到标签完全相同的元素，或者对应元素比较结果为假的，将其删除；
- `bool(InstantVector1 <op> InstantVector2)`：类似前一条：保留的元素置为 1，其余置为 0；**metric_name (`__name__`) 被删除**；
- 不接受 RangeVector。

#### 2.3.3 逻辑（集合）二元运算符

包括 `and`, `or`, `unless`，只在 `InstantVector` 之间定义。其行为是：

- `iv1 and iv2` (intersection)：返回 iv1 中的那些在 iv2 中具有完全相同标签的元素；
- `iv1 or iv2` (union)：返回 iv1 的所有元素，以及 iv2 中那些在 iv1 中没有完全相同标签集的元素；
- `iv1 unless iv2` (complement)：返回 iv1 中的那些在 iv2 中没有完全相同标签的元素。

#### 2.3.4 Vector Matching

上面三节中描述的 `InstantVector` 之间的操作，都试图从右边的 `InstantVector` 中为左边的 `InstantVector` 中的每个元素找到标签匹配的元素。这个「匹配」过程也是可以控制的：

- `iv1 <bin-op> on(<label-list>) iv2` 将纳入考虑的标签限制在 `<label-list>` 中，其他标签全部忽略；
- `iv1 <bin-op> ignoring(<label-list>) iv2` 将忽略 `<label-list>` 中的标签，其他标签全部纳入考虑。

下图展示了一个例子：

![](assets/2024-07-09-19-48-45.png)

另外在算术和比较运算符中，还可以通过 `group_left` 和 `group_right` 实现一对多和多对一的匹配。本文暂且略去，参见 [文档](https://prometheus.io/docs/prometheus/latest/querying/operators/#many-to-one-and-one-to-many-vector-matches)。

#### 2.3.5 聚合运算符

### 2.4 Functions | 函数

### 2.5 API
