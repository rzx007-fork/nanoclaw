# NanoClaw 用户指南

| 文档版本 | 日期       | 作者     | 变更说明 |
| -------- | ---------- | -------- | -------- |
| v1.0     | 2025-02-24 | 产品团队 | 初始版本 |

---

## 1. 快速开始

### 1.1 系统要求

| 组件        | 最低要求                           |
| ----------- | ---------------------------------- |
| 操作系统    | macOS 12+ / Ubuntu 20.04+          |
| Node.js     | 20.0 或更高版本                    |
| Claude Code | 最新版本                           |
| 容器运行时  | Docker 或 Apple Container（macOS） |

### 1.2 安装步骤

#### 第 1 步：克隆仓库

```bash
git clone https://github.com/qwibitai/nanoclaw.git
cd nanoclaw
```

#### 第 2 步：运行设置

```bash
claude
/setup
```

Claude Code 将自动完成：

1. 检测环境和依赖
2. 安装 Node.js 依赖包
3. 构建 Docker 容器镜像
4. 扫描 WhatsApp 二维码进行认证
5. 创建主通道（自聊）
6. 配置系统服务（自动启动）

#### 第 3 步：验证安装

在 WhatsApp 自聊中发送：

```
@Andy 你好
```

如果收到回复，安装成功！

---

## 2. 基础使用

### 2.1 触发助手

在已注册的群组或自聊中，使用触发词激活助手：

```
@Andy [你的请求]
```

示例：

```
@Andy 帮我检查 src/api.ts 中的错误
@Andy 解释一下什么是微服务
@Andy 总结今天的会议记录
```

### 2.2 主通道功能

**主通道（Main Channel）** 是你的自聊（Self-Chat），拥有管理权限：

| 功能         | 命令示例                         |
| ------------ | -------------------------------- |
| 列出所有任务 | `@Andy 列出所有计划任务`         |
| 暂停任务     | `@Andy 暂停任务 task-123`        |
| 注册新群组   | `@Andy 注册 "Project Team" 群组` |
| 查看所有群组 | `@Andy 显示所有已注册群组`       |

### 2.3 群组隔离

每个 WhatsApp 群组都有：

- **独立上下文**: 对话历史不会泄露
- **独立文件系统**: 每个组有自己的 `groups/{name}/` 文件夹
- **独立记忆**: 每个组有各自的 `CLAUDE.md`

这意味着：

- 在"项目团队"群中的对话不会影响"家庭"群
- 每个组可以设置不同的偏好和规则

---

## 3. 定时任务

### 3.1 创建任务

在任何群组中发送：

```
@Andy 每周一早上 8 点给我发一份 Hacker News 的 AI 资讯简报
```

助手将自动：

1. 识别 Cron 表达式（每周一 8 点）
2. 创建定时任务
3. 在指定时间执行
4. 发送结果到当前群组

### 3.2 任务类型

#### Cron 表达式（周期性）

```
@Andy 每个工作日早上 9 点提醒我开会
@Andy 每周五下午 5 点生成本周报告
@Andy 每月 1 号提醒我交账单
```

#### Interval 间隔（定时）

```
@Andy 每 2 小时检查一次服务器状态
@Andy 每天晚上 10 点总结今天的对话
```

#### 单次执行（Once）

```
@Andy 在 2025 年 3 月 1 日早上 8 点发送生日提醒
```

### 3.3 任务管理

| 操作     | 命令                      | 说明                 |
| -------- | ------------------------- | -------------------- |
| 列出任务 | `@Andy 列出所有任务`      | 显示所有任务及其状态 |
| 暂停任务 | `@Andy 暂停任务 task-123` | 暂停指定任务         |
| 恢复任务 | `@Andy 恢复任务 task-123` | 恢复暂停的任务       |
| 取消任务 | `@Andy 取消任务 task-123` | 删除任务             |

### 3.4 任务上下文模式

创建任务时可指定上下文模式：

