# NanoClaw 开发实施计划

| 文档版本 | 日期       | 作者     | 变更说明 |
| -------- | ---------- | -------- | -------- |
| v1.0     | 2025-02-24 | 开发团队 | 初始版本 |

---

## 1. 概述

### 1.1 项目目标

开发一个轻量级、安全的个人 AI 助手，通过 WhatsApp 与用户交互，在容器隔离环境中运行 Claude Agent。

### 1.2 开发时间线

| 阶段                   | 持续时间     | 里程碑                     |
| ---------------------- | ------------ | -------------------------- |
| Phase 0: 环境准备      | 1-2 天       | 开发环境就绪               |
| Phase 1: 基础架构      | 2-3 周       | 单进程、SQLite、消息存储   |
| Phase 2: 容器集成      | 2-3 周       | Docker 容器、安全隔离      |
| Phase 3: WhatsApp 集成 | 1-2 周       | 消息收发、认证             |
| Phase 4: Agent 执行    | 2-3 周       | Claude Agent SDK、工具调用 |
| Phase 5: 任务调度      | 1-2 周       | 定时任务、Cron 解析        |
| Phase 6: IPC 通信      | 1 周         | 文件系统 IPC、权限控制     |
| Phase 7: 多队列管理    | 1 周         | 按组队列、并发控制         |
| Phase 8: 生产化        | 1 周         | 服务守护、日志、监控       |
| **总计**               | **11-17 周** | 完整功能系统               |

---

## 2. Phase 0: 环境准备

### 2.1 目标

搭建开发环境，确保所有工具链正常工作。

### 2.2 任务清单

| 任务 ID | 任务                    | 优先级 | 预估时间 | 依赖   |
| ------- | ----------------------- | ------ | -------- | ------ |
| P0-001  | 初始化 Git 仓库         | P0     | 30 分钟  | -      |
| P0-002  | 配置 TypeScript         | P0     | 30 分钟  | P0-001 |
| P0-003  | 配置 ESLint 和 Prettier | P1     | 30 分钟  | P0-002 |
| P0-004  | 配置 Vitest 测试框架    | P0     | 1 小时   | P0-002 |
| P0-005  | 验证 Docker 安装        | P0     | 30 分钟  | -      |
| P0-006  | 创建基础目录结构        | P0     | 30 分钟  | P0-001 |

### 2.3 验收标准

- [ ] Git 仓库可正常提交
- [ ] TypeScript 编译无错误
- [ ] 测试框架可运行 `npm test`
- [ ] Docker 可正常启动 `docker run hello-world`
- [ ] 目录结构符合规范

### 2.4 完成标志

```bash
# 验证环境
npm run typecheck  # 无错误
npm run format:check  # 无差异
npm test  # 通过

# 验证 Docker
docker run --rm node:20 node --version  # 输出 v20.x.x
```

---

## 3. Phase 1: 基础架构

### 3.1 目标

建立核心架构：配置管理、日志系统、数据库基础。

### 3.2 任务清单

| 任务 ID | 任务                          | 优先级 | 预估时间 | 依赖          |
| ------- | ----------------------------- | ------ | -------- | ------------- |
| P1-001  | 实现配置模块（src/config.ts） | P0     | 2 小时   | P0-006        |
| P1-002  | 实现日志模块（src/logger.ts） | P0     | 1 小时   | P0-002        |
| P1-003  | 定义核心类型（src/types.ts）  | P0     | 2 小时   | -             |
| P1-004  | 实现 SQLite 数据库初始化      | P0     | 4 小时   | P1-003        |
| P1-005  | 创建 chats 表和索引           | P1     | 1 小时   | P1-004        |
| P1-006  | 创建 messages 表和索引        | P1     | 1 小时   | P1-004        |
| P1-007  | 实现路由状态存储              | P1     | 1 小时   | P1-004        |
| P1-008  | 编写数据库单元测试            | P2     | 3 小时   | P1-004-P1-007 |

### 3.3 详细实现

#### P1-001: 配置模块

```typescript
// src/config.ts
import os from 'os';
import path from 'path';

export const ASSISTANT_NAME = process.env.ASSISTANT_NAME || 'Andy';
export const POLL_INTERVAL = 2000;
export const SCHEDULER_POLL_INTERVAL = 60000;

const PROJECT_ROOT = process.cwd();
export const STORE_DIR = path.resolve(PROJECT_ROOT, 'store');
export const GROUPS_DIR = path.resolve(PROJECT_ROOT, 'groups');
export const DATA_DIR = path.resolve(PROJECT_ROOT, 'data');
export const MAIN_GROUP_FOLDER = 'main';

export const CONTAINER_IMAGE = 'nanoclaw-agent:latest';
export const CONTAINER_TIMEOUT = 1800000;
export const MAX_CONCURRENT_CONTAINERS = 5;

export const TRIGGER_PATTERN = new RegExp(`^@${ASSISTANT_NAME}\\b`, 'i');
```

#### P1-002: 日志模块

```typescript
// src/logger.ts
import pino from 'pino';

const transport = pino.transport({
  target: 'pino-pretty',
  options: {
    colorize: true,
  },
});

export const logger = pino(transport);
```

#### P1-004: 数据库初始化

