# hermes-a2a

**让多个 Hermes Agent 在飞书群中协作对话 —— 分工明确、安全可靠、人类随时可打断。**

> A complete solution for multi-Agent collaboration on Feishu/Lark — with built-in anti-loop safety.

[English](#english) | [中文](#中文)

---

## 中文

### 这是什么?

`hermes-a2a` 是一套 **让多个 Hermes Agent 在飞书群中协作对话的完整方案**，解决的核心问题是:

> 当你有多个 Hermes 实例（比如一个在服务器、一个在 Windows），如何让它们在同一个飞书群中互相配合完成任务——而不是各干各的、互相死循环、或者抢着回答?

### 为什么需要这个?

| 场景 | 没有 hermes-a2a | 有 hermes-a2a |
|------|----------------|---------------|
| 两个 Bot 同时收到任务 | 同时执行，同时撞限流 | Leader 分配，Executor 执行 |
| Bot A 回复 Bot B，B 又回复 A | 无限循环，Token 耗尽 | 信号标签 `[DONE]` 自动终止 |
| 用户想打断 Bot 对话 | 无法打断 | 用户消息立即暂停 Bot 协作 |
| Bot 执行失败 | 对方不知道，继续等 | 超时通知 + 降级策略 |
| 想让 Bot 分工合作 | 没有机制 | Leader/Executor 角色分配 |

### 核心特性

- 🤝 **飞书原生协作**: 通过 @mention 实现 Bot 间直接对话，无需额外中间件
- 🎯 **角色分配**: Leader (主协调人) + Executor (执行者)，任务分工清晰
- 🏷️ **四级信号标签**: `[CONTINUE]` / `[DONE]` / `[NEEDS_HUMAN]` / `[INFO]`，每条消息明确意图
- 🛡️ **四层防死循环**: 内容相似度检测 → 连续发言限制 → Turn 软限制 → 全局超时兜底
- ✋ **人类打断**: 用户一说话，Bot 对话立即暂停
- 📦 **开箱即用**: 配置模板 + Hermes Skill 包 + 实战案例

### 快速开始

详细步骤见 [飞书接入指南](feishu/setup-guide.md)。

```bash
# 1. 复制配置模板
cp feishu/config-templates/.env.example ~/.hermes-bot-a/.env
cp feishu/config-templates/.env.example ~/.hermes-bot-b/.env
# 编辑 .env，填入各自的飞书应用凭证

# 2. 安装 Skill（让 Bot 遵守协作协议）
cp -r skills/hermes-a2a/ ~/.hermes-bot-a/skills/hermes-a2a/
cp -r skills/hermes-a2a/ ~/.hermes-bot-b/skills/hermes-a2a/

# 3. 启动
HERMES_HOME=~/.hermes-bot-a hermes gateway start
HERMES_HOME=~/.hermes-bot-b hermes gateway start
```

#### 在飞书群里测试

```
@Bot-A 帮我查一下服务器状态
@Bot-B 今天 API 用量是多少?
```

### 实战案例

| 案例 | 描述 | 涉及信号 |
|------|------|---------|
| [基础双 Bot 教程](examples/basic-collaboration/) | 从零搭建两个 Bot 协作 | `[CONTINUE]` `[DONE]` |
| [Token 监控](examples/token-monitor/) | 跨机器 Token 用量 Dashboard | `[INFO]` `[DONE]` |
| [信箱系统](examples/inbox-system/) | Bot 间离线消息信箱 | `[CONTINUE]` `[INFO]` `[DONE]` |

### 项目结构

```
hermes-a2a/
├── README.md                          # 本文件
├── LICENSE                            # MIT 协议
├── ROADMAP.md                         # 版本演进计划
│
├── protocol/                          # 协议规范
│   ├── spec.md                        # A2A 通信协议完整规范
│   ├── rules.md                       # 11 条协作规则 + 系统提示词模板
│   └── anti-loop.md                   # 四层防死循环机制详解
│
├── docs/                              # 技术文档
│   ├── architecture.md                # 多 Agent 扩展架构设计
│   ├── collaboration-guide.md         # 协作规范与行为准则
│   └── troubleshooting.md             # 踩坑记录 & FAQ
│
├── feishu/                            # 飞书接入
│   ├── setup-guide.md                 # 完整接入指南
│   ├── bot-identity-guide.md          # Bot 身份识别机制
│   ├── thread-and-reply-guide.md      # 话题与回复控制
│   ├── patches/                       # 代码补丁说明
│   │   └── README.md                  # 补丁状态与维护策略
│   └── config-templates/              # 配置模板
│       ├── .env.example               # 环境变量模板
│       ├── config-bot-a.yaml          # Bot A 配置
│       └── config-bot-b.yaml          # Bot B 配置
│
├── skills/                            # Hermes Skill 包
│   └── hermes-a2a/
│       ├── SKILL.md                   # Skill 主文件
│       ├── references/                # 快速参考
│       └── templates/                 # 消息模板
│
└── examples/                          # 实战案例
    ├── basic-collaboration/                    # 双 Bot 基础教程
    ├── token-monitor/                 # Token 监控案例
    └── inbox-system/                  # 信箱系统案例
```

### 文档导航

| 我想... | 去看 |
|---------|------|
| 从零搭建飞书 Bot-to-Bot | [feishu/setup-guide.md](feishu/setup-guide.md) |
| 了解协议规范 | [protocol/spec.md](protocol/spec.md) |
| 学习防死循环机制 | [protocol/anti-loop.md](protocol/anti-loop.md) |
| 看看踩过的坑 | [docs/troubleshooting.md](docs/troubleshooting.md) |
| 了解多 Agent 架构 | [docs/architecture.md](docs/architecture.md) |
| 安装 Skill 包 | [skills/hermes-a2a/SKILL.md](skills/hermes-a2a/SKILL.md) |

---

## English

### What is this?

`hermes-a2a` is a **complete solution for multi-Agent collaboration on Feishu/Lark**, solving:

> When you have multiple Hermes instances (e.g., one on a server, one on Windows), how do they work together in the same Feishu group chat — instead of working independently, looping infinitely, or competing to answer?

### Core Features

- 🤝 **Feishu-native collaboration**: Bot-to-bot communication via @mention, no extra middleware
- 🎯 **Role assignment**: Leader + Executor with clear task division
- 🏷️ **Four-level signal tags**: `[CONTINUE]` / `[DONE]` / `[NEEDS_HUMAN]` / `[INFO]`
- 🛡️ **Four-layer anti-loop**: content similarity → consecutive limit → turn soft cap → global timeout
- ✋ **Human interrupt**: user messages immediately pause Agent collaboration
- 📦 **Batteries included**: config templates + Hermes Skill + real-world examples

### Quick Start

See [feishu/setup-guide.md](feishu/setup-guide.md) for detailed instructions.

```bash
# 1. Copy config templates
cp feishu/config-templates/.env.example ~/.hermes-bot-a/.env
cp feishu/config-templates/.env.example ~/.hermes-bot-b/.env

# 2. Install Skill (makes Bots follow the protocol)
cp -r skills/hermes-a2a/ ~/.hermes-bot-a/skills/hermes-a2a/
cp -r skills/hermes-a2a/ ~/.hermes-bot-b/skills/hermes-a2a/

# 3. Start both instances
HERMES_HOME=~/.hermes-bot-a hermes gateway start
HERMES_HOME=~/.hermes-bot-b hermes gateway start
```

### Examples

| Example | Description | Signals Used |
|---------|-------------|-------------|
| [Basic Collaboration](examples/basic-collaboration/) | Build two Bots from scratch | `[CONTINUE]` `[DONE]` |
| [Token Monitor](examples/token-monitor/) | Cross-machine Token usage Dashboard | `[INFO]` `[DONE]` |
| [Inbox System](examples/inbox-system/) | Offline messaging between Bots | `[CONTINUE]` `[INFO]` `[DONE]` |

### Project Structure

```
hermes-a2a/
├── README.md
├── LICENSE
├── ROADMAP.md
├── protocol/           # Protocol specs (spec, rules, anti-loop)
├── docs/               # Architecture, troubleshooting, collaboration guide
├── feishu/             # Feishu integration (setup, bot identity, config templates)
├── skills/             # Hermes Skill package
└── examples/           # Real-world examples
```

### Documentation

| I want to... | Go to |
|--------------|-------|
| Set up Feishu Bot-to-Bot from scratch | [feishu/setup-guide.md](feishu/setup-guide.md) |
| Read the protocol spec | [protocol/spec.md](protocol/spec.md) |
| Understand anti-loop mechanisms | [protocol/anti-loop.md](protocol/anti-loop.md) |
| See common pitfalls | [docs/troubleshooting.md](docs/troubleshooting.md) |
| Learn about multi-Agent architecture | [docs/architecture.md](docs/architecture.md) |
| Install the Skill package | [skills/hermes-a2a/SKILL.md](skills/hermes-a2a/SKILL.md) |

### License

[MIT](LICENSE)

---

*Built with ❤️ by the Hermes community*