| 模式         | 说明                         | 示例                 |
| ------------ | ---------------------------- | -------------------- |
| **Group**    | 复用群组当前会话，保持上下文 | 适合连续性的每日简报 |
| **Isolated** | 每次执行创建新会话           | 适合独立的任务执行   |

示例：

```
@Anna 使用 group 模式，每天早上 9 点总结昨天的消息
```

---

## 4. 记忆系统

### 4.1 记忆类型

NanoClaw 有两层记忆：

#### 全局记忆（Global Memory）

- **位置**: `groups/CLAUDE.md`
- **访问**: 所有群组可读，仅主通道可写
- **用途**: 存储通用偏好、规则、全局信息

示例内容：

```markdown
# 全局配置

- 助手名称: Andy
- 响应风格: 简洁直接
- 默认语言: 中文

# 全局规则

- 不涉及政治敏感话题
- 技术问题提供详细解释
```

#### 组记忆（Group Memory）

- **位置**: `groups/{name}/CLAUDE.md`
- **访问**: 仅该群组可读写
- **用途**: 存储组特定偏好、项目信息、上下文

示例内容（`groups/project-team/CLAUDE.md`）：

```markdown
# Project Team 配置

## 项目信息

- 项目名称: NanoClaw
- 技术栈: Node.js, TypeScript, SQLite
- 仓库: https://github.com/qwibitai/nanoclaw

## 偏好设置

- 代码风格: Prettier
- 测试框架: Vitest
- 优先考虑性能而非可读性

## 团队成员

- John: 前端开发
- Jane: 后端开发
```

### 4.2 记忆管理

**在群组中直接对话**：

```
@Andy 记住：这个项目使用 TypeScript 开发
@Anna 从现在开始，所有代码回复都要包含注释
@Anna 忘记之前的规则，使用默认风格
```

**直接编辑记忆文件**（高级用户）：

```bash
# 编辑全局记忆
vim groups/CLAUDE.md

# 编辑组记忆
vim groups/project-team/CLAUDE.md
```

### 4.3 记忆最佳实践

1. **明确关键信息**

   ```markdown
   ## 关键信息

   - 服务器地址: 192.168.1.100
   - API 密钥: 见 store/auth/api-key.txt
   ```

2. **结构化组织**

   ```markdown
   # 项目配置

   ## 基础信息

   ## 技术栈

   ## 团队成员
   ```

3. **定期更新**
   ```markdown
   # 最后更新: 2025-02-24

   ## 最新变更

   - 添加了 Telegram 支持
   ```

---

## 5. 文件操作

### 5.1 文件系统访问

助手可以在组文件夹内读写文件：

```
@Andy 读取 config.json 文件
@Anna 在 logs/ 目录下创建新的错误日志
@Anna 更新 README.md，添加新的安装说明
```

### 5.2 安全性

- **默认隔离**: 助手只能访问 `groups/{name}/` 文件夹
- **挂载控制**: 通过 `~/.config/nanoclaw/mount-allowlist.json` 配置额外挂载
- **只读保护**: 可配置只读访问敏感目录

### 5.3 常见文件操作

```
# 读取文件
@Andy 读取 src/index.ts 的前 100 行

# 写入文件
@Anna 在项目根目录创建 CHANGELOG.md

# 编辑文件
@Andy 在 README.md 中添加新的章节

# 搜索文件
@Anna 查找所有包含 "TODO" 的文件
```

---

## 6. 网络访问

### 6.1 Web 搜索

```
@Andy 搜索 "TypeScript 5.0 新特性"
@Anna 查找最新的 Node.js 版本
```

### 6.2 网页抓取

```
@Andy 获取 https://github.com/qwibitai/nanoclaw 的 README
@Anna 抓取 Hacker News 首页的新闻标题
```

### 6.3 浏览器自动化

助手可以控制浏览器执行复杂任务：

```
@Andy 打开 https://example.com，登录并导出报表
@Anna 在 Google 上搜索，然后打开第一个结果
```

