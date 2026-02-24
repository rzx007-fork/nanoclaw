# NanoClaw 数据模型文档

| 文档版本 | 日期       | 作者     | 变更说明 |
| -------- | ---------- | -------- | -------- |
| v1.0     | 2025-02-24 | 开发团队 | 初始版本 |

---

## 1. 概述

### 1.1 数据库选择

NanoClaw 使用 **SQLite** 作为主要数据存储，原因：

| 优势      | 说明                       |
| --------- | -------------------------- |
| 零配置    | 无需数据库服务器进程       |
| 轻量级    | 单个文件存储所有数据       |
| ACID 事务 | 保证数据一致性             |
| 跨平台    | Windows、macOS、Linux 通用 |
| 足够性能  | 个人助手场景性能充足       |

### 1.2 数据库位置

```
store/messages.db
```

### 1.3 数据库模式

```
- WAL (Write-Ahead Logging) 模式已启用
- 外键约束已启用
- 自动迁移机制（ALTER TABLE if column not exists）
```

---

## 2. 数据库表结构

### 2.1 ER 图

```
┌─────────────────┐
│     chats       │
├─────────────────┤
│ jid (PK)       │◄──────┐
│ name           │       │
│ last_message   │       │
│ channel        │       │
│ is_group       │       │
└─────────────────┘       │
                          │
                          │ 1:N
                          │
┌─────────────────┐       │
│    messages     │       │
├─────────────────┤       │
│ id (PK)        │       │
│ chat_jid (FK)  │───────┘
│ sender         │
│ sender_name    │
│ content       │
│ timestamp     │
│ is_from_me    │
│ is_bot_message│
└─────────────────┘

┌─────────────────┐
│scheduled_tasks │
├─────────────────┤
│ id (PK)        │◄──────┐
│ group_folder    │       │
│ chat_jid (FK)  │       │
│ prompt         │       │
│ schedule_type  │       │
│ schedule_value │       │
│ context_mode   │       │
│ next_run       │       │
│ last_run       │       │
│ last_result    │       │
│ status         │       │
│ created_at     │       │
└─────────────────┘       │
                          │
                          │ 1:N
                          │
┌─────────────────┐       │
│ task_run_logs  │       │
├─────────────────┤       │
│ id (PK)        │       │
│ task_id (FK)   │───────┘
│ run_at         │
│ duration_ms    │
│ status         │
│ result         │
│ error          │
└─────────────────┘

┌─────────────────┐
│  router_state  │
├─────────────────┤
│ key (PK)       │
│ value          │
└─────────────────┘

┌─────────────────┐
│   sessions     │
├─────────────────┤
│ group_folder(PK)│
│ session_id     │
└─────────────────┘

┌─────────────────┐
│registered_groups│
├─────────────────┤
│ jid (PK)        │
│ name            │
│ folder          │
│ trigger_pattern │
│ added_at        │
│ container_config│
│ requires_trigger│
└─────────────────┘
```

---

## 3. 表详解

### 3.1 chats - 聊天元数据

存储所有已知聊天的元数据（不包含消息内容）。

#### 表结构

| 字段              | 类型    | 约束        | 说明                                        |
| ----------------- | ------- | ----------- | ------------------------------------------- |
| jid               | TEXT    | PRIMARY KEY | 聊天 ID（WhatsApp/Discord/Telegram 等）     |
| name              | TEXT    | NOT NULL    | 聊天名称（群组名或联系人名）                |
| last_message_time | TEXT    | NOT NULL    | 最后消息时间（ISO 8601）                    |
| channel           | TEXT    | NULLABLE    | 通道类型：`whatsapp`, `discord`, `telegram` |
| is_group          | INTEGER | DEFAULT 0   | 是否群组：1=是, 0=否                        |

#### 索引

无（主键自动索引）

#### 特殊记录

| JID              | 用途                       |
| ---------------- | -------------------------- |
| `__group_sync__` | 存储最后群组元数据同步时间 |

#### 使用场景

```typescript
// 更新聊天元数据
storeChatMetadata(
  '123456@g.us',
  '2025-02-24T10:00:00Z',
  'Project Team',
  'whatsapp',
  true,
);

// 获取所有聊天
const chats = getAllChats(); // 按最后消息时间排序

// 获取最后同步时间
const lastSync = getLastGroupSync();
```

---

### 3.2 messages - 消息

存储注册群组的完整消息历史。

#### 表结构

