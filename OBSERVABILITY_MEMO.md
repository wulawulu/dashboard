# 📊 多系统监控体系搭建与管理备忘录

## 1. 核心设计思想：标签驱动 (Label-Driven)
在 Prometheus 和 Grafana 体系中，**标签（Label）是区分系统、环境和节点的唯一维度**。

### 1.1 推荐标签体系
所有接入 Prometheus 的 Target（ECS）必须统一携带以下核心标签：

| 标签 (Key) | 说明 | 示例值 |
| :--- | :--- | :--- |
| `project` | 所属系统/项目名称 | `order-system`, `data-platform` |
| `env` | 运行环境 | `prod`, `test`, `dev` |
| `instance` | 具体实例标识（建议用 IP） | `10.1.1.5`, `192.168.1.10` |
| `ecs_display_name` | 可读的业务名称（可选） | `订单中心-生产-01` |

### 1.2 标签原则
*   **全局统一**：所有 9 个系统的 Label Key 必须完全一致。
*   **禁止动态值**：标签值必须是固定的，严禁将随机数、时间戳等高频变动的值作为标签。

---

## 2. Prometheus 自动化采集规范
为了应对 9 个系统、数百台机器的规模，严禁在 `prometheus.yml` 中手动添加静态 IP。

### 2.1 采用文件自动发现 (`file_sd`)
在 `prometheus.yml` 中配置监听特定的目录：

```yaml
scrape_configs:
  - job_name: 'node_exporter'
    scrape_interval: 15s
    file_sd_configs:
      - files:
        - '/etc/prometheus/targets/*.json'  # Prometheus 会自动扫描此目录下所有 JSON
```

### 2.2 按系统管理 JSON 配置文件
在 `/etc/prometheus/targets/` 目录下，为每个系统创建独立的 `.json` 文件：

**示例文件：`system_order.json`**
```json
[
  {
    "targets": ["10.1.1.5:9100", "10.1.1.6:9100"],
    "labels": {
      "project": "order-system",
      "env": "prod"
    }
  }
]
```
*   **优势**：新增机器只需修改/增加 JSON 文件，Prometheus 自动加载，无需重启服务。

---

## 3. Node-Exporter 标准化部署
### 3.1 部署要点
*   **统一端口**：固定使用 `9100`。
*   **安全加固**：安全组必须限制仅允许 Prometheus Server IP 访问 `9100`。
*   **开机自启**：必须使用 `systemd` 管理。
*   **版本一致**：确保所有 ECS 安装的 node-exporter 版本一致，避免指标名称差异。

### 3.2 存活判定
使用基础指标 `up` 进行节点在线监测：
*   `up{project="xxx"} == 1`：节点在线且抓取正常。
*   `up{project="xxx"} == 0`：节点离线、服务挂掉或网络不通。

---

## 4. Grafana 仪表板交互设计
实现“一套仪表板查看所有系统”的关键在于**变量（Variables）**。

### 4.1 级联变量设计
在 Dashboard 设置中创建两个级联变量：

1.  **变量 `$project` (系统级)**
    *   Query: `label_values(node_uname_info, project)`
2.  **变量 `$instance` (实例级)**
    *   Query: `label_values(node_uname_info{project="$project"}, instance)`
    *   *说明：当你在顶部下拉框选择系统 A 时，实例列表会自动过滤为系统 A 下的 IP。*

### 4.2 通用查询模板
所有图表的 PromQL 必须使用变量过滤：
*   **CPU 使用率示例**：
    ```promql
    100 - (avg by (instance) (irate(node_cpu_seconds_total{project="$project", instance=~"$instance", mode="idle"}[5m])) * 100)
    ```

---

## 5. 运维必备 PromQL 集锦
*   **查看系统总机器数**：`count(up{project="$project"})`
*   **查看负载最高的前 5 台机器**：`topk(5, node_load1{project="$project"})`
*   **磁盘使用率预测（预测 4 小时后是否占满）**：
    ```promql
    predict_linear(node_filesystem_free_bytes{project="$project"}[1h], 4 * 3600) < 0
    ```

---

## 6. 新系统接入检查清单 (Checklist)
1.  [ ] **网络确认**：Prometheus Server 能够 Telnet 通目标 ECS 的 9100 端口。
2.  [ ] **端点部署**：目标 ECS 已启动 node-exporter 且设置为开机自启。
3.  [ ] **标签对齐**：确定新系统的 `project` 唯一标识符。
4.  [ ] **配置更新**：在 `targets/` 目录下创建对应的 `.json` 文件。
5.  [ ] **变量验证**：打开 Grafana，检查 `$project` 变量下拉框是否出现了新系统。
6.  [ ] **告警配置**：确认该系统的 CPU、内存、磁盘报警阈值及通知对象。

---

## 7. 注意事项与坑点
*   **时钟同步**：所有 ECS 必须配置 NTP 服务，时间偏差过大会导致 Prometheus 数据抓取异常。
*   **存储空间**：Prometheus 默认保存 15 天数据，如果 9 个系统数据量大，需调整 `--storage.tsdb.retention.time`。
*   **性能瓶颈**：单台 Prometheus 抓取数百个节点压力不大，但如果查询语句太复杂（如大范围的 `rate`），会拖慢 Grafana 加载速度。