```typescript
// src/db.ts
import Database from 'better-sqlite3';
import fs from 'fs';
import path from 'path';

let db: Database.Database;

function createSchema(): void {
  db.exec(`
    CREATE TABLE IF NOT EXISTS chats (
      jid TEXT PRIMARY KEY,
      name TEXT,
      last_message_time TEXT,
      channel TEXT,
      is_group INTEGER DEFAULT 0
    );

    CREATE TABLE IF NOT EXISTS messages (
      id TEXT,
      chat_jid TEXT,
      sender TEXT,
      sender_name TEXT,
      content TEXT,
      timestamp TEXT,
      is_from_me INTEGER,
      is_bot_message INTEGER DEFAULT 0,
      PRIMARY KEY (id, chat_jid),
      FOREIGN KEY (chat_jid) REFERENCES chats(jid)
    );

    CREATE INDEX IF NOT EXISTS idx_timestamp ON messages(timestamp);

    CREATE TABLE IF NOT EXISTS router_state (
      key TEXT PRIMARY KEY,
      value TEXT NOT NULL
    );
  `);
}

export function initDatabase(): void {
  const dbPath = path.join(STORE_DIR, 'messages.db');
  fs.mkdirSync(path.dirname(dbPath), { recursive: true });
  db = new Database(dbPath);
  createSchema();
}
```

### 3.4 验收标准

- [ ] 配置模块可读取环境变量
- [ ] 日志可输出到文件和控制台
- [ ] SQLite 数据库可创建和查询
- [ ] 所有单元测试通过（>90% 覆盖率）

### 3.5 完成标志

```bash
# 运行测试
npm test db.test.ts

# 验证数据库
sqlite3 store/messages.db ".schema"

# 查看日志
cat logs/nanoclaw.log
```

---

## 4. Phase 2: 容器集成

### 4.1 目标

实现容器运行时：Docker 集成、挂载系统、安全验证。

### 4.2 任务清单

| 任务 ID | 任务                                       | 优先级 | 预估时间 | 依赖           |
| ------- | ------------------------------------------ | ------ | -------- | -------------- |
| P2-001  | 设计容器镜像（Dockerfile）                 | P0     | 3 小时   | -              |
| P2-002  | 实现容器运行时抽象（container-runtime.ts） | P0     | 4 小时   | P2-001         |
| P2-003  | 实现挂载安全验证（mount-security.ts）      | P0     | 3 小时   | -              |
| P2-004  | 实现容器启动器（container-runner.ts）      | P0     | 6 小时   | P2-002, P2-003 |
| P2-005  | 实现流式输出处理                           | P1     | 2 小时   | P2-004         |
| P2-006  | 编写容器单元测试                           | P2     | 3 小时   | P2-002-P2-005  |
| P2-007  | 创建容器构建脚本（build.sh）               | P1     | 1 小时   | P2-001         |

### 4.3 详细实现

#### P2-001: Dockerfile

```dockerfile
# container/Dockerfile
FROM node:20-bullseye-slim

RUN apt-get update && \
    apt-get install -y --no-install-recommends \
    chromium \
    chromium-driver && \
    rm -rf /var/lib/apt/lists/*

RUN groupadd -r node && useradd -r -g node node
USER node

WORKDIR /app

COPY package*.json ./
RUN npm ci --only=production

COPY . .

ENV PATH="/app/node_modules/.bin:${PATH}"

ENTRYPOINT ["node", "dist/index.js"]
```

#### P2-002: 容器运行时抽象

```typescript
// src/container-runtime.ts
import { execSync } from 'child_process';

export interface ContainerConfig {
  image: string;
  mounts: Array<{
    source: string;
    target: string;
    readonly?: boolean;
  }>;
  env?: Record<string, string>;
  timeout?: number;
}

export async function runContainer(
  config: ContainerConfig,
): Promise<{ pid: number; containerId: string }> {
  // Docker 实现细节
  const mountArgs = config.mounts.map(
    (m) =>
      `--mount type=bind,source=${m.source},target=${m.target}${m.readonly ? ',readonly' : ''}`,
  );

  const envArgs = Object.entries(config.env || {}).map(
    ([k, v]) => `-e ${k}=${v}`,
  );

  const cmd = [
    'docker',
    'run',
    '-d',
    '-i',
    ...mountArgs,
    ...envArgs,
    config.image,
    'node',
    'dist/index.js',
  ];

  const output = execSync(cmd.join(' '), { encoding: 'utf-8' });
  const containerId = output.trim();

  return { pid: 0, containerId };
}

export async function stopContainer(containerId: string): Promise<void> {
  execSync(`docker stop ${containerId}`);
  execSync(`docker rm ${containerId}`);
}
```

#### P2-003: 挂载安全验证

```typescript
// src/mount-security.ts
import fs from 'fs';
import path from 'path';

const ALLOWLIST_PATH = path.join(
  process.env.HOME || '',
  '.config',
  'nanoclaw',
  'mount-allowlist.json',
);

interface MountAllowlist {
  allowedRoots: Array<{
    path: string;
    allowReadWrite: boolean;
  }>;
  blockedPatterns: string[];
  nonMainReadOnly: boolean;
}

export function validateMount(
  groupFolder: string,
  hostPath: string,
): { valid: boolean; reason?: string } {
  const allowlist: MountAllowlist = JSON.parse(
    fs.readFileSync(ALLOWLIST_PATH, 'utf-8'),
  );

  // 检查是否在允许的根目录下
  const underAllowedRoot = allowlist.allowedRoots.some((root) => {
    const resolvedRoot = root.path.replace('~', process.env.HOME);
    const resolvedHost = hostPath.replace('~', process.env.HOME);
    return resolvedHost.startsWith(resolvedRoot);
  });

  if (!underAllowedRoot) {
    return { valid: false, reason: 'Path not in allowed roots' };
  }

  // 检查是否匹配阻止模式
  const blocked = allowlist.blockedPatterns.some((pattern) =>
    hostPath.includes(pattern),
  );

  if (blocked) {
    return { valid: false, reason: 'Path matches blocked pattern' };
  }

  return { valid: true };
}
```

