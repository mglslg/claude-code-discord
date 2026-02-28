# Claude Code 多智能体协作方案

## 一、需求背景

### 核心需求
- **多个 Claude Code 实例互相协作**：1个 Opus 作为 Manager，2个 Haiku 分别负责编码和测试
- **多轮对话讨论**：AI 之间需要讨论方案，而非简单的任务分发
- **人类可干预**：用户可以随时介入对话流程
- **所见即所得**：在 Discord 上可视化看到所有对话内容
- **保留 CC 代码能力**：不能牺牲 Claude Code 的 Read/Edit/Write/Bash 等工具能力

### 关键挑战
Claude Code 是**被动响应式**工具：
- ❌ 不能主动监听消息
- ❌ 不能作为持续运行的服务
- ✅ 只能在接收输入时才开始工作

如果每个 Agent 都需要人工触发，就失去了自动化协作的意义。

---

## 二、方案探索历程

### 方案A：纯文件系统通信 ❌
**思路**：共享目录，通过读写文件实现通信
**问题**：需要人工不断触发各实例去检查文件

### 方案B：HTTP 本地服务 ❌
**思路**：Manager 启动 HTTP 服务器，其他实例调用 API
**问题**：CC 仍然是被动的，需要外部程序决定何时触发

### 方案C：纯 API 方案 ❌
**思路**：直接调用 Claude API，自己实现多智能体逻辑
**问题**：失去了 Claude Code 的代码操作能力（需要重新实现 Read/Edit/Write 等工具）

### 方案D：`--print` + `stream-json` ❌
**思路**：使用 CC 的 headless 模式通过 stdin/stdout 控制
**问题**：`--print` 是一次性执行，完成后进程退出，仍需调度器决定何时触发

---

## 三、最终方案：tmux + Discord Bot ✅

### 核心突破：Remote Control + tmux

