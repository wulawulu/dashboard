# 内网 Grafana 导入说明

本目录用于把 ECS 仪表板导入内网 Grafana。

## 文件说明

- `ECS单机明细-通用.json`
  - 只需要导入 1 次。
  - 所有系统共用这一份单机明细。
  - 通过 `系统分组` 和 `查看明细 IP` 两个变量查看具体 ECS。

- `数据中台-ECS监控总览.json`
  - 数据中台总览 dashboard，可直接导入。
  - 表格里的 IP 会跳转到 `ECS单机明细-通用.json`，并自动带上 `var-group=数据中台` 和 `var-instance=<IP>`。

- `总览模板-复制后替换系统名和分组.json`
  - 给其他 8 个系统复制使用。
  - 每个系统复制一份，然后替换占位符。

## 导入顺序

1. 先导入 `ECS单机明细-通用.json`。
2. 再导入各系统总览 JSON。
3. 导入时选择内网 Prometheus 数据源。

## 创建其他系统总览

复制 `总览模板-复制后替换系统名和分组.json`，每个系统替换 3 类内容：

```text
__系统名称__
__系统分组__
__system_uid__
```

示例：

```text
__系统名称__  -> 营销系统
__系统分组__  -> 营销系统
__system_uid__ -> marketing
```

替换后建议文件命名为：

```text
营销系统-ECS监控总览.json
```

## 必须保证的 Prometheus label

内网 Prometheus 中 ECS 指标至少需要这些 label：

```text
vendor
account
group
name
instance
iid
```

其中：

- `group`：系统分组，例如 `数据中台`、`营销系统`。
- `name`：ECS 展示名称。
- `instance`：ECS IP，或和内网参考看板一致的 IP/实例地址。
- `iid`：实例 ID。

## 注意事项

- `ECS单机明细-通用.json` 的 UID 是 `data-middle-ecs-detail`，不要修改。总览中的 IP 链接依赖这个 UID。
- 每个总览 dashboard 的 UID 必须不同，模板中的 `__system_uid__` 就是为了避免冲突。
- 如果导入后没有数据，优先检查内网 Prometheus 是否有对应 `group` 标签。
- 如果内网数据源名称不是 `Prometheus`，导入时在 Grafana 页面选择正确的数据源。
