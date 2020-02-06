## 4. Prometheus 理论（入门篇）

<p align="center"><img src="./images/prometheus-logo.png" width="400px"></p>

### 4.1 Prometheus 是什么？

[Prometheus](https://prometheus.io) 是由 SoundCloud 公司开源和维护的监控和告警的解决方案。项目始于 2012 年，目前已有多家公司将 Prometheus 投入于生产环境中。Prometheus 是继 Kubernetes 之后第二个从 [CNCF](https://www.cncf.io/) 毕业的项目，在这个 CloudNative 盛行的时代，Prometheus 已经成为最热门的监控方案。Prometheus 有着良好的社区氛围，在社区的驱动下，项目正在如火如荼的开发迭代中。

作为新一代的监控框架，Prometheus 具有以下特性：
 
* 强大的多维度数据模型，时间序列数据通过 metric 名和键值对来区分，并且所有的 metrics 都可以设置任意的多维标签。
* 灵活的查询语句（PromQL），在同一个查询语句，可以对多个 metrics 进行乘法、加法、连接、取分数位等操作。
* 部署方便，Prometheus server 是一个单独的二进制文件，可直接在本地工作，不依赖于分布式存储。
* 采样高效，一个 Prometheus server 可以处理数百万的 metrics。
* 解耦客户端和服务端，使用 pull 模式采集时间序列数据，由服务端主动配置和抓取客户端 metrics。不过也可以采用 push gateway 的方式把 metrics 数据推送至服务端。
* 支持多种服务发现机制。
* 丰富的可视化图形界面。

prometheus 组件

* [Prometheus server](https://github.com/prometheus/prometheus): Prometheus 服务端，负责采集来自客户端的时间序列数据。
* [client libraries](https://prometheus.io/docs/instrumenting/clientlibs/): 不同语言的 SDK 客户端。
* [push gateway](https://github.com/prometheus/pushgateway): 主动推送网关，用于采集短时间或瞬时任务，这种情况在一个 Prometheus 采集周期可能会被忽略，所以需要自己主动推送数据。
* [alertmanger](https://github.com/prometheus/alertmanager): 处理告警信息，将警告通知给指定的接受者。
* [exporters](https://prometheus.io/docs/instrumenting/exporters/): exporters 是 Prometheus 的一类采集数据的客户端组件，比如数据库数据采集，主机节点数据采集等等。

***prometheus 架构***

![Prometheus 架构图](./images/prometheus-architecture.png)

### 4.2 Prometheus 目标

**prometheus 适合干什么？**

Prometheus 天生就是为记录纯数字时间序列数据而准备的。它既适合监控主机级别的数据，也适合于高度动态的面向服务的数据的监控。在微服务领域对多维数据收集和查询的支持是非常强大的特性。Prometheus 的设计旨在提高可靠性，以便我们能快速地发现和排查系统的问题。每个 Prometheus 服务都是独立的，不依赖于网络存储或其他远程服务。

**prometheus 不适合干什么？**

Prometheus 强调可靠性，即使在故障情况下也始终可以查看有关系统的可用统计信息。但是如果需要 100％ 的准确性（例如审计系统），则 Prometheus 并不是一个好的选择，因为其所收集的数据不够详细和完整（数据压缩也会丢失精度）。在这种情况下，使用传统的关系型数据库会是一个更好的选择。

### 4.3 prometheus 数据类型

Prometheus 有四种基本的数据类型，Counter、Gauge、Histogram、Summary。

#### 4.3.1 Counter（计数器）

Counter 单调递增，违反单调性时重置为 0。可以用于统计某些事件出现的次数，或者服务的 uptime。指标名称一般以 `_total` 作为后缀。

#### 4.3.2 Gauge（仪表盘）

Gauge 记录瞬时值，可以用于记录系统当下时刻的状态，比如 CPU 使用率，使用内存大小，网络 IO 情况。

#### 4.3.3 Histogram（直方图）

Histogram 由 `<basename>_bucket{le="<upper inclusive bound>"}`，`<basename>_bucket{le="+Inf"}`, `<basename>_sum`，`<basename>_count` 组成，主要用于表示一段时间范围内对数据进行采样（通常是请求持续时间或响应大小），并能够对其指定区间以及总数进行统计，通常它采集的数据展示为直方图。例如 Prometheus server 中 prometheus_local_storage_series_chunks_persisted, 表示 Prometheus 中每个时序需要存储的 chunks 数量，我们可以用它计算待持久化的数据的分位数。（主要计算消耗在服务端）

#### 4.3.4 Summary（摘要）

Summary 和 Histogram 类似，由 `<basename>{quantile="<φ>"}`，`<basename>_sum`，`<basename>_count` 组成，主要用于表示一段时间内数据采样结果（通常是请求持续时间或响应大小），它直接存储了 quantile 数据，而不是根据统计区间计算出来的（主要计算消耗在客户端）。例如 Prometheus server 中 prometheus_target_interval_length_seconds。

关于 Histogram 和 Summary 的详细对比，可参考文档：[practices/histograms](https://prometheus.io/docs/practices/histograms/) 。