| 字段           | 类型    | 约束        | 说明                       |
| -------------- | ------- | ----------- | -------------------------- |
| id             | TEXT    | PRIMARY KEY | 消息 ID                    |
| chat_jid       | TEXT    | FOREIGN KEY | 聊天 ID（引用 chats.jid）  |
| sender         | TEXT    | NOT NULL    | 发送者 JID                 |
| sender_name    | TEXT    | NOT NULL    | 发送者名称                 |
| content        | TEXT    | NOT NULL    | 消息内容                   |
| timestamp      | TEXT    | NOT NULL    | 时间戳（ISO 8601）         |
| is_from_me     | INTEGER | NOT NULL    | 是否本人发送：1=是, 0=否   |
| is_bot_message | INTEGER | DEFAULT 0   | 是否机器人消息：1=是, 0=否 |

#### 复合主键

```sql
PRIMARY KEY (id, chat_jid)
```

#### 索引

```sql
CREATE INDEX idx_timestamp ON messages(timestamp);
```

#### 查询示例

```typescript
// 获取新消息（所有注册群组）
const { messages, newTimestamp } = getNewMessages(
  ['123456@g.us', '789012@g.us'],
  '2025-02-24T09:00:00Z',
  'Andy',
);

// 获取指定聊天自某时间后的消息
const messages = getMessagesSince(
  '123456@g.us',
  '2025-02-24T09:00:00Z',
  'Andy',
);

// 存储消息
storeMessage({
  id: 'msg-123',
  chat_jid: '123456@g.us',
  sender: 'user@s.whatsapp.net',
  sender_name: 'John Doe',
  content: '@Andy 帮我检查日志',
  timestamp: '2025-02-24T10:00:00Z',
  is_from_me: false,
});
```

#### 过滤规则

查询时自动过滤：

1. `is_bot_message = 0` - 排除机器人消息
2. `content NOT LIKE 'Andy:%'` - 排除旧版助手消息（向后兼容）
3. `content != '' AND content IS NOT NULL` - 排除空消息

---

### 3.3 scheduled_tasks - 定时任务

存储所有定时任务的配置和状态。

#### 表结构

| 字段           | 类型 | 约束               | 说明                                         |
| -------------- | ---- | ------------------ | -------------------------------------------- |
| id             | TEXT | PRIMARY KEY        | 任务 ID（格式：`task-{timestamp}-{random}`） |
| group_folder   | TEXT | NOT NULL           | 所属组文件夹                                 |
| chat_jid       | TEXT | NOT NULL           | 关联的聊天 JID                               |
| prompt         | TEXT | NOT NULL           | 任务提示词                                   |
| schedule_type  | TEXT | NOT NULL           | 调度类型：`cron`, `interval`, `once`         |
| schedule_value | TEXT | NOT NULL           | 调度值（cron 表达式/毫秒/ISO 时间戳）        |
| context_mode   | TEXT | DEFAULT 'isolated' | 上下文模式：`group`, `isolated`              |
| next_run       | TEXT | NULLABLE           | 下次运行时间（ISO 8601）                     |
| last_run       | TEXT | NULLABLE           | 最后运行时间                                 |
| last_result    | TEXT | NULLABLE           | 最后运行结果摘要                             |
| status         | TEXT | DEFAULT 'active'   | 状态：`active`, `paused`, `completed`        |
| created_at     | TEXT | NOT NULL           | 创建时间（ISO 8601）                         |

#### 索引

```sql
CREATE INDEX idx_next_run ON scheduled_tasks(next_run);
CREATE INDEX idx_status ON scheduled_tasks(status);
```

#### 调度类型说明

| 类型       | schedule_value 示例    | 说明                               |
| ---------- | ---------------------- | ---------------------------------- |
| `cron`     | `0 9 * * 1-5`          | Cron 表达式（每天周一到周五 9:00） |
| `interval` | `86400000`             | 间隔毫秒（每 24 小时）             |
| `once`     | `2025-03-01T08:00:00Z` | 单次执行（ISO 8601 时间戳）        |

#### 上下文模式说明

| 模式       | 说明                               |
| ---------- | ---------------------------------- |
| `group`    | 复用群的当前会话，保持上下文连续性 |
| `isolated` | 创建新会话，独立上下文             |

#### 使用示例

```typescript
// 创建任务
createTask({
  id: 'task-1708838400000-abc123',
  group_folder: 'project-team',
  chat_jid: '123456@g.us',
  prompt: '生成本周 git commit 报告',
  schedule_type: 'cron',
  schedule_value: '0 17 * * 5', // 每周五 17:00
  context_mode: 'group',
  next_run: '2025-02-28T17:00:00Z',
  status: 'active',
  created_at: '2025-02-24T10:00:00Z',
});

// 获取到期任务
const dueTasks = getDueTasks(); // status='active' AND next_run <= now

// 更新任务
updateTask(taskId, {
  status: 'paused',
  next_run: '2025-03-01T09:00:00Z',
});

// 任务执行后更新
updateTaskAfterRun(taskId, nextRun, lastResult);

// 删除任务
deleteTask(taskId);
```

