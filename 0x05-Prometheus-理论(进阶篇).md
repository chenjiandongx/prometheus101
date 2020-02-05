## 5. Prometheus 理论（进阶篇）

### 5.1 PromQL

#### 5.1.1 数据格式

时间序列是由样本构成的，每个样本包括：

* float64 精度的数值
* 毫秒精度的时间戳
* 标签（label）

Prometheus label 可能包含 ASCII 字母，数字和下划线。label 必须与正则表达式 `[a-zA-Z_:][a-zA-Z0-9_:]*.` 相匹配。以 `__` 作为前缀的 label 保留供内部使用，如 Kubernetes_sd 中的 `__meta_*`。label 值可以包含任何 Unicode 字符且空 label 等同于该 label 不存在。

样本形式如下
```
<metric name>{<label name>=<label value>, ...}
```

#### 5.1.2 PromQL 标签匹配四种操作符

* **=**: label 完全匹配。
* **!=**: label 非完全匹配。
* **=~**: label 正则匹配。
* **!~**: label 非正则匹配。

#### 5.1.3 PromQL 四种数据类型

**Instant vector（即时向量）**：一组时间序列，拥有共同的时间戳，每个时间序列中都包含一个样本。

下面表达式描述的是 environment 值可能为 `staging|testing|development` 三种中的一种，且 method 不为 `GET` 的 HTTP 请求的数据。其实也挺好的理解 😎。
```
http_requests_total{environment=~"staging|testing|development",method!="GET"}

# 等价于该表达式，__name__ 是指 metrics_name
{__name__="http_requests_total",environment=~"staging|testing|development",method!="GET"}
```

即时向量之间还可以进行多种算术操作，算术操作符包括。
```
+ (加法)
- (减法)
* (乘法)
/ (除法)
% (求余)
^ (幂运算)
```

即时向量之间同时也还可以进行多种布尔操作，布尔操作符包括。
```
== (相等)
!= (不相等)
> (大于)
< (小于)
>= (大于等于)
<= (小于等于)
```
**Range vector（范围向量）**：一组时间序列，其中每个时间序列都包含一系列时间范围内的数据点。

如果取的是一段时间范围内的样本数据，则需要使用区间向量表达式。区间向量表达式和瞬时向量表达式之间的差异在于在区间向量表达式中我们需要定义时间选择的范围，时间范围通过时间范围选择器 `[]` 进行定义。
```
# 该样本最近五分钟内的数据
http_request_total{}[5m]
```

Prometheus 支持多种时间维度
```
s - 秒
m - 分钟
h - 小时
d - 天
w - 周
y - 年
```

**Scalar（标量）**：一个简单的浮点值。

只有数字，没有时序性质。

**String（字符串）**：一个简单的字符串，目前暂未使用。

#### 5.1.4 PromQL Function

