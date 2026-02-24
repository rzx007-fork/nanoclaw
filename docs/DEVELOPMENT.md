# NanoClaw 开发指南

| 文档版本 | 日期       | 作者     | 变更说明 |
| -------- | ---------- | -------- | -------- |
| v1.0     | 2025-02-24 | 开发团队 | 初始版本 |

---

## 1. 概述

### 1.1 文档目的

本文档为 NanoClaw 贡献者提供开发环境搭建、代码规范、测试指南等信息，帮助开发者快速上手项目。

### 1.2 代码哲学

NanoClaw 遵循以下开发原则：

| 原则         | 说明                                         |
| ------------ | -------------------------------------------- |
| **极简主义** | 代码库保持在 10,000 行以内，拒绝不必要的抽象 |
| **可理解性** | 任何有经验的开发者应在 8 分钟内理解整体架构  |
| **直接修改** | 不添加配置文件，直接修改代码实现需求         |
| **技能扩展** | 新功能通过技能（Skills）而非直接修改代码库   |

---

## 2. 开发环境

### 2.1 系统要求

| 组件       | 最低版本                  | 推荐版本                  |
| ---------- | ------------------------- | ------------------------- |
| 操作系统   | macOS 12+ / Ubuntu 20.04+ | macOS 14+ / Ubuntu 22.04+ |
| Node.js    | 20.0                      | 20.10+                    |
| TypeScript | 5.0                       | 5.7+                      |
| Docker     | 24.0                      | 24.0+                     |
| 构建工具   | Xcode 14+ / gcc 9+        | Xcode 15+ / gcc 11+       |

### 2.2 环境搭建

#### 步骤 1: 克隆仓库

```bash
git clone https://github.com/qwibitai/nanoclaw.git
cd nanoclaw
```

#### 步骤 2: 安装依赖

```bash
npm install
```

#### 步骤 3: 构建项目

```bash
npm run build
```

#### 步骤 4: 验证环境

```bash
# 运行测试
npm test

# 类型检查
npm run typecheck

# 代码格式检查
npm run format:check
```

### 2.3 开发工具

#### 推荐工具

| 工具        | 用途        | 版本   |
| ----------- | ----------- | ------ |
| VS Code     | IDE         | 最新版 |
| Claude Code | AI 协作开发 | 最新版 |
| ts-node-dev | 热重载      | 2.0+   |

#### VS Code 扩展

```json
{
  "recommendations": [
    "biomejs.biome",
    "dbaeumer.vscode-eslint",
    "ms-vscode.vscode-typescript-next"
  ]
}
```

---

## 3. 项目结构

### 3.1 目录树

```
nanoclaw/
├── src/                      # 源代码
│   ├── channels/             # 消息通道
│   │   └── whatsapp.ts      # WhatsApp 实现
│   ├── container-runner.ts   # 容器执行引擎
│   ├── container-runtime.ts  # 容器运行时
│   ├── db.ts                # 数据库操作
│   ├── group-queue.ts       # 消息队列
│   ├── group-folder.ts      # 组文件夹管理
│   ├── index.ts             # 主入口
│   ├── ipc.ts               # IPC 通信
│   ├── logger.ts            # 日志系统
│   ├── mount-security.ts    # 挂载安全
│   ├── task-scheduler.ts    # 任务调度
│   ├── types.ts             # 类型定义
│   ├── router.ts            # 消息路由
│   ├── config.ts            # 全局配置
│   └── env.ts              # 环境变量
│
├── container/               # 容器相关
│   ├── Dockerfile          # Docker 镜像
│   ├── build.sh           # 构建脚本
│   ├── agent-runner/       # Agent 启动器
│   └── skills/            # 容器内技能
│
├── setup/                  # 安装脚本
│   ├── environment.ts     # 环境检查
│   ├── container.ts       # 容器设置
│   ├── service.ts        # 服务配置
│   └── ...
│
├── groups/                 # 组文件夹（运行时生成）
│   ├── main/             # 主通道
│   ├── project-team/      # 项目团队群
│   └── {name}/           # 其他群组
│
├── store/                  # 数据存储
│   ├── auth/             # 认证文件
│   └── messages.db       # SQLite 数据库
│
├── data/                   # 运行时数据
│   └── ipc/              # IPC 文件
│
├── .claude/skills/         # Claude Code 技能
│   ├── setup/            # 初始设置
│   ├── add-telegram/     # Telegram 集成
│   └── ...
│
├── docs/                   # 文档
├── tests/                  # 测试文件
└── package.json
```