---

### 3.4 task_run_logs - 任务执行日志

记录定时任务的每次执行历史。

#### 表结构

| 字段        | 类型    | 约束                      | 说明                         |
| ----------- | ------- | ------------------------- | ---------------------------- |
| id          | INTEGER | PRIMARY KEY AUTOINCREMENT | 自增 ID                      |
| task_id     | TEXT    | FOREIGN KEY               | 关联任务 ID                  |
| run_at      | TEXT    | NOT NULL                  | 执行时间（ISO 8601）         |
| duration_ms | INTEGER | NOT NULL                  | 执行耗时（毫秒）             |
| status      | TEXT    | NOT NULL                  | 执行状态：`success`, `error` |
| result      | TEXT    | NULLABLE                  | 执行结果                     |
| error       | TEXT    | NULLABLE                  | 错误信息                     |

#### 索引

```sql
CREATE INDEX idx_task_run_logs ON task_run_logs(task_id, run_at);
```

#### 使用示例

```typescript
// 记录任务执行
logTaskRun({
  task_id: 'task-123',
  run_at: '2025-02-24T10:00:00Z',
  duration_ms: 5000,
  status: 'success',
  result: '报告已发送到群组',
  error: null,
});
```

---

### 3.5 router_state - 路由状态

存储路由器和系统的关键状态。

#### 表结构

| 字段  | 类型 | 约束        | 说明   |
| ----- | ---- | ----------- | ------ |
| key   | TEXT | PRIMARY KEY | 状态键 |
| value | TEXT | NOT NULL    | 状态值 |

#### 预定义键值

| 键                     | 值类型   | 说明                                                     |
| ---------------------- | -------- | -------------------------------------------------------- |
| `last_timestamp`       | ISO 8601 | 最后处理的全球消息时间戳                                 |
| `last_agent_timestamp` | JSON     | 每个组最后处理的消息时间（格式：`{"jid": "timestamp"}`） |

#### 使用示例

```typescript
// 获取状态
const lastTimestamp = getRouterState('last_timestamp');
const agentTimestamps = JSON.parse(
  getRouterState('last_agent_timestamp') || '{}',
);

// 设置状态
setRouterState('last_timestamp', '2025-02-24T10:00:00Z');
setRouterState('last_agent_timestamp', JSON.stringify(agentTimestamps));
```

---

### 3.6 sessions - Claude 会话

存储每个群的 Claude Agent SDK 会话 ID。

#### 表结构

| 字段         | 类型 | 约束        | 说明           |
| ------------ | ---- | ----------- | -------------- |
| group_folder | TEXT | PRIMARY KEY | 组文件夹       |
| session_id   | TEXT | NOT NULL    | Claude 会话 ID |

#### 使用示例

```typescript
// 获取会话
const sessionId = getSession('project-team');

// 设置会话
setSession('project-team', 'session-uuid-123');

// 获取所有会话
const allSessions = getAllSessions(); // { "project-team": "session-123", ... }
```

---

### 3.7 registered_groups - 注册群组

存储所有注册到 NanoClaw 的群组配置。

#### 表结构

| 字段             | 类型    | 约束            | 说明                       |
| ---------------- | ------- | --------------- | -------------------------- |
| jid              | TEXT    | PRIMARY KEY     | 聊天 JID                   |
| name             | TEXT    | NOT NULL        | 群组名称                   |
| folder           | TEXT    | UNIQUE NOT NULL | 组文件夹（路径安全验证）   |
| trigger_pattern  | TEXT    | NOT NULL        | 触发词模式（如 `@Andy`）   |
| added_at         | TEXT    | NOT NULL        | 添加时间                   |
| container_config | TEXT    | NULLABLE        | 容器配置（JSON）           |
| requires_trigger | INTEGER | DEFAULT 1       | 是否需要触发词：1=是, 0=否 |

#### container_config 格式

```json
{
  "additionalMounts": [
    {
      "hostPath": "/Users/john/projects",
      "containerPath": "/workspace/projects",
      "readonly": true
    }
  ],
  "timeout": 300000
}
```

#### 使用示例