### 4.4 验收标准

- [ ] Docker 镜像可成功构建
- [ ] 容器可启动并运行
- [ ] 挂载点在容器内可访问
- [ ] 安全验证可阻止非法挂载
- [ ] 容器超时后自动停止
- [ ] 单元测试覆盖所有边界情况

### 4.5 完成标志

```bash
# 构建镜像
cd container
./build.sh

# 验证镜像
docker images | grep nanoclaw-agent

# 测试运行
docker run --rm nanoclaw-agent:latest node --version

# 运行测试
npm test container-runner.test.ts
```

---

## 5. Phase 3: WhatsApp 集成

### 5.1 目标

实现 WhatsApp 连接、消息收发、认证流程。

### 5.2 任务清单

| 任务 ID | 任务                              | 优先级 | 预估时间 | 依赖           |
| ------- | --------------------------------- | ------ | -------- | -------------- |
| P3-001  | 安装 baileys 依赖                 | P0     | 30 分钟  | -              |
| P3-002  | 定义 Channel 接口                 | P0     | 1 小时   | P1-003         |
| P3-003  | 实现 WhatsApp 连接（whatsapp.ts） | P0     | 6 小时   | P3-002         |
| P3-004  | 实现消息接收和存储                | P0     | 4 小时   | P3-003, P1-006 |
| P3-005  | 实现消息发送                      | P0     | 2 小时   | P3-003         |
| P3-006  | 实现 QR 码认证                    | P0     | 3 小时   | P3-003         |
| P3-007  | 实现连接状态监听                  | P1     | 2 小时   | P3-003         |
| P3-008  | 编写 WhatsApp 集成测试            | P2     | 4 小时   | P3-003-P3-007  |

### 5.3 详细实现

#### P3-002: Channel 接口

```typescript
// src/types.ts
export interface Channel {
  name: string;
  connect(): Promise<void>;
  sendMessage(jid: string, text: string): Promise<void>;
  isConnected(): boolean;
  ownsJid(jid: string): boolean;
  disconnect(): Promise<void>;
  setTyping?(jid: string, isTyping: boolean): Promise<void>;
}

export type OnInboundMessage = (chatJid: string, message: NewMessage) => void;
```

#### P3-003: WhatsApp 连接

```typescript
// src/channels/whatsapp.ts
import makeWASocket from '@whiskeysockets/baileys';

export class WhatsAppChannel implements Channel {
  name = 'whatsapp';
  private socket?: ReturnType<typeof makeWASocket>;
  private onMessage?: OnInboundMessage;

  constructor(private opts: { onMessage: OnInboundMessage }) {}

  async connect(): Promise<void> {
    this.socket = makeWASocket({
      auth: {
        creds: this.loadSession(),
      },
      printQRInTerminal: true,
    });

    this.socket.ev.on('connection.update', async ({ connection }) => {
      if (connection === 'open') {
        console.log('WhatsApp connected');
      }
    });

    this.socket.ev.on('messages.upsert', ({ messages, type }) => {
      if (type !== 'notify') return;

      for (const msg of messages) {
        this.handleMessage(msg);
      }
    });
  }

  async sendMessage(jid: string, text: string): Promise<void> {
    await this.socket?.sendMessage(jid, { text });
  }

  isConnected(): boolean {
    return this.socket?.user !== undefined;
  }

  ownsJid(jid: string): boolean {
    return jid.endsWith('@s.whatsapp.net') || jid.endsWith('@g.us');
  }

  async disconnect(): Promise<void> {
    await this.socket?.logout();
  }

  private handleMessage(msg: any): void {
    const newMessage: NewMessage = {
      id: msg.key.id,
      chat_jid: msg.key.remoteJid,
      sender: msg.key.participant || msg.key.remoteJid,
      sender_name: msg.pushName,
      content: msg.message?.conversation || '',
      timestamp: msg.messageTimestamp.toString(),
      is_from_me: msg.key.fromMe,
    };

    this.opts.onMessage(newMessage.chatJid, newMessage);
  }
}
```

### 5.4 验收标准

- [ ] 可扫描二维码完成认证
- [ ] 消息可成功接收并存储
- [ ] 消息可成功发送到 WhatsApp
- [ ] 连接断开可自动重连
- [ ] 会话可持久化
- [ ] 集成测试覆盖主要流程

### 5.5 完成标志

```bash
# 启动 WhatsApp 连接
npm run auth

# 扫描二维码后验证
# 手机上看到已连接

# 发送测试消息
# 在 WhatsApp 中发送消息给任何联系人

# 查看数据库
sqlite3 store/messages.db "SELECT * FROM messages LIMIT 5"
```

---

## 6. Phase 4: Agent 执行

### 6.1 目标