### 3.2 核心模块说明

| 模块       | 文件                    | 职责                             |
| ---------- | ----------------------- | -------------------------------- |
| **主入口** | src/index.ts            | 消息循环协调、状态管理、容器调度 |
| **通道层** | src/channels/           | 消息通道抽象和实现               |
| **队列层** | src/group-queue.ts      | 按组消息队列、并发控制           |
| **容器层** | src/container-runner.ts | 容器启动、挂载、流式输出         |
| **调度层** | src/task-scheduler.ts   | 定时任务执行                     |
| **通信层** | src/ipc.ts              | Agent ↔ 主进程 IPC               |
| **数据层** | src/db.ts               | SQLite 操作                      |
| **路由层** | src/router.ts           | 消息格式化和路由                 |

---

## 4. 代码规范

### 4.1 命名约定

| 类型     | 约定             | 示例                        |
| -------- | ---------------- | --------------------------- |
| 文件     | kebab-case       | `container-runner.ts`       |
| 类       | PascalCase       | `GroupQueue`                |
| 函数     | camelCase        | `sendMessage()`             |
| 常量     | UPPER_SNAKE_CASE | `MAX_CONCURRENT_CONTAINERS` |
| 私有成员 | 前缀 `_`         | `_initTestDatabase()`       |
| 接口     | PascalCase       | `Channel`                   |
| 类型别名 | PascalCase       | `NewMessage`                |

### 4.2 代码组织

#### 文件结构

```typescript
// 1. 导入（标准库 → 外部依赖 → 内部模块）
import fs from 'fs';
import path from 'path';
import Database from 'better-sqlite3';
import { logger } from './logger.js';

// 2. 类型定义
interface Config {
  name: string;
  timeout: number;
}

// 3. 常量
const DEFAULT_TIMEOUT = 30000;

// 4. 私有函数
function validateConfig(config: Config): boolean {
  return config.timeout > 0;
}

// 5. 公共函数/类
export function runAgent(config: Config): void {
  if (!validateConfig(config)) {
    throw new Error('Invalid config');
  }
  // ...
}
```

#### 函数复杂度

- 单个函数不超过 50 行
- 圈复杂度不超过 10
- 单个文件不超过 500 行

### 4.3 注释规范

```typescript
/**
 * 在容器中运行 Claude Agent
 *
 * @param group - 注册的群组信息
 * @param prompt - 发送给 Agent 的提示词
 * @param onOutput - 流式输出回调
 * @returns 返回状态 'success' 或 'error'
 *
 * @throws {Error} 容器启动失败时抛出异常
 */
export async function runAgent(
  group: RegisteredGroup,
  prompt: string,
  onOutput?: (output: ContainerOutput) => Promise<void>,
): Promise<'success' | 'error'> {
  // 实现
}
```

### 4.4 错误处理

```typescript
// 好的做法：明确错误类型
try {
  const db = new Database(dbPath);
  return db.prepare('SELECT * FROM tasks').all();
} catch (err) {
  const message = err instanceof Error ? err.message : String(err);
  logger.error({ err }, 'Failed to query database');
  throw new Error(`Database query failed: ${message}`);
}

// 避免：吞掉错误
try {
  doSomething();
} catch {
  // silent failure
}
```

### 4.5 日志规范

```typescript
import { logger } from './logger.js';

// 信息日志
logger.info({ taskId, duration }, 'Task completed');

// 警告日志
logger.warn({ jid, folder }, 'Invalid group folder');

// 错误日志
logger.error({ err, taskId }, 'Task failed');

// 调试日志
logger.debug({ groupJid, count }, 'Queued messages');
```

---

## 5. 测试指南

### 5.1 测试框架

- **单元测试**: Vitest
- **覆盖率**: @vitest/coverage-v8
- **Mock**: Vitest 内置

### 5.2 测试结构

```
src/
├── db.ts
├── db.test.ts           # 单元测试
├── container-runner.ts
└── container-runner.test.ts
```