```typescript
// 注册群组
setRegisteredGroup('123456@g.us', {
  name: 'Project Team',
  folder: 'project-team',
  trigger: '@Andy',
  added_at: '2025-02-24T10:00:00Z',
  containerConfig: {
    additionalMounts: [
      {
        hostPath: '/Users/john/projects',
        containerPath: '/workspace/projects',
        readonly: true,
      },
    ],
    timeout: 300000,
  },
  requiresTrigger: true,
});

// 获取群组
const group = getRegisteredGroup('123456@g.us');

// 获取所有群组
const allGroups = getAllRegisteredGroups();
```

---

## 4. 数据流

### 4.1 消息处理流程

```
用户发送 WhatsApp 消息
    ↓
WhatsApp 通道接收
    ↓
storeChatMetadata() → chats 表
    ↓
storeMessage() → messages 表
    ↓
消息轮询循环（每 2s）
    ↓
getNewMessages() → 查询 messages 表
    ↓
检查触发词匹配
    ↓
GroupQueue.enqueueMessageCheck()
    ↓
启动容器或复用容器
    ↓
Agent 处理
    ↓
结果通过 IPC 写入文件
    ↓
IPC Watcher 读取
    ↓
发送回复到 WhatsApp
    ↓
setRouterState('last_timestamp') → router_state 表
```

### 4.2 定时任务流程

```
调度器循环（每 60s）
    ↓
getDueTasks() → 查询 scheduled_tasks 表
    ↓
对于每个到期任务
    ↓
启动容器
    ↓
Agent 执行任务
    ↓
logTaskRun() → task_run_logs 表
    ↓
updateTaskAfterRun() → 更新 scheduled_tasks 表
    ↓
计算下次运行时间
    └─ cron: cron-parser 计算下次时间
    └─ interval: 当前时间 + 间隔
    └─ once: next_run = null
```

---

## 5. 数据迁移

### 5.1 迁移策略

NanoClaw 使用 **渐进式迁移**（Incremental Migration）：

```typescript
// 1. 尝试添加新列
try {
  database.exec(
    `ALTER TABLE messages ADD COLUMN is_bot_message INTEGER DEFAULT 0`,
  );
} catch {
  // 列已存在，忽略错误
}

// 2. 回填数据（如需要）
database.exec(
  `UPDATE messages SET is_bot_message = 1 WHERE content LIKE 'Andy:%'`,
);
```

### 5.2 JSON → SQLite 迁移

从早期版本的 JSON 文件自动迁移到 SQLite：

| JSON 文件                     | 目标表            | 迁移逻辑         |
| ----------------------------- | ----------------- | ---------------- |
| `data/router_state.json`      | router_state      | 迁移所有键值对   |
| `data/sessions.json`          | sessions          | 迁移所有会话     |
| `data/registered_groups.json` | registered_groups | 迁移所有注册群组 |

迁移后，原文件重命名为 `*.migrated`。

### 5.3 已执行的迁移

| 版本 | 迁移内容                                   |
| ---- | ------------------------------------------ |
| v1.0 | 创建初始表结构                             |
| v1.1 | 添加 `context_mode` 列到 `scheduled_tasks` |
| v1.1 | 添加 `is_bot_message` 列到 `messages`      |
| v1.2 | 添加 `channel` 和 `is_group` 列到 `chats`  |

---

## 6. 性能优化

### 6.1 索引策略

| 表              | 索引                | 查询优化           |
| --------------- | ------------------- | ------------------ |
| messages        | `idx_timestamp`     | 按时间范围查询     |
| scheduled_tasks | `idx_next_run`      | 查询到期任务       |
| scheduled_tasks | `idx_status`        | 按状态筛选         |
| task_run_logs   | `idx_task_run_logs` | 按任务 ID 查询历史 |

### 6.2 查询优化

```typescript
// ✅ 使用索引的查询
const dueTasks = db
  .prepare(
    `
  SELECT * FROM scheduled_tasks
  WHERE status = ? AND next_run IS NOT NULL AND next_run <= ?
  ORDER BY next_run
`,
  )
  .all('active', nowIso);

// ❌ 避免全表扫描
const allMessages = db
  .prepare(
    `
  SELECT * FROM messages
  WHERE content LIKE '%error%'
`,
  )
  .all(); // 无索引，性能差
```

### 6.3 批量操作

```typescript
// ✅ 批量插入（使用预编译语句）
const stmt = db.prepare(`INSERT INTO messages VALUES (...)`);
const transaction = db.transaction((messages) => {
  for (const msg of messages) {
    stmt.run(msg.id, msg.chat_jid, ...);
  }
});
transaction(messages);

// ❌ 避免逐条插入
for (const msg of messages) {
  db.prepare(`INSERT INTO messages VALUES (...)`).run(...); // 慢
}
```

