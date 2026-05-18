# 内网 Grafana 导入说明

本目录用于把 ECS 仪表板导入内网 Grafana。

## 文件说明

- `ECS单机明细-通用.json`
  - 只需要导入 1 次。
  - 所有系统共用这一份单机明细。
  - 通过 `系统分组` 和 `查看明细 IP` 两个变量查看具体 ECS。

- `数据中台-ECS监控总览.json`
  - ECS 总览 dashboard，可直接导入。
  - 顶部 `系统` 变量会从 Prometheus 的 `group` 标签读取系统列表，不再按系统复制多份总览。
  - 表格里的 IP 会跳转到 `ECS单机明细-通用.json`，并自动带上当前 `var-group` 和 `var-instance=<IP>`。

## 导入顺序

1. 先导入 `ECS单机明细-通用.json`。
2. 再导入 `数据中台-ECS监控总览.json`。
3. 导入时选择内网 Prometheus 数据源 `Prometheus-tenant`。

## 导入前验证

在内网 Grafana Explore 中先验证下面几条查询。这里以 `数据中台` 和 `25.64.19.107:19188` 为例，实际 IP 换成内网 ECS 的 `instance` 值：

```promql
node_uname_info{group="数据中台"}
```

```promql
node_cpu_seconds_total{group="数据中台"}
```

```promql
node_cpu_seconds_total{group="数据中台",instance=~"25.64.19.107:19188"}
```

```promql
node_filesystem_size_bytes{group="数据中台",instance=~"25.64.19.107:19188"}
```

前三条必须有数据。第四条如果没有数据，说明内网当前没有采到文件系统指标，或文件系统指标的 `group/instance/fstype/mountpoint` 标签口径与参考 dashboard 不一致；这会影响分区使用率、最小可用空间和分区容量明细。

## 健康值算法

总览表格中的 `健康值` 使用木板效应：

```text
健康值 = 在线状态 × min(CPU健康分, 内存健康分, 分区健康分, 磁盘IO健康分)
```

- 在线状态：在线为 1，不在线为 0。
- 单项资源使用率不超过 80% 时，该项健康分为 100。
- 使用率超过 80% 后，每增加 1% 扣 5 分，直到 0 分。
- 最终取所有资源健康分中的最低值。

处置建议：

```text
健康值 >= 80：正常观察
60 <= 健康值 < 80：需要关注
健康值 < 60：需要处理
```

## 必须保证的 Prometheus label

总览资源指标以参考 dashboard 中确认可用的 `group` 作为核心过滤条件，不再强制要求 CPU、内存、磁盘、网络等指标同时带 `vendor/account/name/job`。`node_uname_info` 用来提供展示信息，至少需要这些 label：

```text
vendor
account
group
name
instance
iid
```

其中：

- `group`：系统分组，例如 `数据中台`、`量测系统`、`营销系统`。
- `name`：ECS 展示名称。
- `instance`：ECS IP，或和内网参考看板一致的 IP/实例地址。
- `iid`：实例 ID。

单机明细指标按内网真实 label 查询，主要使用：

```text
group="<系统分组>"
instance=~"<查看明细 IP>"
```

## 注意事项

- `ECS单机明细-通用.json` 的 UID 是 `data-middle-ecs-detail`，不要修改。总览中的 IP 链接依赖这个 UID。
- 总览 dashboard 的 UID 是 `ecs-overview`，所有系统共用这一份总览。
- 如果导入后没有数据，优先检查内网 Prometheus 是否有对应 `group` 标签。
- 当前 JSON 内的数据源名称是 `Prometheus-tenant`。如果内网数据源名称后续变化，需要在导入时或 JSON 中同步替换。
