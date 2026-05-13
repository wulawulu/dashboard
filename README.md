# Dashboard Observability

本项目用于通过本地文件统一管理 Grafana 仪表板和 Prometheus 采集配置，逐步建设面向多个系统的可观测性看板。

当前第一阶段目标是监控包括数据中台在内的几个系统的 ECS 资源状态，先把主机层面的基础风险看清楚：哪些 ECS 在线、哪些 ECS 可能异常、CPU/内存/磁盘/网络是否存在明显压力。后续会继续扩展到服务组件、对外服务、业务链路、容量与告警等主题。

## 管理方式

- Grafana 仪表板 JSON 放在 `grafana/dashboards/` 下。
- Grafana 页面只用于预览效果，不在页面里手动编辑仪表板。
- Prometheus 数据源名称固定为 `Prometheus`。
- Grafana 通过 file provisioning 加载 dashboard，配置在 `grafana/provisioning/` 下。
- CI 部署到服务器后，会同步 `/data/dashboard` 并重建 Prometheus 数据，避免旧标签或旧 mock 数据影响当前展示。

## 主题文档

每个观测主题单独维护设计文档，说明目标、数据来源、关键指标、异常判定和健康度计算。

- [ECS 观测设计](docs/ecs-observability.md)

后续建议继续补充：

- 服务组件观测：数据库、中间件、采集组件、调度组件等。
- 对外服务观测：接口可用性、延迟、错误率、吞吐、依赖状态。
- 业务链路观测：核心链路成功率、处理积压、端到端时延。
- 告警与值班视图：异常聚合、负责人、处理状态、历史趋势。