---

## 7. 数据备份与恢复

### 7.1 备份

```bash
# 方法 1: 直接复制数据库文件
cp store/messages.db store/messages.db.backup.$(date +%Y%m%d)

# 方法 2: SQLite dump
sqlite3 store/messages.db ".backup store/messages.db.backup"

# 方法 3: 备份整个 store 目录
tar -czf backup-$(date +%Y%m%d).tar.gz store/
```

### 7.2 恢复

```bash
# 停止 NanoClaw
launchctl unload ~/Library/LaunchAgents/com.nanoclaw.plist

# 恢复数据库
cp store/messages.db.backup.20250224 store/messages.db

# 重启 NanoClaw
launchctl load ~/Library/LaunchAgents/com.nanoclaw.plist
```

### 7.3 备份建议

- **频率**: 每日自动备份
- **保留**: 最近 30 天的备份
- **异地**: 考虑云存储（如 AWS S3）

---

## 8. 数据清理

### 8.1 清理策略

| 数据类型     | 保留策略   |
| ------------ | ---------- |
| 消息         | 无限期保留 |
| 任务执行日志 | 保留 90 天 |
| 旧版本备份   | 保留 30 天 |

### 8.2 清理脚本

```sql
-- 清理 90 天前的任务日志
DELETE FROM task_run_logs
WHERE run_at < datetime('now', '-90 days');

-- 清理已完成且 30 天未更新的单次任务
DELETE FROM scheduled_tasks
WHERE status = 'completed'
  AND schedule_type = 'once'
  AND last_run < datetime('now', '-30 days');

-- 优化数据库
VACUUM;
ANALYZE;
```

---

## 9. 数据完整性

### 9.1 外键约束

```sql
-- messages.chat_jid 引用 chats.jid
FOREIGN KEY (chat_jid) REFERENCES chats(jid)

-- task_run_logs.task_id 引用 scheduled_tasks.id
FOREIGN KEY (task_id) REFERENCES scheduled_tasks(id)
```

### 9.2 级联删除

```typescript
// 删除任务时，先删除关联的执行日志
db.prepare('DELETE FROM task_run_logs WHERE task_id = ?').run(taskId);
db.prepare('DELETE FROM scheduled_tasks WHERE id = ?').run(taskId);
```

### 9.3 唯一性约束

```sql
-- 组文件夹名称唯一（防止冲突）
UNIQUE(folder) in registered_groups
```

---

## 10. 数据访问层 API

### 10.1 完整 API 列表

参见 `src/db.ts` 源代码，主要函数分类：

| 类别     | 函数                                                                                              |
| -------- | ------------------------------------------------------------------------------------------------- |
| 初始化   | `initDatabase()`, `_initTestDatabase()`                                                           |
| 聊天     | `storeChatMetadata()`, `getAllChats()`, `updateChatName()`                                        |
| 消息     | `storeMessage()`, `getNewMessages()`, `getMessagesSince()`                                        |
| 任务     | `createTask()`, `getTaskById()`, `getAllTasks()`, `updateTask()`, `deleteTask()`, `getDueTasks()` |
| 任务日志 | `logTaskRun()`                                                                                    |
| 路由状态 | `getRouterState()`, `setRouterState()`                                                            |
| 会话     | `getSession()`, `setSession()`, `getAllSessions()`                                                |
| 注册群组 | `getRegisteredGroup()`, `setRegisteredGroup()`, `getAllRegisteredGroups()`                        |

---

## 11. 扩展性考虑

### 11.1 新增表的指导原则

1. **命名**: 复数形式，如 `messages`, `tasks`
2. **主键**: 优先使用 TEXT（兼容 JID），自增 ID 使用 INTEGER
3. **时间戳**: 统一使用 ISO 8601 字符串（TEXT）
4. **索引**: 查询字段必须添加索引
5. **迁移**: 新列使用 `ALTER TABLE IF NOT EXISTS`

### 11.2 多通道支持扩展

```sql
-- channels 表（未来扩展）
CREATE TABLE IF NOT EXISTS channels (
  id TEXT PRIMARY KEY,
  type TEXT NOT NULL,  -- 'whatsapp', 'telegram', 'discord'
  config TEXT,          -- JSON 配置
  is_active INTEGER DEFAULT 1
);

-- messages 表已支持多通道（channel 字段）
-- 未来可通过 chat_jid 前缀区分通道
--   'tg:' for Telegram
--   'dc:' for Discord
```

---

**文档维护**: 开发团队
**最后更新**: 2025-02-24