集成 Claude Agent SDK，实现容器内 agent 执行和工具调用。

### 6.2 任务清单

| 任务 ID | 任务                                     | 优先级 | 预估时间 | 依赖          |
| ------- | ---------------------------------------- | ------ | -------- | ------------- |
| P4-001  | 创建容器代码库（container/agent-runner） | P0     | 2 小时   | -             |
| P4-002  | 实现 Agent 启动器（index.ts）            | P0     | 4 小时   | P4-001        |
| P4-003  | 集成 Claude Agent SDK                    | P0     | 6 小时   | P4-002        |
| P4-004  | 实现查询循环（stdin/stdout）             | P0     | 3 小时   | P4-003        |
| P4-005  | 实现工具调用（MCP）                      | P1     | 4 小时   | P4-003        |
| P4-006  | 实现会话持久化                           | P0     | 2 小时   | P4-003        |
| P4-007  | 实现流式输出处理                         | P1     | 2 小时   | P4-003        |
| P4-008  | 编写 Agent 测试                          | P2     | 3 小时   | P4-003-P4-007 |

### 6.3 详细实现

#### P4-002: Agent 启动器

```typescript
// container/agent-runner/src/index.ts
import { createAgent } from '@anthropic-ai/claude-agent-sdk';

async function main() {
  const agent = await createAgent({
    prompt: process.env.PROMPT || 'Hello, how can I help you today?',
    model: 'claude-sonnet-4-20250214',
    tools: ['@anthropic-ai/claude-code-tools'],
  });

  for await (const { text, usage } of agent.messages()) {
    console.log(JSON.stringify({ type: 'result', result: text }));
  }
}

main().catch(console.error);
```

#### P4-006: 会话持久化

```typescript
// container/agent-runner/src/session.ts
import fs from 'fs';
import path from 'path';

const CLAUDE_DIR = path.join(process.env.HOME || '/home/node', '.claude');

export function saveSession(sessionId: string, transcript: any[]): void {
  const sessionDir = path.join(CLAUDE_DIR, sessionId);
  fs.mkdirSync(sessionDir, { recursive: true });

  const transcriptPath = path.join(sessionDir, 'transcript.jsonl');
  const lines = transcript.map(JSON.stringify).join('\n') + '\n';
  fs.appendFileSync(transcriptPath, lines);
}

export function loadSession(sessionId: string): any[] {
  const transcriptPath = path.join(CLAUDE_DIR, sessionId, 'transcript.jsonl');

  if (!fs.existsSync(transcriptPath)) {
    return [];
  }

  const content = fs.readFileSync(transcriptPath, 'utf-8');
  return content
    .split('\n')
    .filter((line) => line.trim())
    .map(JSON.parse);
}
```

### 6.4 验收标准

- [ ] Agent 可在容器内启动
- [ ] 可接收输入并输出结果
- [ ] 会话可持久化和恢复
- [ ] 工具调用正常工作
- [ ] 流式输出可实时返回
- [ ] 错误可正确捕获和返回

### 6.5 完成标志

```bash
# 构建 agent-runner
cd container/agent-runner
npm run build

# 测试运行
docker run --rm \
  -v $(pwd)/dist:/app \
  -e PROMPT="test" \
  nanoclaw-agent:latest

# 验证输出
# 应看到 JSON 格式的结果流
```

---

## 7. Phase 5: 任务调度

### 7.1 目标

实现定时任务系统：Cron 解析、任务执行、历史记录。

### 7.2 任务清单

| 任务 ID | 任务                                | 优先级 | 预估时间 | 依赖           |
| ------- | ----------------------------------- | ------ | -------- | -------------- |
| P5-001  | 创建 scheduled_tasks 表             | P0     | 1 小时   | P1-004         |
| P5-002  | 创建 task_run_logs 表               | P1     | 1 小时   | P5-001         |
| P5-003  | 实现 Cron 表达式解析                | P0     | 2 小时   | -              |
| P5-004  | 实现任务调度器（task-scheduler.ts） | P0     | 6 小时   | P5-001, P5-003 |
| P5-005  | 实现任务执行                        | P0     | 3 小时   | P5-004, P2-002 |
| P5-006  | 实现任务管理 API                    | P1     | 2 小时   | P5-001         |
| P5-007  | 编写调度器测试                      | P2     | 3 小时   | P5-004-P5-006  |

### 7.3 详细实现

#### P5-003: Cron 解析

```typescript
// src/task-scheduler.ts
import { CronExpressionParser } from 'cron-parser';

function getNextRunTime(cronExpr: string, timezone: string): Date | null {
  try {
    const interval = CronExpressionParser.parse(cronExpr, { tz: timezone });
    return interval.next().toDate();
  } catch (err) {
    console.error(`Invalid cron expression: ${cronExpr}`);
    return null;
  }
}

// 使用示例
const nextRun = getNextRunTime('0 9 * * 1', 'America/New_York');
// 下周一上午 9 点
```

#### P5-004: 任务调度器