---

## 7. 高级功能

### 7.1 群组注册

**仅主通道可注册新群组**：

```
@Andy 注册 "产品讨论" 群组
```

注册后：

- 群组获得独立的 `groups/product-discussion/` 文件夹
- 可以在该群组中使用 `@Andy` 触发助手
- 助手将记忆该群组的上下文

### 7.2 自定义触发词

默认触发词是 `@Andy`，可以通过修改代码更改：

```typescript
// 编辑 src/config.ts
export const ASSISTANT_NAME = process.env.ASSISTANT_NAME || 'Anna';
```

然后重启服务：

```bash
# macOS
launchctl kickstart -k gui/$(id -u)/com.nanoclaw

# Linux
systemctl --user restart nanoclaw
```

### 7.3 容器超时配置

默认容器超时 30 分钟，可通过环境变量调整：

```bash
# 编辑 .env
CONTAINER_TIMEOUT=600000  # 10 分钟

# 重启服务
systemctl --user restart nanoclaw
```

### 7.4 并发控制

默认最多同时运行 5 个容器，可调整：

```bash
# 编辑 .env
MAX_CONCURRENT_CONTAINERS=3

# 重启服务
systemctl --user restart nanoclaw
```

---

## 8. 故障排查

### 8.1 常见问题

#### 问题：助手没有响应

**可能原因**：

1. 容器未启动
2. 消息未存储到数据库
3. 触发词未匹配

**解决方法**：

```bash
# 1. 检查服务状态
launchctl list | grep nanoclaw  # macOS
systemctl --user status nanoclaw  # Linux

# 2. 查看日志
tail -f logs/nanoclaw.log

# 3. 检查数据库
sqlite3 store/messages.db "SELECT COUNT(*) FROM messages WHERE chat_jid = '你的JID'"

# 4. 重启服务
launchctl kickstart -k gui/$(id -u)/com.nanoclaw
```

#### 问题：WhatsApp 认证失败

**可能原因**：

1. 二维码未及时扫描
2. Session 过期
3. 网络问题

**解决方法**：

```bash
# 重新认证
npm run auth

# 扫描新的二维码

# 重启服务
launchctl kickstart -k gui/$(id -u)/com.nanoclaw
```

#### 问题：定时任务未执行

**可能原因**：

1. Cron 表达式错误
2. 任务被暂停
3. 时区配置不正确

**解决方法**：

```bash
# 检查任务状态
sqlite3 store/messages.db "SELECT * FROM scheduled_tasks WHERE status = 'active'"

# 查看任务日志
sqlite3 store/messages.db "SELECT * FROM task_run_logs ORDER BY run_at DESC LIMIT 10"

# 验证 Cron 表达式
# 在线工具: https://crontab.guru/
```

#### 问题：容器启动失败

**可能原因**：

1. Docker 未运行
2. 镜像未构建
3. 权限问题

**解决方法**：

```bash
# 1. 检查 Docker
docker ps

# 2. 重新构建镜像
cd container
./build.sh

# 3. 测试运行
docker run --rm nanoclaw-agent:latest echo "Container OK"

# 4. 检查挂载权限
ls -la groups/
```

### 8.2 调试模式

启用详细日志：

```bash
# 方法 1: 环境变量
export LOG_LEVEL=debug
npm run dev

# 方法 2: 编辑 src/logger.ts
const level = 'debug';
```

### 8.3 获取帮助

如果问题仍未解决：

