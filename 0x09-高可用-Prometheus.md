## 9. 高可用 Prometheus

鉴于自己的开发机器性能有限，也没有那么大数据量的业务场景。所以实际上高可用这部分我也没真正折腾过（诚实不做作！），不过也聊一聊社区一些现有的方案吧。

### 9.1 数据高可用

首先我们知道，Prometheus 本身的数据是不依赖与网络存储的，也就是说时序数据是存放在「本地的」，所以对于超大量数据来说瓶颈就很明显了。

但是不用慌，Prometheus 团队也考虑到这个事情了，Prometheus 提供了两种接口，remote-read 和 remote-write，意味着可以对接第三方的时序数据库，将旧数据堆放到其他地方。查询需要的时候再从第三方数据库读取回来。

***remote write***

![remote write](./images/remote-write.png)

***remote read***

![remote read](./images/remote-read.png)

目前已经支持的 remote-* 接口的实现方案如下。

* [AppOptics](https://github.com/solarwinds/prometheus2appoptics): write
* [Azure Data Explorer](https://github.com/cosh/PrometheusToAdx): read and write
* [Azure Event Hubs](https://github.com/bryanklewis/prometheus-eventhubs-adapter): write
* [Chronix](https://github.com/ChronixDB/chronix.ingester): write
* [Cortex](https://github.com/cortexproject/cortex): read and write
* [CrateDB](https://github.com/crate/crate_adapter): read and write
* [Elasticsearch](https://github.com/infonova/prometheusbeat): write
* [Gnocchi](https://gnocchi.xyz/prometheus.html): write
* [Graphite](https://github.com/prometheus/prometheus/tree/master/documentation/examples/remote_storage/remote_storage_adapter): write
* [InfluxDB](https://docs.influxdata.com/influxdb/latest/supported_protocols/prometheus): read and write
* [IRONdb](https://github.com/circonus-labs/irondb-prometheus-adapter): read and write
* [Kafka](https://github.com/Telefonica/prometheus-kafka-adapter): write
* [M3DB](https://m3db.github.io/m3/integrations/prometheus): read and write
* [OpenTSDB](https://github.com/prometheus/prometheus/tree/master/documentation/examples/remote_storage/remote_storage_adapter): write
* [PostgreSQL/TimescaleDB](https://github.com/timescale/prometheus-postgresql-adapter): read and write
* [QuasarDB](https://doc.quasardb.net/master/user-guide/integration/prometheus.html): read and write
* [SignalFx](https://github.com/signalfx/metricproxy#prometheus): write
* [Splunk](https://github.com/kebe7jun/ropee): read and write
* [TiKV](https://github.com/bragfoo/TiPrometheus): read and write
* [Thanos](https://github.com/thanos-io/thanos): write
* [VictoriaMetrics](https://github.com/VictoriaMetrics/VictoriaMetrics): write
* [Wavefront](https://github.com/wavefrontHQ/prometheus-storage-adapter): write

### 9.2 Prometheus 服务高可用

Prometheus server 是没有原生的高可用方案的，不过它有一种联邦机制（Federation）。也就是说，可以把 promethues 本身的数据上报给上层的 promethues 实例，是一个层级结构。

![prometheus-federation](prometheus-federation.png)

联邦机制解决了什么问题呢。比如，你可以在一个数据中心部署一个 Promethues server，然后各个业务部门部署自己单独的 promethues server，然后数据统一上报给数据中心，由数据中心负责对数据进行聚合。

然后问题又来了，如果我一个业务部门只部署了一个 promethues server 用于抓取数据，这样就存在单点问题了，万一这个 server 挂掉了，那不就会丢失重要的监控样本数据了。所以为了保险起见，是否要再部署一个副本，抓取相同的数据，做冗余备份呢？再继续思考，那我们是不是还得再部署一个 promethues 来监控这两个 promethues 的运行情况呢？🐶

仁者见仁，智者见智。

### 9.3 Alertmanager 高可用

Alertmanager 组件本身提供了高可用方案，所以在 prometheus-operator 中，只需要提高 replica 数量即可（舒服！）。

### 敲黑板！划重点！

Prometheus 的折腾之旅到这里就告一段落啦，通过这段时间的捣鼓也学习到了不少东西。最后还是一句话，多翻文档，多动手。

**:wq**