PromQL 最具威力的还是其提供的多种使用函数，函数列表请参考 [querying/functions](https://prometheus.io/docs/prometheus/latest/querying/functions/)。

这里就介绍最常用的几个函数，剩下的还是需要自己多阅读文档，用到时再查询即可。
```
# 计算最近 5 分钟平均请求速率。
rate(http_requests_total[5m])

# 计算所有 HTTP 请求访问总量。
sum(http_requests_total)

# 计算访问量最大的 5 个请求
topk(5, http_requests_total)

# 计算最近 5 分钟请求量新增了多少。
increase(http_requests_total{job="api-server"}[5m])

# 计算最近 5 分钟请求量的增长率
increase(http_requests_total{job="api-server"}[5m]) / 300

# 计算当前样本数据中的中位数
quantile(0.5, http_requests_total)

# 计算以最近 1 个小时的磁盘空间消耗速度，是否会在 2 个小时内将磁盘空间消耗殆尽。
predict_linear(node_filesystem_free{job="node1"}[1h], 2 * 3600) < 0
```

### 5.2 Prometheus 配置详解

Prometheus 的配置项是整个组件的核心，配置是声明式的，这点我觉得是蛮棒的，降低使用门槛，如果想熟练的使用 Prometheus，还是需要把配置文档好好阅读几遍，Prometheus 的官方文档质量还是蛮高的。最基础的配置如下：
```yaml
global:
  # How frequently to scrape targets by default.
  [ scrape_interval: <duration> | default = 1m ]

  # How long until a scrape request times out.
  [ scrape_timeout: <duration> | default = 10s ]

  # How frequently to evaluate rules.
  [ evaluation_interval: <duration> | default = 1m ]

  # The labels to add to any time series or alerts when communicating with
  # external systems (federation, remote storage, Alertmanager).
  external_labels:
    [ <labelname>: <labelvalue> ... ]

# Rule files specifies a list of globs. Rules and alerts are read from
# all matching files.
rule_files:
  [ - <filepath_glob> ... ]

# A list of scrape configurations.
scrape_configs:
  [ - <scrape_config> ... ]

# Alerting specifies settings related to the Alertmanager.
alerting:
  alert_relabel_configs:
    [ - <relabel_config> ... ]
  alertmanagers:
    [ - <alertmanager_config> ... ]

# Settings related to the remote write feature.
remote_write:
  [ - <remote_write> ... ]

# Settings related to the remote read feature.
remote_read:
  [ - <remote_read> ... ]
```

#### 5.2.1 scrape_configs

scrape_configs 是 Promethues 最为重要的配置之一，指定了 promethues 实例如何抓取 metrics。
```yaml
# scrape_configs

# job 是 promethues 中最基本的调度单位，一个 job 可能会拥有多个 instances，就像一个服务可能会拥有多个实例一样。
job_name: <job_name>

# 抓取时间间隔。
[ scrape_interval: <duration> | default = <global_config.scrape_interval> ]

# 抓取超时时间。
[ scrape_timeout: <duration> | default = <global_config.scrape_timeout> ]

# 默认的 web path 是 /metrics，现在对整个社区来说也是一个约定俗成的东西，基本不会有变动。
[ metrics_path: <path> | default = /metrics ]

# 协议格式 http/https？
[ scheme: <scheme> | default = http ]

# 查询还可以带参数，但一般也没这个必要。
params:
  [ <string>: [<string>, ...] ]

# tls 证书配置，不过目前如果使用的服务都是跑在 kubernetes 集群内，且无对外部暴露的话，也不用考虑 tls。
tls_config:
  [ <tls_config> ]

# 代理配置。
[ proxy_url: <string> ]

# 下面的都是服务发现的配置，promethues 原生地提供了多种服务发现的方案。
# 目的是为了结合服务发现实现 promethues 的热更新，不必再手动地更新配置。
# 使用 prometheus-operator 本质上是与 kubernetes_sd_configs 相结合，只是 operator 帮我们屏蔽了这些复杂性。
# 对于其他服务发现体系的，可以到官网上查看具体的配置项。
# https://prometheus.io/docs/prometheus/latest/configuration/configuration/#scrape_config

azure_sd_configs:
  [ - <azure_sd_config> ... ]

consul_sd_configs:
  [ - <consul_sd_config> ... ]

dns_sd_configs:
  [ - <dns_sd_config> ... ]

ec2_sd_configs:
  [ - <ec2_sd_config> ... ]

openstack_sd_configs:
  [ - <openstack_sd_config> ... ]

file_sd_configs:
  [ - <file_sd_config> ... ]

gce_sd_configs:
  [ - <gce_sd_config> ... ]

# kubernetes_sd_configs 实现的思路其实就是通过 Kubernetes REST API 获取对应的资源的信息。
# 包括 node/service/pod/endpoints/ingress
# kubernetes 会给每种资源注入自己的信息，当然资源本身也可以自定义一些附带信息，如 labels/annotation 等。
# 获取到的数据都以 __meta 作为前缀，在 promethues 中，双下划线为前缀的 metrics 是不会被暴露到外部的。
# 所以可用 relabels 来将 __meta_* metrics 转换为自己想要的形式。
kubernetes_sd_configs:
  [ - <kubernetes_sd_config> ... ]

marathon_sd_configs:
  [ - <marathon_sd_config> ... ]

nerve_sd_configs:
  [ - <nerve_sd_config> ... ]

serverset_sd_configs:
  [ - <serverset_sd_config> ... ]

triton_sd_configs:
  [ - <triton_sd_config> ... ]

# 如果你是单机使用并且没有任何服务发现体系的话，可以用 static_configs
# <static_config>
# targets:
# targets 就是上面指的一个 job 可能会有多个 instances 的情况，列表类型。
#  [ - '<host>' ]
# Labels assigned to all metrics scraped from the targets.
# labels:
# [ <labelname>: <labelvalue> ... ]
static_configs:
  [ - <static_config> ... ]

# relabel 涉及到几种 action，action 指的是你可以根据正则捕获结果对 label 进行何种操作，是丢弃呢，还是改写呢？
# 具体内容可以参考下面这篇博客
# https://www.li-rui.top/2019/04/16/monitor/Prometheus%E4%B8%ADrelabel_configs%E7%9A%84%E4%BD%BF%E7%94%A8/
# 
# actions 类型如下
# 
# replace       根据正则匹配标签的值进行替换标签
# keep          根据正则匹配标签的值保留数据采集源
# dro           根据正则匹配标签的值剔除数据采集源
# hashmod       hash 模式
# labelmap      根据正则匹配标签的名称进行映射
# labeldrop     根据正则匹配标签的名称剔除标签
# labelkeep     根据正则匹配标签的名称保留标签
relabel_configs:
  [ - <relabel_config> ... ]
```

#### 5.2.2 alertingRule

采集了 metrics 可以被告警系统使用，所以我们需要根据手上掌握的数据来定义告警规则。首先来看看官方给出的一个基础示例。
```yaml
groups:
- name: example
  rules:
  - alert: HighRequestLatency
    expr: job:request_latency_seconds:mean5m{job="myjob"} > 0.5
    for: 10m
    labels:
      severity: page
    annotations:
      summary: High request latency
```

规则模板如下
```yaml
# The name of the alert. Must be a valid metric name.
# 告警规则名字。
alert: <string>

# The PromQL expression to evaluate. Every evaluation cycle this is
# evaluated at the current time, and all resultant time series become
# pending/firing alerts.
# promQL 表达式，需要符合 promQL 语法
expr: <string>

# Alerts are considered firing once they have been returned for this long.
# Alerts which have not yet fired for long enough are considered pending.
# 规则匹配的持续时间
[ for: <duration> | default = 0s ]

# Labels to add or overwrite for each alert.
# labels 用于提供附带信息，在 alertmanagers 中可以用到。
labels:
  [ <labelname>: <tmpl_string> ]

# Annotations to add to each alert.
# Annotations 用于提供附带信息，在 alertmanagers 中可以用到。
annotations:
  [ <labelname>: <tmpl_string> ]
```

#### 5.2.3 recordingRule

Prometheus 提供一种记录规则（Recording Rule） 来支持后台计算的方式，可以实现对复杂查询的 PromQL 语句的性能优化，提高查询效率。记录规则的基本思想是，它允许我们基于其他时间序列创建自定义的 meta-time 序列。

在 Prometheus Operator 中已经有了大量此类规则，比如：
```yaml
groups:
  - name: k8s.rules
    rules:
    - expr: |
        sum(rate(container_cpu_usage_seconds_total{image!="", container!=""}[5m])) by (namespace)
      record: namespace:container_cpu_usage_seconds_total:sum_rate
    - expr: |
        sum(container_memory_usage_bytes{image!="", container!=""}) by (namespace)
      record: namespace:container_memory_usage_bytes:sum
```

上面的这两个规则就完全可以执行上面我们的查询，它们会连续执行并以很小的时间序列将结果存储起来。
```
sum(rate(container_cpu_usage_seconds_total{job="kubelet", image!="", container_name!=""}[5m])) by (namespace)
```
将以预定义的时间间隔进行评估，并存储为新的指标，新指标与内存查询相同，效率更高
```
namespace:container_cpu_usage_seconds_total:sum_rate
```

同样的，我们先来看官方给出的基础示例。
```yaml
groups:
  - name: example
    rules:
    - record: job:http_inprogress_requests:sum
      expr: sum(http_inprogress_requests) by (job)
```

rule 规则模板如下
```yaml
# The name of the time series to output to. Must be a valid metric name.
record: <string>

# The PromQL expression to evaluate. Every evaluation cycle this is
# evaluated at the current time, and the result recorded as a new set of
# time series with the metric name as given by 'record'.
# promQL 表达式，需要符合 promQL 语法
expr: <string>

# Labels to add or overwrite before storing the result.
labels:
  [ <labelname>: <labelvalue> ]
```

推荐一篇博客：[Prometheus 记录规则的使用](https://www.qikqiak.com/post/recording-rules-on-prometheus/)