```typescript
// src/task-scheduler.ts
import { runContainerAgent } from './container-runner.js';

let schedulerRunning = false;

export function startSchedulerLoop(): void {
  if (schedulerRunning) return;
  schedulerRunning = true;

  const loop = async () => {
    try {
      const dueTasks = getDueTasks();

      for (const task of dueTasks) {
        await executeTask(task);
      }
    } catch (err) {
      console.error('Scheduler error:', err);
    }

    setTimeout(loop, SCHEDULER_POLL_INTERVAL);
  };

  loop();
}

async function executeTask(task: ScheduledTask): Promise<void> {
  const startTime = Date.now();

  try {
    const output = await runContainerAgent(
      {
        prompt: task.prompt,
        sessionId: undefined, // 新会话
        groupFolder: task.group_folder,
      },
      (proc, containerId) => {},
      (streamedOutput) => {
        // 处理输出
      },
    );

    const duration = Date.now() - startTime;
    logTaskRun({
      task_id: task.id,
      run_at: new Date().toISOString(),
      duration_ms: duration,
      status: 'success',
      result: output.result,
      error: null,
    });

    // 计算下次运行
    const nextRun = calculateNextRun(task);
    updateTaskAfterRun(task.id, nextRun, output.result);
  } catch (err) {
    logTaskRun({
      task_id: task.id,
      run_at: new Date().toISOString(),
      duration_ms: Date.now() - startTime,
      status: 'error',
      result: null,
      error: err.message,
    });
  }
}
```

### 7.4 验收标准

- [ ] Cron 表达式可正确解析
- [ ] 到期任务可自动执行
- [ ] 任务执行可记录历史
- [ ] 任务状态可更新（暂停/恢复/取消）
- [ ] 调度器循环可持续运行
- [ ] 测试覆盖所有任务类型（cron/interval/once）

### 7.5 完成标志

```bash
# 创建测试任务
sqlite3 store/messages.db "
INSERT INTO scheduled_tasks
  (id, group_folder, chat_jid, prompt, schedule_type, schedule_value, next_run, status, created_at)
VALUES
  ('task-test', 'main', 'self@g.us', 'Test task', 'interval', '60000',
   datetime('now', '+1 minute'), 'active', datetime('now'))
"

# 启动调度器
npm run dev

# 等待 1 分钟，检查执行
sqlite3 store/messages.db "SELECT * FROM task_run_logs"
# 应看到一条执行记录
```

---

## 8. Phase 6: IPC 通信

### 8.1 目标

实现基于文件系统的 IPC，支持 Agent 向主进程发送消息和创建任务。

### 8.2 任务清单

| 任务 ID | 任务                          | 优先级 | 预估时间 | 依赖           |
| ------- | ----------------------------- | ------ | -------- | -------------- |
| P6-001  | 设计 IPC 文件结构             | P0     | 1 小时   | -              |
| P6-002  | 实现 IPC 文件监听器（ipc.ts） | P0     | 4 小时   | P6-001         |
| P6-003  | 实现 send_message 处理        | P0     | 2 小时   | P6-002         |
| P6-004  | 实现 schedule_task 处理       | P0     | 3 小时   | P6-002, P5-001 |
| P6-005  | 实现权限验证                  | P0     | 2 小时   | P6-002         |
| P6-006  | 编写 IPC 测试                 | P2     | 3 小时   | P6-002-P6-005  |

### 8.3 详细实现

#### P6-001: IPC 文件结构

```
data/ipc/
├── {group-folder}/
│   ├── messages/
│   │   ├── {timestamp}-{random}.json
│   │   └── _close
│   └── tasks/
│       └── {timestamp}-{random}.json
└── errors/
    └── {group-folder}-{filename}.json
```

#### P6-002: IPC 监听器

```typescript
// src/ipc.ts
import fs from 'fs';
import path from 'path';

const IPC_POLL_INTERVAL = 1000;

export interface IpcDeps {
  sendMessage: (jid: string, text: string) => Promise<void>;
  registeredGroups: () => Record<string, RegisteredGroup>;
}

export function startIpcWatcher(deps: IpcDeps): void {
  const ipcBaseDir = path.join(DATA_DIR, 'ipc');

  const processIpcFiles = async () => {
    const groupFolders = fs.readdirSync(ipcBaseDir);

    for (const groupFolder of groupFolders) {
      await processMessages(groupFolder, deps);
      await processTasks(groupFolder, deps);
    }

    setTimeout(processIpcFiles, IPC_POLL_INTERVAL);
  };

  processIpcFiles();
}

async function processMessages(
  groupFolder: string,
  deps: IpcDeps,
): Promise<void> {
  const messagesDir = path.join(DATA_DIR, 'ipc', groupFolder, 'messages');

  if (!fs.existsSync(messagesDir)) return;

  const messageFiles = fs.readdirSync(messagesDir);

  for (const file of messageFiles) {
    const filePath = path.join(messagesDir, file);
    const data = JSON.parse(fs.readFileSync(filePath, 'utf-8'));

    if (data.type === 'message') {
      await deps.sendMessage(data.chatJid, data.text);
      fs.unlinkSync(filePath);
    }
  }
}
```

### 8.4 验收标准

- [ ] IPC 文件可正确监听和解析
- [ ] Agent 可通过 IPC 发送消息
- [ ] Agent 可通过 IPC 创建任务
- [ ] 权限验证可阻止未授权操作
- [ ] 错误文件可正确保存失败的 IPC
- [ ] 测试覆盖所有 IPC 操作类型

### 8.5 完成标志

```bash
# 启动主进程
npm run dev

# 在另一个终端，创建测试 IPC 文件
echo '{"type":"message","chatJid":"test@g.us","text":"Hello"}' \
  > data/ipc/main/messages/test-$(date +%s).json

# 等待 1 秒，检查 WhatsApp
# 应该收到 "Hello" 消息
```

