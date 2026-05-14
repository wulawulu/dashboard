# 数据中台 ECS Dashboard 标签与面板口径

本文档基于 `参考/数据中台-ECS监控-1778728407266.json` 梳理最终 ECS 仪表板使用的标签、变量、指标和面板命名。当前可直接使用的仪表板文件为 `grafana/dashboards/local-node-exporter.json`。

## 标签口径

最终仪表板优先使用内网参考看板中的标签口径：

| 标签 | 用途 | 当前 mock 取值说明 |
| --- | --- | --- |
| `vendor` | 云厂商筛选 | `内网云` |
| `account` | 云账户筛选 | `数据中台生产账户` |
| `group` | ECS 分组筛选 | `数据中台` |
| `name` | ECS 展示名称 | 业务可读名称，例如 `数据中台调度数据接入生产ECS01` |
| `instance` | ECS IP | 直接使用 IP，例如 `10.10.20.11` |
| `iid` | 云实例 ID | 例如 `i-mock-0001` |
| `cservice` | 云服务类型 | `ecs` |
| `region` | 区域 | `内网` |
| `exp` | 到期或生命周期信息 | 当前 mock 为 `长期` |

真实接入时必须保证 `vendor/account/group/name/instance/iid` 六个标签稳定存在。这样可以直接复用内网参考看板中的变量链路和本项目最终仪表板。

## 变量设计

最终仪表板保留参考看板中的核心变量：

| 变量 | 展示名 | 来源 |
| --- | --- | --- |
| `$vendor` | 云厂商 | `label_values(node_uname_info, vendor)` |
| `$account` | 账户 | `label_values(node_uname_info{vendor=~"$vendor"}, account)` |
| `$group` | 分组 | `label_values({group=~"数据中台"}, group)` |
| `$name` | 名称 | `label_values(node_uname_info{vendor=~"$vendor",account=~"$account",group=~"$group"}, name)` |
| `$sname` | 查询 | 文本框，用于名称模糊搜索 |
| `$interval` | 间隔 | 默认 `3m` |
| `$instance` | IP | `label_values(node_uname_info{...}, instance)` |
| `$device` | 网卡 | 排除 `tap/veth/br/docker/virbr/lo/cni` 后的网卡 |
| `$show_name` | 展示使用的名称 | 隐藏变量，用于标题展示 |
| `$iid` | 实例 ID | 隐藏变量，用于标题展示 |

## 面板分区

按领导意见，最终仪表板把内网参考看板中的资源总览、容量、单机、磁盘和网络内容合并进来，但将排障明细默认折叠。

| 分区 | 默认状态 | 用途 |
| --- | --- | --- |
| 资源总览 | 展开 | 第一屏判断 ECS 是否在线、是否存在 CPU/内存/磁盘/网络风险 |
| 容量分析 | 折叠 | 最近 7 天 P99 资源水位，用于容量评估 |
| 单机排障明细 | 折叠 | 选择 `$instance` 后查看单台 ECS 的 CPU、内存、分区、负载、进程和 FD |
| 磁盘 I/O 明细 | 折叠 | 查看单盘耗时、IOPS、吞吐和 I/O Util |
| 网络与连接明细 | 折叠 | 查看网卡带宽、TCP 连接、TIME_WAIT、重传和 ListenDrops |

## 标题命名原则

原参考看板中部分标题偏短，例如 `CPU使用率`、`内存信息`、`磁盘使用率`、`负载与进程`。最终仪表板统一按“对象 + 指标 + 用途”命名，并给复杂面板增加说明：

| 原标题或问题 | 最终标题 | 说明 |
| --- | --- | --- |
| `CPU使用率` | `CPU 使用率明细` | 明确展示的是总使用率、系统态、用户态和 IO 等待 |
| `内存信息` | `内存容量与使用率` | 明确同时包含容量值和使用率 |
| `磁盘使用率` | `分区容量明细` | 明确按挂载点查看空间，而不是 I/O 繁忙 |
| `每1秒内I/O操作耗时占比（I/O Util）` | `磁盘 I/O Util` | 保留专业名词并在描述中解释为磁盘繁忙度 |
| `进程运行状态` | `系统负载与进程状态` | 明确负载、运行进程和阻塞进程在同一排障视角 |
| `网络Socket连接信息` | `Socket 与 TCP 连接` | 明确关注连接数、TIME_WAIT、重传和 ListenDrops |

## 指标范围

参考看板和最终仪表板共同覆盖以下 node-exporter 指标：

- 主机身份与在线：`node_uname_info`、`up`
- 运行时间：`node_boot_time_seconds`
- CPU：`node_cpu_seconds_total`
- 内存与 Swap：`node_memory_MemTotal_bytes`、`node_memory_MemAvailable_bytes`、`node_memory_SwapTotal_bytes`、`node_memory_SwapFree_bytes`
- 文件系统：`node_filesystem_size_bytes`、`node_filesystem_free_bytes`、`node_filesystem_avail_bytes`
- 负载与进程：`node_load1`、`node_load5`、`node_load15`、`node_procs_running`、`node_procs_blocked`
- 文件描述符和上下文切换：`node_filefd_allocated`、`node_filefd_maximum`、`node_context_switches_total`
- 磁盘 I/O：`node_disk_io_time_seconds_total`、`node_disk_io_time_weighted_seconds_total`、`node_disk_io_now`、`node_disk_read_bytes_total`、`node_disk_written_bytes_total`、`node_disk_reads_completed_total`、`node_disk_writes_completed_total`、`node_disk_read_time_seconds_total`、`node_disk_write_time_seconds_total`
- 网络与连接：`node_network_info`、`node_network_receive_bytes_total`、`node_network_transmit_bytes_total`、`node_netstat_Tcp_CurrEstab`、`node_netstat_Tcp_RetransSegs`、`node_netstat_TcpExt_ListenDrops`、`node_sockstat_TCP_tw`、`node_sockstat_sockets_used`

## 使用注意

- 真实 ECS 接入时，`instance` 应直接表示 IP 或 `IP:port`，需要与内网参考看板保持一致。
- `group` 当前固定围绕 `数据中台`，后续扩到其他系统时可以放开正则。
- `node_network_info` 用于网卡变量；如果真实 exporter 未开启该指标，网络明细仍可通过 `$device=All` 查看聚合数据，但网卡下拉可能为空。
- 最近 7 天 P99 面板依赖 Prometheus 保留 7 天以上数据；当前 Docker 配置保留 15 天，满足该面板使用。
