# 九格看板查询与边界

来源：实际导出的 dashboard.json，Grafana v13.3.0，dashboard.grafana.app/v2 格式。
按 L-08 第29页分为业务、服务 RED、资源 USE。业务缺失项保留说明，不视为已完成测量。

## 业务 1｜开放后十分钟完成率（未采集）

原讲义业务指标，示例服务缺少目标学生数和去重完成计数。

### 未采集，不能填 0 或 100%

完成率 = 去重完成选课人数 / 目标选课人数。

示例无学生总量、开放时刻及去重完成指标，无法计算。讲义目标线为 95%。需补业务埋点后才能绘制，不能拿 HTTP 请求成功率替代。

## 业务 2｜提交成功率（HTTP 口径）

近似展示：2xx / 非4xx已完成HTTP提交；不等于端到端业务成功率。仅在业务拒绝确实用4xx表示时适用。无写负载时无数据，不补零。

A：提交HTTP成功比例

```promql
(sum(rate(http_requests_total{job="enrolld",route="/api/enroll",code=~"2.."}[1m])) or vector(0)) / sum(rate(http_requests_total{job="enrolld",route="/api/enroll",code!~"4.."}[1m]))
```

## 业务 3｜在线人数（未采集）

无登录会话和去重用户指标；在途请求、TCP连接数均不能充当在线人数。

### 未采集

示例没有登录会话或在线用户去重指标。

不能用 inflight、连接池占用、TCP连接数冒充在线人数。需补在线会话指标及其去重/超时定义。

## 服务 RED 1｜按接口 QPS / 5xx比例

读写接口分别展示。QPS与比例使用独立单位轴；仅表示服务端看到的请求。

A：{{route}} QPS

```promql
sum by (route) (rate(http_requests_total{job="enrolld",route=~"/api/courses/.*|/api/enroll"}[1m]))
```

B：{{route}} 5xx比例

```promql
(sum by (route) (rate(http_requests_total{job="enrolld",route=~"/api/courses/.*|/api/enroll",code=~"5.."}[1m])) or (0 * sum by (route) (rate(http_requests_total{job="enrolld",route=~"/api/courses/.*|/api/enroll"}[1m])))) / sum by (route) (rate(http_requests_total{job="enrolld",route=~"/api/courses/.*|/api/enroll"}[1m]))
```

## 服务 RED 2｜按接口 p50 / p99

直方图插值，秒转毫秒。先按route、le聚合。

A：{{route}} p50

```promql
1000 * histogram_quantile(0.5, sum by (route,le) (rate(http_request_duration_seconds_bucket{job="enrolld",route=~"/api/courses/.*|/api/enroll"}[1m])))
```

B：{{route}} p99

```promql
1000 * histogram_quantile(0.99, sum by (route,le) (rate(http_request_duration_seconds_bucket{job="enrolld",route=~"/api/courses/.*|/api/enroll"}[1m])))
```

## 服务 RED 3｜在途与连接池等待

无Kafka组件，Kafka lag不适用。请求数和等待时长单位不同。

A：{{route}} 在途请求

```promql
http_inflight_requests{job="enrolld",route=~"/api/courses/.*|/api/enroll"}
```

B：连接池等待p99

```promql
1000 * histogram_quantile(0.99,sum by(le)(rate(db_pool_wait_seconds_bucket{job="enrolld"}[1m])))
```

## 资源 USE 1｜CPU与内存

服务核1；CPU忙碌排除idle/iowait。CPU队列为整机。

A：CPU1忙碌比例

```promql
sum(rate(node_cpu_seconds_total{job="node",cpu="1",mode!~"idle|iowait"}[1m]))
```

B：整机运行队列

```promql
node_procs_running{job="node"}
```

C：整机steal比例

```promql
avg(rate(node_cpu_seconds_total{job="node",mode="steal"}[1m]))
```

D：Available MiB

```promql
node_memory_MemAvailable_bytes{job="node"}/1048576
```

## 资源 USE 2｜vda磁盘等待与利用率

使用enrolld数据盘vda替代讲义MySQL盘；没有MySQL，无buffer pool命中率。无I/O时await可能NaN。

A：写await ms

```promql
1000*rate(node_disk_write_time_seconds_total{job="node",device="vda"}[1m])/rate(node_disk_writes_completed_total{job="node",device="vda"}[1m])
```

B：磁盘util

```promql
rate(node_disk_io_time_seconds_total{job="node",device="vda"}[1m])
```

C：磁盘平均队列

```promql
rate(node_disk_io_time_weighted_seconds_total{job="node",device="vda"}[1m])
```

## 资源 USE 3｜网卡流量与TCP连接

读写必做经lo，不经过EIP，不能推断公网带宽上限。TCP为整机计数；listen overflow仅在指标存在时显示，缺失不补0。

A：{{device}} TX bit/s

```promql
8*rate(node_network_transmit_bytes_total{job="node"}[1m])
```

B：{{device}} RX bit/s

```promql
8*rate(node_network_receive_bytes_total{job="node"}[1m])
```

C：整机TCP Established

```promql
node_netstat_Tcp_CurrEstab{job="node"}
```

D：整机ListenOverflow/s

```promql
rate(node_netstat_TcpExt_ListenOverflows{job="node"}[1m])
```

E：整机conntrack entries

```promql
node_nf_conntrack_entries{job="node"}
```