---

## 9. Phase 7: 多队列管理

### 9.1 目标

实现按组消息队列，支持并发控制和容器复用。

### 9.2 任务清单

| 任务 ID | 任务                                 | 优先级 | 预估时间 | 依赖          |
| ------- | ------------------------------------ | ------ | -------- | ------------- |
| P7-001  | 设计队列数据结构                     | P0     | 1 小时   | -             |
| P7-002  | 实现 GroupQueue 类（group-queue.ts） | P0     | 6 小时   | P7-001        |
| P7-003  | 实现消息入队逻辑                     | P0     | 2 小时   | P7-002        |
| P7-004  | 实现任务入队逻辑                     | P0     | 2 小时   | P7-002        |
| P7-005  | 实现并发控制                         | P0     | 2 小时   | P7-002        |
| P7-006  | 实现容器复用                         | P1     | 2 小时   | P7-002        |
| P7-007  | 实现重试机制                         | P1     | 2 小时   | P7-002        |
| P7-008  | 编写队列测试                         | P2     | 3 小时   | P7-002-P7-007 |

### 9.3 详细实现

#### P7-002: GroupQueue 类

```typescript
// src/group-queue.ts
interface QueuedTask {
  id: string;
  groupJid: string;
  fn: () => Promise<void>;
}

interface GroupState {
  active: boolean;
  idleWaiting: boolean;
  isTaskContainer: boolean;
  pendingMessages: boolean;
  pendingTasks: QueuedTask[];
  process: ChildProcess | null;
  containerName: string | null;
  groupFolder: string | null;
}

export class GroupQueue {
  private groups = new Map<string, GroupState>();
  private activeCount = 0;
  private waitingGroups: string[] = [];
  private processMessagesFn: ((groupJid: string) => Promise<boolean>) | null =
    null;

  setProcessMessagesFn(fn: (groupJid: string) => Promise<boolean>): void {
    this.processMessagesFn = fn;
  }

  enqueueMessageCheck(groupJid: string): void {
    const state = this.getGroup(groupJid);

    if (state.active) {
      state.pendingMessages = true;
      return;
    }

    if (this.activeCount >= MAX_CONCURRENT_CONTAINERS) {
      state.pendingMessages = true;
      this.waitingGroups.push(groupJid);
      return;
    }

    this.runForGroup(groupJid, 'messages');
  }

  sendMessage(groupJid: string, text: string): boolean {
    const state = this.getGroup(groupJid);
    if (!state.active || !state.groupFolder) return false;

    // 写入 IPC 文件
    const inputDir = path.join(DATA_DIR, 'ipc', state.groupFolder, 'input');
    fs.mkdirSync(inputDir, { recursive: true });
    fs.writeFileSync(
      path.join(inputDir, `${Date.now()}.json`),
      JSON.stringify({ type: 'message', text }),
    );

    return true;
  }

  async runForGroup(groupJid: string, reason: string): Promise<void> {
    const state = this.getGroup(groupJid);
    state.active = true;
    this.activeCount++;

    try {
      if (this.processMessagesFn) {
        const success = await this.processMessagesFn(groupJid);
        if (success) {
          state.retryCount = 0;
        }
      }
    } finally {
      state.active = false;
      this.activeCount--;
      this.drainGroup(groupJid);
    }
  }

  private drainGroup(groupJid: string): void {
    // 排队等待的任务
    const state = this.getGroup(groupJid);

    if (state.pendingTasks.length > 0) {
      const task = state.pendingTasks.shift()!;
      this.runTask(groupJid, task);
      return;
    }

    if (state.pendingMessages) {
      this.runForGroup(groupJid, 'drain');
      return;
    }

    this.drainWaiting();
  }
}
```

### 9.4 验收标准

- [ ] 消息可按组排队
- [ ] 并发数可正确限制
- [ ] 活跃容器可接收新消息
- [ ] 任务优先级可正确处理
- [ ] 失败任务可自动重试
- [ ] 测试覆盖并发和边缘情况

### 9.5 完成标志

```bash
# 测试并发控制
# 设置 MAX_CONCURRENT_CONTAINERS=2
export MAX_CONCURRENT_CONTAINERS=2

# 快速发送 5 个组的消息
# 验证只有 2 个容器在运行

# 等待一个完成后
# 第 3 个容器应启动

# 检查队列状态
# 应看到等待的组被正确处理
```

---

## 10. Phase 8: 生产化

### 10.1 目标

实现系统服务、日志监控、错误处理，支持生产环境部署。

### 10.2 任务清单

| 任务 ID | 任务                   | 优先级 | 预估时间 | 依赖          |
| ------- | ---------------------- | ------ | -------- | ------------- |
| P8-001  | 实现 graceful shutdown | P0     | 2 小时   | -             |
| P8-002  | 完善日志和错误处理     | P0     | 3 小时   | P8-001        |
| P8-003  | 创建 launchd 配置      | P0     | 2 小时   | -             |
| P8-004  | 创建 systemd 配置      | P1     | 2 小时   | -             |
| P8-005  | 实现健康检查           | P1     | 2 小时   | -             |
| P8-006  | 编写部署文档           | P2     | 2 小时   | P8-003-P8-004 |