1. 查看 [常见问题解答](../README.md#faq)
2. 阅读 [故障排查清单](DEBUG_CHECKLIST.md)
3. 加入 [Discord 社区](https://discord.gg/VDdww8qS42)
4. 在 [GitHub Issues](https://github.com/qwibitai/nanoclaw/issues) 提问

---

## 9. 定制化

### 9.1 改变触发词

```bash
# 方法 1: 环境变量
export ASSISTANT_NAME=Bob

# 方法 2: 修改代码
vim src/config.ts
# 修改: export const ASSISTANT_NAME = 'Bob';

# 重启服务
systemctl --user restart nanoclaw
```

### 9.2 添加额外功能

NanoClaw 通过 **技能（Skills）** 扩展：

#### 可用技能

| 技能                          | 功能                   |
| ----------------------------- | ---------------------- |
| `/add-telegram`               | 添加 Telegram 支持     |
| `/add-gmail`                  | 集成 Gmail             |
| `/add-discord`                | 添加 Discord 支持      |
| `/add-voice-transcription`    | 语音消息转录           |
| `/x-integration`              | X (Twitter) 集成       |
| `/convert-to-apple-container` | 切换到 Apple Container |

#### 使用技能

```bash
claude
/add-telegram
```

Claude Code 将引导你完成安装。

### 9.3 修改代码

直接修改源代码实现定制：

```bash
# 1. 编辑代码
vim src/index.ts

# 2. 重新构建
npm run build

# 3. 重启服务
systemctl --user restart nanoclaw
```

---

## 10. 备份与恢复

### 10.1 备份

```bash
# 方法 1: 备份整个项目
tar -czf nanoclaw-backup-$(date +%Y%m%d).tar.gz nanoclaw/

# 方法 2: 仅备份数据
tar -czf nanoclaw-data-$(date +%Y%m%d).tar.gz store/ groups/

# 方法 3: 仅备份数据库
cp store/messages.db store/messages.db.backup.$(date +%Y%m%d)
```

### 10.2 恢复

```bash
# 1. 停止服务
launchctl unload ~/Library/LaunchAgents/com.nanoclaw.plist

# 2. 解压备份
tar -xzf nanoclaw-backup-20250224.tar.gz

# 3. 重启服务
launchctl load ~/Library/LaunchAgents/com.nanoclaw.plist
```

### 10.3 自动备份（推荐）

创建 cron 任务每日备份：

```bash
# 编辑 crontab
crontab -e

# 添加每日 2:00 备份
0 2 * * * /bin/tar -czf /path/to/backup/nanoclaw-$(date +\%Y\%m\%d).tar.gz /path/to/nanoclaw
```

---

## 11. 性能优化

### 11.1 减少延迟

- **增加并发数**: 提高 `MAX_CONCURRENT_CONTAINERS`
- **优化数据库**: 定期运行 `VACUUM; ANALYZE;`
- **使用 SSD**: 将数据库放在 SSD 上

### 11.2 减少内存占用

- **降低并发数**: 减少 `MAX_CONCURRENT_CONTAINERS`
- **缩短超时**: 减少 `CONTAINER_TIMEOUT`
- **定期清理**: 删除 90 天前的任务日志

### 11.3 提高响应速度

- **保持容器活跃**: 通过 `IDLE_TIMEOUT` 控制容器保持时间
- **使用更快的机器**: 升级 CPU 和内存
- **优化网络**: 使用更快的网络连接

---

## 12. 安全建议

### 12.1 密钥管理

- **不要在记忆文件中存储密钥**
- **使用 `store/auth/` 目录存储敏感文件**
- **确保 `store/` 目录不被提交到 Git**

### 12.2 网络安全

- **使用 VPN**: 在公共网络上使用
- **防火墙**: 限制端口访问
- **更新 Docker**: 定期更新 Docker 版本

### 12.3 访问控制

- **限制注册群组**: 只注册信任的群组
- **审查挂载权限**: 检查 `~/.config/nanoclaw/mount-allowlist.json`
- **定期审计**: 审查日志和数据库

---

## 13. 更新与升级

### 13.1 检查更新

```bash
cd nanoclaw
git pull origin main
```

### 13.2 运行迁移

```bash
claude
/update
```

`/update` 技能将：

1. 拉取上游更改
2. 合并你的自定义代码
3. 运行数据库迁移
4. 重新构建

### 13.3 版本兼容性

- **v1.0 → v1.1**: 自动迁移，无破坏性变更
- **v1.1 → v1.2**: 添加新列，向后兼容
- **v1.2 → v2.0**: 重大升级，需要手动迁移

---

## 14. 最佳实践

### 14.1 使用建议

| 场景     | 建议                               |
| -------- | ---------------------------------- |
| 代码审查 | 在主通道中，助手可访问所有项目文件 |
| 日常简报 | 使用定时任务，group 模式保持上下文 |
| 紧急修复 | 直接在群组中触发助手，快速响应     |
| 长期项目 | 在记忆文件中记录项目上下文         |

### 14.2 效率技巧

1. **使用明确的需求**

   ```
   ❌ @Andy 帮我弄一下
   ✅ @Andy 修复 src/api.ts:150 的空指针错误
   ```

2. **提供上下文**

   ```
   ❌ @Anna 生成报告
   ✅ @Anna 基于上周的 commit 历史，生成一份技术报告
   ```

3. **分步执行**
   ```
   ❌ @Andy 完成整个项目的重构
   ✅ @Andy 先分析当前代码结构，然后建议重构方案
   ```

### 14.3 避免常见错误

| 错误         | 避免                       |
| ------------ | -------------------------- |
| 过度依赖助手 | 助手是工具，不是替代品     |
| 不验证输出   | 总是审查助手生成的代码     |
| 忽略错误日志 | 及时查看日志，快速定位问题 |
| 不备份       | 定期备份，防止数据丢失     |

---

## 15. 资源链接

### 15.1 官方资源

- [主 README](../README.md) - 项目介绍
- [中文 README](../README_zh.md) - 中文文档
- [产品需求文档](PRD.md) - 功能规格
- [开发指南](DEVELOPMENT.md) - 开发者文档
- [数据模型](DATA_MODEL.md) - 数据库设计

### 15.2 社区资源

- [Discord 社区](https://discord.gg/VDdww8qS42) - 实时讨论
- [GitHub Issues](https://github.com/qwibitai/nanoclaw/issues) - 问题追踪
- [GitHub Discussions](https://github.com/qwibitai/nanoclaw/discussions) - 社区讨论

### 15.3 学习资源

- [Claude Code 文档](https://code.claude.com/docs) - AI 编程工具
- [Claude Agent SDK](https://docs.anthropic.com/) - AI SDK 文档
- [Docker 文档](https://docs.docker.com/) - 容器技术

---

## 16. FAQ

### 16.1 基础问题

**Q: NanoClaw 是免费的吗？**
A: NanoClaw 是开源软件，免费使用。但 Claude API 可能产生费用。

**Q: 我可以在多个手机上使用吗？**
A: 可以，WhatsApp 支持多设备，扫描同一二维码即可。

**Q: 助手能看到我的所有消息吗？**
A: 不。助手只能看到注册群组中的消息，且内容存储在你的本地。

### 16.2 技术问题

**Q: 需要一直在线吗？**
A: 需要设备保持在线以接收 WhatsApp 消息。建议使用服务器或 NAS 部署。

**Q: 数据会同步到云端吗？**
A: 不会。所有数据存储在本地，你需要手动备份。

**Q: 可以同时运行多个 NanoClaw 实例吗？**
A: 不建议。多个实例会导致 WhatsApp 连接冲突。

### 16.3 高级问题

**Q: 可以自己训练模型吗？**
A: NanoClaw 使用 Claude API，不支持自定义模型。但可以通过记忆文件定制行为。

**Q: 支持其他语言吗？**
A: 支持所有 Claude 支持的语言，包括中文、英文、日文等。

**Q: 可以集成其他 AI 服务吗？**
A: 可以。通过修改代码或开发技能集成 OpenAI、Gemini 等服务。

---

**文档维护**: 产品团队
**最后更新**: 2025-02-24
**反馈**: 在 [GitHub Issues](https://github.com/qwibitai/nanoclaw/issues) 提交反馈