### 5.3 测试示例

```typescript
// src/db.test.ts
import { describe, it, expect, beforeEach } from 'vitest';
import { initDatabase, storeMessage, getMessagesSince } from './db.js';
import { _initTestDatabase } from './db.js';

describe('Database Operations', () => {
  beforeEach(() => {
    // 使用内存数据库
    _initTestDatabase();
  });

  it('should store and retrieve messages', () => {
    const msg = {
      id: 'msg-1',
      chat_jid: 'test@g.us',
      sender: 'user@s.whatsapp.net',
      sender_name: 'Test User',
      content: 'Hello',
      timestamp: '2025-02-24T10:00:00Z',
    };

    storeMessage(msg);
    const retrieved = getMessagesSince(
      'test@g.us',
      '2025-02-24T09:00:00Z',
      'Andy',
    );

    expect(retrieved).toHaveLength(1);
    expect(retrieved[0].content).toBe('Hello');
  });
});
```

### 5.4 运行测试

```bash
# 运行所有测试
npm test

# 监听模式
npm run test:watch

# 生成覆盖率报告
npm test -- --coverage

# 运行特定测试文件
npm test db.test.ts

# 匹配测试名称
npm test -- --grep "should store message"
```

### 5.5 测试覆盖率要求

| 模块     | 最低覆盖率 |
| -------- | ---------- |
| 核心逻辑 | 90%+       |
| 数据层   | 95%+       |
| 工具函数 | 100%       |
| 集成测试 | 80%+       |

---

## 6. 调试指南

### 6.1 本地运行

```bash
# 开发模式（热重载）
npm run dev

# 构建后运行
npm run build
npm start
```

### 6.2 日志查看

```bash
# 实时日志
tail -f logs/nanoclaw.log

# 错误日志
tail -f logs/nanoclaw.error.log

# 设置日志级别（编辑 src/logger.ts）
const level = process.env.LOG_LEVEL || 'debug';
```

### 6.3 容器调试

```bash
# 查看运行中的容器
docker ps

# 进入容器
docker exec -it <container_id> /bin/bash

# 查看容器日志
docker logs <container_id>

# 清理所有容器
docker system prune -a
```

### 6.4 数据库调试

```bash
# 安装 SQLite 命令行工具
sqlite3 store/messages.db

# 常用查询
sqlite3> .schema messages
sqlite3> SELECT * FROM messages LIMIT 10;
sqlite3> SELECT COUNT(*) FROM messages;
```

---

## 7. 提交规范

### 7.1 分支模型

```
main         # 稳定版本
  └─ feature/xxx    # 功能分支
  └─ fix/xxx        # Bug 修复
  └─ docs/xxx       # 文档更新
```

### 7.2 提交消息格式

```
<type>(<scope>): <subject>

[optional body]

[optional footer]
```

| 类型       | 说明          |
| ---------- | ------------- |
| `feat`     | 新功能        |
| `fix`      | Bug 修复      |
| `docs`     | 文档更新      |
| `refactor` | 代码重构      |
| `test`     | 测试相关      |
| `chore`    | 构建/工具相关 |

### 7.3 提交示例

```bash
# Bug 修复
git commit -m "fix(group-queue): prevent deadlock when container timeout"

# 功能添加
git commit -m "feat(container-runner): add Apple Container runtime support"

# 文档更新
git commit -m "docs(readme): add troubleshooting section"
```

### 7.4 PR 流程

1. Fork 仓库
2. 创建特性分支
3. 编写代码和测试
4. 运行 `npm run typecheck` 和 `npm test`
5. 提交 PR，描述变更
6. 等待 Code Review
7. 根据反馈修改
8. 合并到 main

---

## 8. 性能优化

### 8.1 优化检查点

| 指标         | 目标值       | 检查方式           |
| ------------ | ------------ | ------------------ |
| 容器启动时间 | < 5 秒       | `docker time`      |
| 数据库查询   | < 100ms      | EXPLAIN QUERY PLAN |
| 内存使用     | < 500MB      | `docker stats`     |
| CPU 使用率   | < 20% (空闲) | `docker stats`     |

### 8.2 常见优化技巧