### 10.3 详细实现

#### P8-001: Graceful Shutdown

```typescript
// src/index.ts
async function main(): Promise<void> {
  // ... 初始化代码

  const shutdown = async (signal: string) => {
    console.log(`${signal} received, shutting down...`);

    // 停止消息循环
    await queue.shutdown(10000);

    // 断开所有通道
    for (const ch of channels) {
      await ch.disconnect();
    }

    process.exit(0);
  };

  process.on('SIGTERM', () => shutdown('SIGTERM'));
  process.on('SIGINT', () => shutdown('SIGINT'));
}
```

#### P8-003: Launchd 配置

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>com.nanoclaw</string>

    <key>ProgramArguments</key>
    <array>
        <string>{{NODE_PATH}}</string>
        <string>{{PROJECT_ROOT}}/dist/index.js</string>
    </array>

    <key>WorkingDirectory</key>
    <string>{{PROJECT_ROOT}}</string>

    <key>RunAtLoad</key>
    <true/>

    <key>KeepAlive</key>
    <true/>

    <key>StandardOutPath</key>
    <string>{{PROJECT_ROOT}}/logs/nanoclaw.log</string>

    <key>StandardErrorPath</key>
    <string>{{PROJECT_ROOT}}/logs/nanoclaw.error.log</string>
</dict>
</plist>
```

### 10.4 验收标准

- [ ] 服务可通过 SIGTERM/SIGINT 优雅关闭
- [ ] 错误可正确记录到日志文件
- [ ] launchd 可成功加载和启动服务
- [ ] systemd 可成功加载和启动服务
- [ ] 健康检查可正确报告状态
- [ ] 部署文档完整可执行

### 10.5 完成标志

```bash
# 启动服务
launchctl load ~/Library/LaunchAgents/com.nanoclaw.plist
launchctl start com.nanoclaw

# 验证运行
launchctl list | grep nanoclaw
# 应看到 "com.nanoclaw" 状态

# 检查日志
tail -f logs/nanoclaw.log
# 应看到正常启动日志

# 测试关闭
launchctl stop com.nanoclaw
# 应看到优雅关闭日志
```

---

## 11. 跨阶段整合

### 11.1 主入口（index.ts）

将所有模块整合到单一主进程：

```typescript
// src/index.ts
async function main(): Promise<void> {
  // 1. 初始化
  ensureContainerRuntimeRunning();
  initDatabase();
  logger.info('NanoClaw starting...');

  // 2. 加载状态
  loadState();

  // 3. 注册 graceful shutdown
  const shutdown = async (signal: string) => {
    logger.info({ signal }, 'Shutdown signal received');
    await queue.shutdown(10000);
    for (const ch of channels) await ch.disconnect();
    process.exit(0);
  };
  process.on('SIGTERM', () => shutdown('SIGTERM'));
  process.on('SIGINT', () => shutdown('SIGINT'));

  // 4. 创建通道
  whatsapp = new WhatsAppChannel({
    onMessage: (chatJid, msg) => storeMessage(msg),
    onChatMetadata: (jid, ts, name, ch, isGroup) =>
      storeChatMetadata(jid, ts, name, ch, isGroup),
    registeredGroups: () => registeredGroups,
  });
  channels.push(whatsapp);
  await whatsapp.connect();

  // 5. 启动子系统
  queue.setProcessMessagesFn(processGroupMessages);
  startSchedulerLoop({
    registeredGroups: () => registeredGroups,
    getSessions: () => sessions,
    queue,
    onProcess: (jid, proc, name, folder) =>
      queue.registerProcess(jid, proc, name, folder),
    sendMessage: async (jid, text) => {
      const ch = findChannel(channels, jid);
      if (ch) await ch.sendMessage(jid, text);
    },
  });
  startIpcWatcher({
    sendMessage: (jid, text) => {
      const ch = findChannel(channels, jid);
      if (!ch) throw new Error(`No channel for ${jid}`);
      return ch.sendMessage(jid, text);
    },
    registeredGroups: () => registeredGroups,
    registerGroup,
    syncGroupMetadata: (force) => whatsapp.syncGroupMetadata(force),
    getAvailableGroups,
    writeGroupsSnapshot,
  });

  recoverPendingMessages();
  startMessageLoop().catch((err) => {
    logger.fatal({ err }, 'Message loop crashed');
    process.exit(1);
  });
}

main().catch((err) => {
  logger.error({ err }, 'Failed to start NanoClaw');
  process.exit(1);
});
```

### 11.2 集成验收

- [ ] 所有模块可正常协作
- [ ] 消息可从 WhatsApp 流转到 Agent
- [ ] Agent 响应可流式返回 WhatsApp
- [ ] 定时任务可自动执行
- [ ] 并发控制可正常工作
- [ ] 服务可持续稳定运行

---

## 12. 测试策略

### 12.1 测试金字塔

| 测试类型   | 比例 | 工具         | 覆盖率目标 |
| ---------- | ---- | ------------ | ---------- |
| 单元测试   | 300+ | Vitest       | > 90%      |
| 集成测试   | 50+  | Vitest       | > 80%      |
| 端到端测试 | 20+  | 手动/自动化  | -          |
| 性能测试   | 5+   | Docker Stats | -          |

### 12.2 测试场景

#### 单元测试场景

- 数据库 CRUD 操作
- Cron 表达式解析
- 挂载安全验证
- 队列入队/出队
- IPC 文件解析

#### 集成测试场景

- WhatsApp 消息接收 → 存储 → 路由 → 容器启动
- 容器启动 → Agent 执行 → 流式输出 → 返回 WhatsApp
- 定时任务创建 → 调度 → 执行 → 结果记录

#### 端到端测试场景

```
场景 1: 基础对话
1. 用户发送: "@Andy 你好"
2. 验证: 收到 "Andy: 你好！有什么我可以帮助你的吗？"

