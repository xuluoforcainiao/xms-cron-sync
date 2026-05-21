# XMS Cron 同步 — 参考文档

## SQLite 数据库结构

数据库路径：`C:\Users\浩正\AppData\Roaming\QoderWork\data\agents.db`

### scheduled_tasks 表

| 字段 | 类型 | 说明 |
|------|------|------|
| id | TEXT | 任务唯一标识（UUID） |
| name | TEXT | 任务名称 |
| description | TEXT | 任务描述 |
| schedule | TEXT | Cron 表达式 JSON（如 `{"kind":"cron","expr":"0 8 * * *","tz":"Asia/Shanghai"}`） |
| payload | TEXT | Agent 执行时的完整指令（JSON 字符串，含 `kind` 和 `message`） |
| enabled | INTEGER | 1=启用，0=禁用 |
| next_run_at | INTEGER | 下次运行时间戳（毫秒） |
| updated_at | INTEGER | 最后更新时间戳（毫秒） |

### 关键任务 ID

- XMS 主任务：`d8116018-4d30-4ec1-acc0-6bdebf2a0f60`
- 结果通知任务：`659a6c92-6b08-49ec-a80c-0358c6ae6290`

## qoder_cron update 失败的历史原因

| 失败原因 | 现象 | 解决方案 |
|----------|------|----------|
| Patch 过大 | 传入 21KB JSON 后无响应或超时 | 降级为 SQLite 直接修改 |
| 参数截断 | 大字符串被中间层截断 | 同样降级为 SQLite 直接修改 |
| 正常成功 | 返回成功，next_run_at 更新 | 无需降级 |

## 数据库直接修改的安全原则

1. **始终先备份**：`agents.db.bak`
2. **使用事务**：`conn.commit()` 确保原子性
3. **验证后才算完成**：读取回 payload 对比长度和关键字
4. **不要修改其他字段**：仅更新 `payload` 和 `updated_at`
5. **不要修改其他任务**：WHERE 条件必须包含具体任务 ID

## 相关文件路径速查

| 文件 | 路径 |
|------|------|
| 备用 payload | `C:\Users\浩正\Desktop\AI工具\库内异常件\cron_payload_updated.txt` |
| 配置文件 | `C:\Users\浩正\Desktop\AI工具\库内异常件\webhook_config.json` |
| 执行日志目录 | `C:\Users\浩正\Desktop\AI工具\库内异常件\任务执行日志\` |
| 报告看板 | `C:\Users\浩正\Desktop\AI工具\库内异常件\XMS执行报告看板.html` |
| CHANGELOG | `C:\Users\浩正\.qoderwork\skills\xms-export-automation\CHANGELOG.md` |
| DingTalk dev log | nodeId: `AR4GpnMqJzKpOoklFa0PyON08Ke0xjE3` |
