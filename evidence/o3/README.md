# O3 证据索引

- 原始过载实验：../../runs/o3-read-overload-r2/，2026-10-30 01:20:15～01:23:15（Asia/Shanghai）。
- 规则：../../deploy/rules.yml；引用配置：../../deploy/prometheus.yml；规则健康状态：rules-loaded.json。
- 原始轮询日志：alerts-watch-20261030-011959.log，包含 pending、firing、恢复；轮询间隔约5秒。
- 首次轮询到 firing：2026-10-30T01:20:53+08:00；快照 alerts-firing-from-log.json 逐字提取该时刻原始 JSON 响应。
- 首次轮询到恢复：2026-10-30T01:24:23+08:00；快照 alerts-recovered-from-log.json 逐字提取该时刻原始 JSON 响应。
- 原操作另存的 alerts-firing.json 是空 alerts 列表，未纳入本次归档；firing 证据以上述原始轮询日志和逐字提取的快照为准。
- dashboard.json 为用户实际从 Grafana 导出的新版 v2 看板。dashboard-queries.md 从该文件提取查询。
- dashboard-business-red.png 和 dashboard-use.png 为用户原截图的副本，合起来覆盖三行九格；图表时间范围约01:17～01:30。
- 业务完成率和在线人数缺少埋点；提交成功率在所选时间内没有写请求，因此显示 No data。不能据此声称已测出这些业务指标。
- 此处是本地整理材料。停止监控后看板JSON和截图仍保留，但JSON不包含全部历史时序数据；复现曲线需Prometheus历史数据或重新测量。