场景 2: 定时任务
1. 用户发送: "@Andy 每天上午 9 点提醒我喝水"
2. 验证: 次日 9 点收到提醒

场景 3: 并发限制
1. 快速在 5 个组发送 "@Andy test"
2. 验证: 只有 MAX_CONCURRENT_CONTAINERS 个容器运行
3. 验证: 其他消息正确排队

场景 4: 容器复用
1. 组 A 发送: "@Andy 1"
2. 等待响应
3. 立即发送: "@Andy 2"
4. 验证: 使用同一容器，无重启

场景 5: Graceful Shutdown
1. 运行中的服务
2. 发送 SIGTERM
3. 验证: 所有消息处理完成
4. 验证: 容器正确停止
5. 验证: WhatsApp 正常断开
```

---

## 13. 性能优化

### 13.1 优化目标

| 指标           | 当前目标 | 优化后目标       |
| -------------- | -------- | ---------------- |
| 容器启动时间   | -        | < 5 秒           |
| 消息端到端延迟 | -        | < 3 秒           |
| 数据库查询     | -        | < 100 ms         |
| 内存占用       | -        | < 500 MB (空闲） |
| CPU 占用       | -        | < 20% (空闲）    |

### 13.2 优化措施

| 措施            | 实施 Phase | 预期效果          |
| --------------- | ---------- | ----------------- |
| SQLite WAL 模式 | Phase 1    | +50% 并发写入     |
| 连接池          | Phase 2    | 减少连接开销      |
| 容器复用        | Phase 7    | 减少 80% 启动开销 |
| 索引优化        | Phase 1    | +70% 查询速度     |
| 日志异步写入    | Phase 8    | 减少 I/O 阻塞     |

---

## 14. 风险与缓解

### 14.1 技术风险

| 风险              | 影响 | 概率 | 缓解措施                  |
| ----------------- | ---- | ---- | ------------------------- |
| Docker 兼容性问题 | 高   | 低   | 支持 Apple Container 备选 |
| baileys API 变更  | 高   | 中   | 版本锁定，监控更新        |
| SQLite 并发锁     | 中   | 低   | WAL 模式，重试机制        |
| 容器资源泄漏      | 中   | 低   | 超时保护，定期清理        |

### 14.2 进度风险

| 风险         | 影响 | 概率 | 缓解措施           |
| ------------ | ---- | ---- | ------------------ |
| 技术债务积累 | 高   | 中   | 持续重构，代码审查 |
| 需求变更     | 中   | 低   | 敏捷迭代，小步快跑 |
| 测试覆盖不足 | 中   | 中   | TDD，CI/CD         |

---

## 15. 里程碑检查点

### Milestone 1: MVP（最小可行产品）✅

**完成条件**：

- [x] Phase 1-4 完成
- [x] 可通过 WhatsApp 对话
- [x] Agent 可执行基本请求

**预期时间**：8 周
**实际完成**：**\_**

### Milestone 2: 功能完整 ✅

**完成条件**：

- [x] Phase 5-7 完成
- [x] 定时任务可用
- [x] IPC 通信可用

**预期时间**：14 周
**实际完成**：**\_**

### Milestone 3: 生产就绪 ✅

**完成条件**：

- [x] Phase 8 完成
- [x] 服务可稳定运行
- [x] 文档完整

**预期时间**：17 周
**实际完成**：**\_**

---

## 16. 交付物清单

### 16.1 代码交付

- [ ] 完整的 TypeScript 源代码
- [ ] 单元测试（>90% 覆盖率）
- [ ] 集成测试
- [ ] CI/CD 配置（可选）

### 16.2 配置交付

- [ ] tsconfig.json
- [ ] package.json
- [ ] .env.example
- [ ] launchd plist
- [ ] systemd service

### 16.3 文档交付

- [x] README.md
- [x] 开发指南（DEVELOPMENT.md）
- [x] 数据模型（DATA_MODEL.md）
- [x] 用户指南（USER_GUIDE.md）
- [x] API 文档（可选）

### 16.4 部署交付

- [ ] Dockerfile
- [ ] Docker 镜像
- [ ] 构建脚本
- [ ] 部署脚本

---

## 17. 后续计划

### 17.1 功能扩展

| 功能          | 优先级 | 预估时间 |
| ------------- | ------ | -------- |
| Telegram 集成 | P1     | 2 周     |
| Gmail 集成    | P2     | 2 周     |
| 语音转录      | P2     | 1 周     |
| X 集成        | P3     | 2 周     |
| Agent Swarm   | P2     | 3 周     |

### 17.2 性能优化

| 优化       | 预估时间 |
| ---------- | -------- |
| 多进程架构 | 4 周     |
| 缓存层     | 2 周     |
| 数据库分区 | 2 周     |

---

**文档维护**: 开发团队
**最后更新**: 2025-02-24