**关键发现**：
1. **Remote Control**：`claude remote-control` 可以让 CC 持续运行，轮询等待远程消息
2. **第三方项目**：已有项目证明可通过 Discord/Telegram 控制 CC（[Claude-Code-Remote](https://github.com/JessyTsui/Claude-Code-Remote)）
3. **tmux 模拟输入**：通过 `tmux send-keys` 可以向后台 CC 实例发送命令

### 架构图

```
┌────────────────────────────────┐
│       Discord 频道              │
│  #opus  #coder  #tester  #all  │  ← 人类在此参与讨论
└───────────┬────────────────────┘
            │
    ┌───────▼────────┐
    │  Orchestrator  │  ← Discord Bot (Node.js/Python)
    │  - 监听Discord │
    │  - 解析@mention│
    │  - 路由消息    │
    │  - tmux 控制   │
    └─┬─────┬───────┬┘
      │     │       │
      ▼     ▼       ▼
   [tmux] [tmux] [tmux]  ← 3个持续运行的 tmux session
   Opus   Coder  Tester  ← Claude Code 实例
     ↓      ↓       ↓
   完整的代码能力: Read/Edit/Write/Bash/Grep/...
```

### 工作流程

#### 1. 启动阶段

```bash
# 前置：设置长期认证 token（一次性操作）
claude setup-token

# 启动3个 CC 实例在各自 tmux session
tmux new-session -d -s opus "claude --model=opus"
tmux new-session -d -s coder "claude --model=haiku"
tmux new-session -d -s tester "claude --model=haiku"
```

#### 2. Discord Bot 消息路由

```javascript
// 监听 Discord 消息
discord.on('message', async (msg) => {
  // 解析 @mention
  if (msg.content.includes('@Opus')) {
    // 发送到 Opus 的 tmux session
    exec(`tmux send-keys -t opus "${msg.content}" Enter`);

    // 等待响应后捕获输出
    await sleep(5000);
    let output = exec(`tmux capture-pane -t opus -p`);

    // 发回 Discord
    discord.send(`👑 Opus: ${output}`);

    // 智能路由：解析输出，看是否要@其他智能体
    if (output.includes('@Coder')) {
      exec(`tmux send-keys -t coder "${output}" Enter`);
    }
  }
});
```

#### 3. 多轮对话示例

```
[Discord #claude-team 频道]

皇上: @Opus 请实现用户登录功能

👑 Opus: 收到！我需要：
  1. @Coder 请实现 login.js，要求支持邮箱登录和密码加密
  2. 完成后 @Tester 进行安全测试

⚙️ Coder: @Opus 已完成 login.js，使用 bcrypt 加密密码，返回 JWT token

🧪 Tester: @Opus 测试完成，发现2个问题：
  - 密码长度未验证
  - 缺少 rate limiting

👑 Opus: @Coder 请修复这两个问题

⚙️ Coder: @Opus 已修复，添加了密码验证和限流中间件

皇上: @Tester 再测一次

🧪 Tester: @皇上 所有测试通过！✅
```

---

## 四、关键技术点

### 4.1 setup-token 的作用

**问题**：tmux 后台 session 无法打开浏览器进行 OAuth 认证

**解决**：
```bash
# 步骤1：提前生成长期有效 token（需要交互式环境）
claude setup-token
# → 打开浏览器，完成 OAuth 认证
# → Token 保存到 ~/.claude/.credentials.json（有效期1年）

# 步骤2：之后启动的所有 CC 实例自动使用此 token
tmux new-session -d -s opus "claude --model=opus"
# ✅ 无需人工登录，后台自动认证
```

**环境变量方式**：
```bash
export CLAUDE_CODE_OAUTH_TOKEN="sk-ant-oat01-xxxxx"
claude --model=opus  # 自动使用此 token
```

**优先级**：
- 环境变量 `CLAUDE_CODE_OAUTH_TOKEN` > 配置文件 `~/.claude/.credentials.json`

### 4.2 tmux 命令速查

```bash
# 创建后台 session
tmux new-session -d -s <session_name> "<command>"

# 向 session 发送命令（模拟键盘输入）
tmux send-keys -t <session_name> "your message" Enter

# 捕获 session 的输出
tmux capture-pane -t <session_name> -p

# 列出所有 session
tmux list-sessions

# 附加到 session（人工查看）
tmux attach -t <session_name>

# 杀死 session
tmux kill-session -t <session_name>
```

### 4.3 智能体通信协议

**约定使用 @mention 机制**：
- `@Opus` - 发送给 Manager
- `@Coder` - 发送给编码 Agent
- `@Tester` - 发送给测试 Agent
- `@皇上` / `@all` - 发送给所有人

**消息格式建议**：
```json
{
  "from": "opus_manager",
  "to": "haiku_coder",
  "type": "task_assignment",
  "content": "实现 login.js",
  "context": {
    "files": ["src/auth/"],
    "requirements": ["邮箱登录", "密码加密", "JWT"]
  }
}
```

---

## 五、setup-token 的设计场景

Anthropic 设计此功能主要为了解决**自动化场景下的认证问题**：

### 5.1 官方使用场景

**场景一：远程开发环境**
- 远程服务器（SSH，无 GUI）
- Docker 容器
- 云 IDE（GitHub Codespaces, Gitpod）

**场景二：CI/CD 自动化**（60%+ 团队在用）
- GitHub Actions 自动代码审查
- GitLab CI 自动化测试生成
- 自动安全扫描
- 文档自动生成
- Changelog 自动化

### 5.2 典型用例

**GitHub Actions 示例**：
```yaml
name: AI Code Review
on: [pull_request]

jobs:
  review:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3

      - name: AI Review
        env:
          CLAUDE_CODE_OAUTH_TOKEN: ${{ secrets.CLAUDE_TOKEN }}
        run: |
          git diff main...HEAD | \
          claude -p --output-format=json "审查代码,找出问题"
```

**Docker 容器示例**：
```dockerfile
FROM ubuntu:22.04
ENV CLAUDE_CODE_OAUTH_TOKEN=${OAUTH_TOKEN}
CMD ["claude", "-p", "生成 API 文档"]
```

### 5.3 统计数据

- **60%+** 的团队在 CI/CD 中使用 Claude Code
- **代码审查时间平均减少 45%**
- **90% 的 CI/CD 场景**只需 `-p` flag

---

## 六、方案优势

### ✅ 保留 CC 完整能力
每个实例仍可使用 Read/Edit/Write/Bash/Grep 等所有工具

### ✅ 真正的多轮对话
AI 之间可以讨论方案，通过 @mention 路由消息

### ✅ 人类可随时干预
在 Discord 直接发消息即可介入讨论

### ✅ 所见即所得
Discord 实时展示所有对话，清晰可见

### ✅ 持续运行
tmux session 后台运行，不需要反复启动

### ✅ 已验证可行
第三方项目已证明技术可行性

---

## 七、A2A 协议集成方案

### 7.1 A2A 协议简介

**Agent2Agent（A2A）协议**是 Google 于 2025年4月发布的开放标准，旨在实现不同 AI 智能体之间的标准化通信。

**核心能力**：
- **能力发现**：通过 Agent Card（JSON）广播智能体能力
- **任务管理**：以任务为导向的通信，支持任务生命周期追踪
- **协作通信**：智能体间传递上下文、回复、文件等

**治理**：由 Linux Foundation 管理，150+ 组织支持（Google、IBM、Salesforce、MongoDB 等）

### 7.2 为什么需要 A2A

虽然可以使用纯文本 @mention 机制，但 A2A 协议提供了额外优势：

| 维度 | 纯文本 @mention | A2A 协议 |
|------|----------------|---------|
| **结构化** | ❌ 依赖自然语言解析 | ✅ 标准 JSON 格式 |
| **任务追踪** | ❌ 需要自己实现 | ✅ 内置 task_id 和状态管理 |
| **上下文传递** | ⚠️ 容易丢失 | ✅ context 字段明确 |
| **可扩展性** | ❌ 格式不固定 | ✅ 行业标准，易于扩展 |
| **互操作性** | ❌ 仅限本系统 | ✅ 可与其他 A2A 系统对接 |

### 7.3 A2A 消息格式

在本方案中采用的 A2A 消息格式：

```json
{
  "task_id": "login-feature-001",
  "from_agent": {
    "name": "opus_manager",
    "capabilities": ["planning", "coordination"]
  },
  "to_agent": {
    "name": "haiku_coder",
    "capabilities": ["coding", "refactoring"]
  },
  "message_type": "task_assignment",
  "content": "请实现 login.js，要求支持邮箱登录和密码加密",
  "context": {
    "files": ["src/auth/"],
    "requirements": ["JWT", "bcrypt"]
  },
  "timestamp": "2026-02-28T10:30:00Z",
  "status": "pending"
}
```

### 7.4 混合解析模式（核心设计）

**关键问题**：JSON 由程序解析还是 AI 解析？

**答案**：**混合模式** - Orchestrator 解析 JSON（路由用），同时格式化后发给 CC（AI 理解用）

#### 工作流程

```
1. Discord 收到消息（可以是用户输入或 AI 输出）
   ↓
2. Orchestrator 解析 JSON
   - 提取 to_agent.name → 确定路由目标
   - 提取 task_id → 记录到任务追踪系统
   - 提取 from_agent → 用于回复路由
   ↓
3. Orchestrator 格式化消息（JSON → 人类友好文本）
   ↓
4. 通过 tmux send-keys 发送给目标 CC
   ↓
5. CC（Claude）理解任务，执行代码操作
   ↓
6. Orchestrator 捕获 CC 输出（tmux capture-pane）
   ↓
7. 解析输出，识别 @mention 或任务完成标记
   ↓
8. 构造新的 A2A 消息，发送给下一个智能体（回到步骤2）
```

### 7.5 消息格式转换示例

#### Orchestrator 收到的 A2A JSON：
```json
{
  "task_id": "login-feature-001",
  "from_agent": { "name": "opus_manager" },
  "to_agent": { "name": "haiku_coder" },
  "content": "请实现 login.js，要求支持邮箱登录和密码加密",
  "context": {
    "files": ["src/auth/"],
    "requirements": ["JWT", "bcrypt"]
  }
}
```

#### 发送给 CC 的格式化文本：
```
🔔 新任务: login-feature-001
📩 来自: opus_manager

请实现 login.js，要求支持邮箱登录和密码加密

📂 相关文件: src/auth/
⚙️  技术栈: JWT, bcrypt

完成后请 @opus_manager 汇报
```

#### CC 的输出示例：
```
已完成 login.js 实现：
- ✅ 使用 bcrypt 加密密码（salt rounds: 10）
- ✅ JWT token 生成（过期时间: 24h）
- ✅ 邮箱格式验证
- ✅ 错误处理

@opus_manager 任务完成，请审查
```

#### Orchestrator 解析输出后构造的新 A2A 消息：
```json
{
  "task_id": "login-feature-001",
  "from_agent": { "name": "haiku_coder" },
  "to_agent": { "name": "opus_manager" },
  "message_type": "task_completed",
  "content": "已完成 login.js 实现...",
  "status": "completed",
  "artifacts": ["src/auth/login.js"]
}
```

### 7.6 Orchestrator 核心职责

使用 A2A 协议后，Orchestrator 需要承担以下职责：

1. **消息路由**：根据 `to_agent.name` 路由到对应 tmux session
2. **格式转换**：A2A JSON ⇄ 人类友好文本
3. **任务追踪**：记录所有 `task_id` 的状态（pending/in_progress/completed）
4. **输出解析**：从 CC 的自然语言输出中提取 @mention 和状态
5. **Discord 展示**：将对话内容实时同步到 Discord 频道
6. **历史记录**：保存完整的 A2A 消息链，用于审计和回溯

### 7.7 与纯 @mention 方案的对比

| 特性 | 纯 @mention | A2A 协议 |
|------|------------|---------|
| **实现难度** | 简单 | 中等（需要 JSON 解析） |
| **任务追踪** | 手动实现 | 内置 task_id |
| **消息结构** | 自由文本 | 标准化 JSON |
| **可调试性** | 依赖日志 | 结构化数据，易于追踪 |
| **扩展性** | 有限 | 可与其他 A2A 系统对接 |
| **首选场景** | 快速原型 | 生产环境、长期维护 |

**建议**：
- **MVP 阶段**：可以先用纯 @mention，快速验证可行性
- **生产阶段**：引入 A2A 协议，提升可维护性和扩展性

---

## 八、待实现部分

### 8.1 Orchestrator (Discord Bot)

**功能需求**：
- [ ] 连接 Discord，监听指定频道
- [ ] 解析 @mention，识别目标 Agent
- [ ] 通过 tmux send-keys 发送消息
- [ ] 捕获 tmux 输出并回复到 Discord
- [ ] 智能路由：解析 AI 输出中的 @mention
- [ ] 消息格式化（Emoji、颜色、代码块）
- [ ] 对话历史保存

**技术栈选择**：
- Node.js + discord.js
- Python + discord.py

### 8.2 启动脚本

```bash
#!/bin/bash
# start-claude-team.sh

# 检查 token
if ! grep -q "credentials" ~/.claude/.credentials.json 2>/dev/null; then
  echo "请先运行: claude setup-token"
  exit 1
fi

# 清理旧 session
tmux kill-session -t opus 2>/dev/null
tmux kill-session -t coder 2>/dev/null
tmux kill-session -t tester 2>/dev/null

# 启动3个 CC 实例
echo "启动 Opus Manager..."
tmux new-session -d -s opus "claude --model=opus"

echo "启动 Haiku Coder..."
tmux new-session -d -s coder "claude --model=haiku"

echo "启动 Haiku Tester..."
tmux new-session -d -s tester "claude --model=haiku"

# 启动 Discord Bot
echo "启动 Orchestrator..."
node orchestrator.js

echo "✅ 系统启动完成!"
echo "Discord 频道: #claude-team"
```

### 8.3 停止脚本

```bash
#!/bin/bash
# stop-claude-team.sh

tmux kill-session -t opus
tmux kill-session -t coder
tmux kill-session -t tester

pkill -f orchestrator.js

echo "✅ 系统已停止"
```

---

## 九、参考资料

### 官方文档
- [Run Claude Code programmatically](https://code.claude.com/docs/en/headless)
- [Authentication - Claude Code Docs](https://code.claude.com/docs/en/authentication)
- [Continue local sessions with Remote Control](https://code.claude.com/docs/en/remote-control)
- [GitHub Actions Integration](https://code.claude.com/docs/en/github-actions)

### A2A 协议
- [Agent2Agent Protocol](https://a2aprotocol.ai/)
- [A2A Protocol Specification](https://a2a-protocol.org/latest/)
- [Google A2A Announcement](https://developers.googleblog.com/en/a2a-a-new-era-of-agent-interoperability/)

### 第三方项目
- [Claude-Code-Remote](https://github.com/JessyTsui/Claude-Code-Remote) - 通过 Email/Discord/Telegram 控制 CC
- [claude-code-telegram](https://github.com/RichardAtCT/claude-code-telegram) - Telegram Bot
- [afk-code](https://github.com/clharman/afk-code) - Slack/Discord/Telegram 集成
- [claude-flow](https://github.com/ruvnet/claude-flow) - 企业级多智能体编排

### 技术指南
- [CI/CD and Headless Mode](https://angelo-lima.fr/en/claude-code-cicd-headless-en/)
- [Headless Mode Tutorial - SFEIR](https://institute.sfeir.com/en/claude-code/claude-code-headless-mode-and-ci-cd/tutorial/)
- [Stream-JSON Chaining](https://github.com/ruvnet/claude-flow/wiki/Stream-Chaining)

---

## 十、下一步行动

### MVP 阶段（快速验证）
1. [ ] 创建 Discord Bot，配置 Webhook
2. [ ] 编写 Orchestrator 核心逻辑（纯 @mention 路由）
3. [ ] 测试 tmux 命令集成
4. [ ] 编写启动/停止脚本
5. [ ] 验证多轮对话可行性

### 生产阶段（A2A 协议集成）
6. [ ] 实现 A2A 消息格式解析
7. [ ] 实现格式化转换（JSON ⇄ 人类友好文本）
8. [ ] 实现任务追踪系统（task_id 管理）
9. [ ] 实现输出解析器（提取 @mention 和状态）
10. [ ] 对话历史保存（A2A 格式）
11. [ ] 性能优化（响应延迟、消息队列）

---

**最后更新**: 2026-02-28
**状态**: 方案设计完成（已集成 A2A 协议），待实现
