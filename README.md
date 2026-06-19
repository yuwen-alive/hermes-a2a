# hermes-a2a

**让多个 AI Agent 安全、高效地协作对话 —— 不会死循环、不会浪费 Token、不会打扰用户。**

> A protocol + reference implementation for safe Agent-to-Agent communication.

[English](#english) | [中文](#中文)

---

## 中文

### 这是什么?

`hermes-a2a` 是一套 **多 Agent 协作协议与实现**，解决的核心问题是:

> 当多个 AI Agent 在同一个群聊中对话时，如何防止死循环、避免资源浪费、确保人类随时可以打断?

### 为什么需要这个?

| 场景 | 没有 hermes-a2a | 有 hermes-a2a |
|------|----------------|---------------|
| 两个 Bot 同时收到任务 | 同时执行，同时撞限流 | Leader 分配，Executor 执行 |
| Bot A 回复 Bot B，B 又回复 A | 无限循环，Token 耗尽 | 信号标签 `[DONE]` 终止 |
| 用户想打断 Bot 对话 | 无法打断 | 用户消息立即暂停 Bot 协作 |
| Bot 执行失败 | 对方不知道，继续等 | 超时通知 + 降级策略 |

### 核心特性

- **四级信号标签**: `[CONTINUE]` / `[DONE]` / `[NEEDS_HUMAN]` / `[INFO]`
- **四层防死循环**: 内容相似度检测 -> 连续发言限制 -> Turn 软限制 -> 全局超时兜底
- **角色分配**: Leader (主协调人) + Executor (执行者)，任务分工清晰
- **人类打断**: 用户一说话，Bot 对话立即暂停
- **实战验证**: 小马 + 小牛在飞书群中的真实生产环境

### 快速开始

#### 1. 创建两个飞书应用

参考 [飞书接入指南](feishu/setup-guide.md)。

#### 2. 配置环境变量

```bash
# Bot A
cp feishu/config-templates/.env.example ~/.hermes-bot-a/.env
# 编辑 .env，填入 App A 的凭证

# Bot B
cp feishu/config-templates/.env.example ~/.hermes-bot-b/.env
# 编辑 .env，填入 App B 的凭证
```

#### 3. 启动

```bash
# 终端 1
HERMES_HOME=~/.hermes-bot-a hermes gateway start

# 终端 2
HERMES_HOME=~/.hermes-bot-b hermes gateway start
```

#### 4. 在飞书群里测试

```
@小马 帮我查一下服务器状态
@小牛 今天 API 用量是多少?
```

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
    ├── basic-2bot/                    # 双 Bot 基础教程
    ├── token-monitor/                 # Token 监控案例
    └── inbox-system/                  # 信箱系统案例
```

### 文档导航

| 我想... | 去看 |
|---------|------|
| 了解协议规范 | [protocol/spec.md](protocol/spec.md) |
| 学习防死循环机制 | [protocol/anti-loop.md](protocol/anti-loop.md) |
| 从零搭建飞书 Bot-to-Bot | [feishu/setup-guide.md](feishu/setup-guide.md) |
| 看看我们踩过的坑 | [docs/troubleshooting.md](docs/troubleshooting.md) |
| 了解多 Agent 架构 | [docs/architecture.md](docs/architecture.md) |
| 安装 Skill 包 | [skills/hermes-a2a/SKILL.md](skills/hermes-a2a/SKILL.md) |

---

## English

### What is this?

`hermes-a2a` is a **protocol and reference implementation for safe multi-Agent collaboration**, solving:

> When multiple AI Agents chat in the same group, how to prevent infinite loops, avoid resource waste, and let humans interrupt at any time?

### Core Features

- **Four-level signal tags**: `[CONTINUE]` / `[DONE]` / `[NEEDS_HUMAN]` / `[INFO]`
- **Four-layer anti-loop**: content similarity -> consecutive limit -> turn soft cap -> global timeout
- **Role assignment**: Leader + Executor with clear task division
- **Human interrupt**: user messages immediately pause Agent collaboration
- **Battle-tested**: production environment on Feishu with two real Agents

### Quick Start

See [feishu/setup-guide.md](feishu/setup-guide.md) for detailed instructions.

```bash
# 1. Copy config templates
cp feishu/config-templates/.env.example ~/.hermes-bot-a/.env
cp feishu/config-templates/.env.example ~/.hermes-bot-b/.env

# 2. Edit .env files with your Feishu app credentials

# 3. Start both instances
HERMES_HOME=~/.hermes-bot-a hermes gateway start &
HERMES_HOME=~/.hermes-bot-b hermes gateway start &
```

### License

[MIT](LICENSE)

---

*Built with ❤️ by 小马 🐴 + 小牛 🐮, coordinated by 雨文*
