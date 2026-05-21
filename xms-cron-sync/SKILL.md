---
name: xms-cron-sync
description: Sync XMS export automation configuration and workflow changes to the actual QoderWork Cron job. Use when the user says "sync cron", "同步cron", or after any change to SKILL.md, cron_payload_updated.txt, webhook_config.json, or XMS workflow strategy that needs to be reflected in the scheduled task. Supports both standard qoder_cron update and SQLite DB direct modification fallback for oversized payloads.
---

# XMS Cron 同步

## 触发条件

以下任一情况发生时，执行本同步流程：
- 用户明确说「同步cron」「同步 Cron」「sync cron」
- SKILL.md 发生策略/步骤变更
- `cron_payload_updated.txt` 内容更新
- `webhook_config.json` 配置变更（时间、群、folderId 等）
- 新增/修改 XMS 工作流步骤（如点击策略、轮询策略、异常处理）

## 同步前准备

1. 确认目标 Cron 任务 ID：`d8116018-4d30-4ec1-acc0-6bdebf2a0f60`
2. 确认 SQLite 数据库路径：`C:\Users\浩正\AppData\Roaming\QoderWork\data\agents.db`
3. 确认备用 payload 文件：`C:\Users\浩正\Desktop\AI工具\库内异常件\cron_payload_updated.txt`
4. 确认 DingTalk dev log nodeId：`AR4GpnMqJzKpOoklFa0PyON08Ke0xjE3`

## 方案 A — qoder_cron update（首选）

适用于 patch 较小的场景（< 10KB）。

1. 使用 `qoder_cron update` 更新任务：
   - `jobId`: `d8116018-4d30-4ec1-acc0-6bdebf2a0f60`
   - `patch`: 包含 `payload.message` 的完整新内容
2. 若返回成功，任务完成，进入「同步后文档更新」。
3. 若返回失败（如 patch 过大、参数截断、超时），立即降级到方案 B。

## 方案 B — SQLite 直接修改（降级）

适用于 patch 过大（> 10KB）导致 `qoder_cron update` 失败的场景。

### 步骤

1. **备份数据库**
   ```python
   import shutil
   shutil.copy2(
       r'C:\Users\浩正\AppData\Roaming\QoderWork\data\agents.db',
       r'C:\Users\浩正\AppData\Roaming\QoderWork\data\agents.db.bak'
   )
   ```

2. **读取新 payload**
   从 `cron_payload_updated.txt` 读取完整 message 内容，或从 SKILL.md 重新生成。

3. **更新数据库**
   ```python
   import sqlite3, json
   conn = sqlite3.connect(r'C:\Users\浩正\AppData\Roaming\QoderWork\data\agents.db')
   cursor = conn.cursor()

   # 读取当前 payload
   cursor.execute(
       "SELECT payload FROM scheduled_tasks WHERE id = ?",
       ('d8116018-4d30-4ec1-acc0-6bdebf2a0f60',)
   )
   row = cursor.fetchone()
   payload = json.loads(row[0])

   # 替换 message
   payload['message'] = new_message
   new_payload_json = json.dumps(payload, ensure_ascii=False)

   # 写入
   cursor.execute(
       "UPDATE scheduled_tasks SET payload = ?, updated_at = ? WHERE id = ?",
       (new_payload_json, int(time.time() * 1000), 'd8116018-4d30-4ec1-acc0-6bdebf2a0f60')
   )
   conn.commit()
   conn.close()
   ```

4. **验证更新**
   ```python
   cursor.execute("SELECT payload FROM scheduled_tasks WHERE id = ?", (task_id,))
   verified = json.loads(cursor.fetchone()[0])
   assert len(verified['message']) == len(new_message)
   ```

5. **关键字段说明**
   - 表名：`scheduled_tasks`
   - 任务 ID 字段：`id`
   - payload 字段：`payload`（JSON 字符串，包含 `kind` 和 `message`）
   - `message` 即为 Agent 执行时收到的完整指令文本

## 同步后文档更新

无论使用方案 A 还是 B，同步完成后必须依次执行：

1. **更新本地 CHANGELOG.md**
   - 文件：`C:\Users\浩正\.qoderwork\skills\xms-export-automation\CHANGELOG.md`
   - 在文件末尾追加新条目，包含：变更内容、影响范围、背景说明

2. **更新 DingTalk 开发日志**
   - 工具：`mcp__钉钉文档__update_document`
   - nodeId：`AR4GpnMqJzKpOoklFa0PyON08Ke0xjE3`
   - mode：`append`
   - 内容与 CHANGELOG.md 条目保持一致

3. **更新 MEMORY.md**
   - 工具：`memory`（target: `memory`，action: `add`）
   - 若空间不足（接近 10240 字节上限），先移除较旧或通用性较低的条目
   - 记录关键决策和降级路径发现

## 验证清单

同步完成后检查：
- [ ] `qoder_cron list` 显示任务存在且 enabled
- [ ] payload 长度与预期一致（对比 `cron_payload_updated.txt`）
- [ ] payload 中包含最新变更的关键字（如上架时间警告、双方案降级链等）
- [ ] CHANGELOG.md 已追加条目
- [ ] DingTalk dev log 追加成功
- [ ] MEMORY.md 已更新