```typescript
// 1. 批量操作
// 不好
for (const msg of messages) {
  storeMessage(msg);
}
// 好
const stmt = db.prepare(`INSERT INTO messages VALUES (...)`);
for (const msg of messages) {
  stmt.run(msg);
}

// 2. 索引优化
CREATE INDEX idx_timestamp ON messages(timestamp);

// 3. 连接池
// better-sqlite3 是同步的，不需要连接池

// 4. 缓存
const cached = cache.get(key);
if (cached) return cached;
const result = computeResult();
cache.set(key, result);
return result;
```

---

## 9. 安全实践

### 9.1 敏感信息

```typescript
// ❌ 不要这样做
const password = process.env.DB_PASSWORD;
console.log('Password:', password);

// ✅ 应该这样做
import { readEnvFile } from './env.js';
const secrets = readSecretsFile(); // 从磁盘读取
// secrets 不传递给子进程
```

### 9.2 输入验证

```typescript
// 验证组文件夹名称
function isValidGroupFolder(folder: string): boolean {
  // 防止路径遍历攻击
  const sanitized = path.normalize(folder);
  return (
    sanitized === folder &&
    !sanitized.includes('..') &&
    !path.isAbsolute(sanitized) &&
    !sanitized.startsWith('.')
  );
}
```

### 9.3 权限检查

```typescript
// IPC 操作的权限验证
if (!isMain && targetFolder !== sourceGroup) {
  logger.warn({ sourceGroup, targetFolder }, 'Unauthorized access blocked');
  throw new Error('Permission denied');
}
```

---

## 10. 技能开发

### 10.1 技能结构

```
.claude/skills/add-telegram/
└── SKILL.md    # 技能描述（Markdown）
```

### 10.2 技能编写模板

```markdown
# Add Telegram Support

## Overview

Add Telegram as a messaging channel to NanoClaw.

## Prerequisites

- Telegram Bot API token
- Docker running

## Steps

1. Install dependencies
   Read package.json, add `node-telegram-bot-api`, run `npm install`

2. Create channel implementation
   Create `src/channels/telegram.ts` based on `whatsapp.ts`

3. Register channel
   Modify `src/index.ts` to add Telegram channel

4. Update router
   Modify `src/router.ts` to handle Telegram message format

5. Test
   Run `npm run dev` and verify Telegram connection

## Verification

- Telegram bot responds to messages
- Messages appear in database
- Agent responses sent to Telegram
```

### 10.3 技能测试

```bash
# 测试技能
claude
/add-telegram

# 验证结果
npm test
npm run typecheck
```

---

## 11. 发布流程

### 11.1 版本号规范

遵循 [Semantic Versioning](https://semver.org/):

- **MAJOR**: 破坏性变更
- **MINOR**: 向后兼容的功能新增
- **PATCH**: 向后兼容的 Bug 修复

### 11.2 发布步骤

```bash
# 1. 更新版本号
npm version minor

# 2. 生成 CHANGELOG
npm run changelog

# 3. 构建
npm run build

# 4. 测试
npm test

# 5. 提交
git add .
git commit -m "chore: release v1.2.0"
git tag v1.2.0
git push origin main --tags

# 6. 发布到 npm (如果需要)
npm publish
```

---

## 12. 故障排查

### 12.1 常见问题

| 问题         | 原因          | 解决方案               |
| ------------ | ------------- | ---------------------- |
| 容器启动失败 | Docker 未运行 | 启动 Docker 服务       |
| 数据库锁定   | SQLite 竞争   | 检查 WAL 模式          |
| 认证失败     | Session 过期  | 重新运行 WhatsApp 认证 |
| 内存泄漏     | 未清理资源    | 检查 event listeners   |

### 12.2 调试技巧

```typescript
// 启用详细日志
DEBUG=* npm run dev

// 性能分析
node --prof dist/index.js
nodeprof-visualize isolate-*.log

// 内存快照
node --inspect dist/index.js
# 在 Chrome DevTools 中连接
```

---

## 13. 资源链接

- [主 README](../README.md)
- [产品需求文档](PRD.md)
- [数据模型文档](DATA_MODEL.md)
- [安全文档](SECURITY.md)
- [Claude Code 技能文档](https://code.claude.com/docs/en/skills)

---

**文档维护**: 开发团队
**最后更新**: 2025-02-24
