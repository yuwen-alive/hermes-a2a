1|# hermes-a2a
2|
3|**让多个 Hermes Agent 在飞书群中协作对话 —— 分工明确、安全可靠、人类随时可打断。**
4|
5|> A complete solution for multi-Agent collaboration on Feishu/Lark — with built-in anti-loop safety.
6|
7|[English](#english) | [中文](#中文)
8|
9|---
10|
11|## 中文
12|
13|### 这是什么?
14|
15|`hermes-a2a` 是一套 **让多个 Hermes Agent 在飞书群中协作对话的完整方案**，解决的核心问题是:
16|
17|> 当你有多个 Hermes 实例（比如一个在服务器、一个在 Windows），如何让它们在同一个飞书群中互相配合完成任务——而不是各干各的、互相死循环、或者抢着回答?
18|
19|### 为什么需要这个?
20|
21|| 场景 | 没有 hermes-a2a | 有 hermes-a2a |
22||------|----------------|---------------|
23|| 两个 Bot 同时收到任务 | 同时执行，同时撞限流 | Leader 分配，Executor 执行 |
24|| Bot A 回复 Bot B，B 又回复 A | 无限循环，Token 耗尽 | 信号标签 `[DONE]` 自动终止 |
25|| 用户想打断 Bot 对话 | 无法打断 | 用户消息立即暂停 Bot 协作 |
26|| Bot 执行失败 | 对方不知道，继续等 | 超时通知 + 降级策略 |
27|| 想让 Bot 分工合作 | 没有机制 | Leader/Executor 角色分配 |
28|
29|### 核心特性
30|
31|- 🤝 **飞书原生协作**: 通过 @mention 实现 Bot 间直接对话，无需额外中间件
32|- 🎯 **角色分配**: Leader (主协调人) + Executor (执行者)，任务分工清晰
33|- 🏷️ **四级信号标签**: `[CONTINUE]` / `[DONE]` / `[NEEDS_HUMAN]` / `[INFO]`，每条消息明确意图
34|- 🛡️ **四层防死循环**: 内容相似度检测 → 连续发言限制 → Turn 软限制 → 全局超时兜底
35|- ✋ **人类打断**: 用户一说话，Bot 对话立即暂停
36|- 📦 **开箱即用**: 配置模板 + Hermes Skill 包 + 实战案例
37|
38|### 快速开始
39|
40|详细步骤见 [飞书接入指南](feishu/setup-guide.md)。
41|
42|```bash
43|# 1. 复制配置模板
44|cp feishu/config-templates/.env.example ~/.hermes-bot-a/.env
45|cp feishu/config-templates/.env.example ~/.hermes-bot-b/.env
46|# 编辑 .env，填入各自的飞书应用凭证
47|
48|# 2. 安装 Skill（让 Bot 遵守协作协议）
49|cp -r skills/hermes-a2a/ ~/.hermes-bot-a/skills/hermes-a2a/
50|cp -r skills/hermes-a2a/ ~/.hermes-bot-b/skills/hermes-a2a/
51|
52|# 3. 启动
53|HERMES_HOME=~/.hermes-bot-a hermes gateway start
54|HERMES_HOME=~/.hermes-bot-b hermes gateway start
55|```
56|
57|#### 在飞书群里测试
58|
59|```
60|@Bot-A 帮我查一下服务器状态
61|@Bot-B 今天 API 用量是多少?
62|```
63|
64|### 实战案例
65|
66|| 案例 | 描述 | 涉及信号 |
67||------|------|---------|
68|| [基础双 Bot 教程](case-studies/basic-collaboration/) | 从零搭建两个 Bot 协作 | `[CONTINUE]` `[DONE]` |
69|| [Token 监控](case-studies/token-monitor/) | 跨机器 Token 用量 Dashboard | `[INFO]` `[DONE]` |
70|| [信箱系统](case-studies/inbox-system/) | Bot 间离线消息信箱 | `[CONTINUE]` `[INFO]` `[DONE]` |
71|
72|### 项目结构
73|
74|```
75|hermes-a2a/
76|├── README.md                          # 本文件
77|├── LICENSE                            # MIT 协议
78|├── ROADMAP.md                         # 版本演进计划
79|│
80|├── protocol/                          # 协议规范
81|│   ├── spec.md                        # A2A 通信协议完整规范
82|│   ├── rules.md                       # 11 条协作规则 + 系统提示词模板
83|│   └── anti-loop.md                   # 四层防死循环机制详解
84|│
85|├── docs/                              # 技术文档
86|│   ├── architecture.md                # 多 Agent 扩展架构设计
87|│   ├── collaboration-guide.md         # 协作规范与行为准则
88|│   └── troubleshooting.md             # 踩坑记录 & FAQ
89|│
90|├── feishu/                            # 飞书接入
91|│   ├── setup-guide.md                 # 完整接入指南
92|│   ├── bot-identity-guide.md          # Bot 身份识别机制
93|│   ├── thread-and-reply-guide.md      # 话题与回复控制
94|│   ├── patches/                       # 代码补丁说明
95|│   │   └── README.md                  # 补丁状态与维护策略
96|│   └── config-templates/              # 配置模板
97|│       ├── .env.example               # 环境变量模板
98|│       ├── config-bot-a.yaml          # Bot A 配置
99|│       └── config-bot-b.yaml          # Bot B 配置
100|│
101|├── skills/                            # Hermes Skill 包
102|│   └── hermes-a2a/
103|│       ├── SKILL.md                   # Skill 主文件
104|│       ├── references/                # 快速参考
105|│       └── templates/                 # 消息模板
106|│
107|└── case-studies/                          # 实战案例
108|    ├── basic-collaboration/                    # 双 Bot 基础教程
109|    ├── token-monitor/                 # Token 监控案例
110|    └── inbox-system/                  # 信箱系统案例
111|```
112|
113|### 文档导航
114|
| 我想... | 去看 |
|---------|------|
| **新手入门（推荐）** | **[docs/user-manual.md](docs/user-manual.md)** |
| 从零搭建飞书 Bot-to-Bot | [feishu/setup-guide.md](feishu/setup-guide.md) |
| 了解协议规范 | [protocol/spec.md](protocol/spec.md) |
| 学习防死循环机制 | [protocol/anti-loop.md](protocol/anti-loop.md) |
| 看看踩过的坑 | [docs/troubleshooting.md](docs/troubleshooting.md) |
| 了解多 Agent 架构 | [docs/architecture.md](docs/architecture.md) |
| 安装 Skill 包 | [skills/hermes-a2a/SKILL.md](skills/hermes-a2a/SKILL.md) |
123|
124|---
125|
126|## English
127|
128|### What is this?
129|
130|`hermes-a2a` is a **complete solution for multi-Agent collaboration on Feishu/Lark**, solving:
131|
132|> When you have multiple Hermes instances (e.g., one on a server, one on Windows), how do they work together in the same Feishu group chat — instead of working independently, looping infinitely, or competing to answer?
133|
134|### Core Features
135|
136|- 🤝 **Feishu-native collaboration**: Bot-to-bot communication via @mention, no extra middleware
137|- 🎯 **Role assignment**: Leader + Executor with clear task division
138|- 🏷️ **Four-level signal tags**: `[CONTINUE]` / `[DONE]` / `[NEEDS_HUMAN]` / `[INFO]`
139|- 🛡️ **Four-layer anti-loop**: content similarity → consecutive limit → turn soft cap → global timeout
140|- ✋ **Human interrupt**: user messages immediately pause Agent collaboration
141|- 📦 **Batteries included**: config templates + Hermes Skill + real-world examples
142|
143|### Quick Start
144|
145|See [feishu/setup-guide.md](feishu/setup-guide.md) for detailed instructions.
146|
147|```bash
148|# 1. Copy config templates
149|cp feishu/config-templates/.env.example ~/.hermes-bot-a/.env
150|cp feishu/config-templates/.env.example ~/.hermes-bot-b/.env
151|
152|# 2. Install Skill (makes Bots follow the protocol)
153|cp -r skills/hermes-a2a/ ~/.hermes-bot-a/skills/hermes-a2a/
154|cp -r skills/hermes-a2a/ ~/.hermes-bot-b/skills/hermes-a2a/
155|
156|# 3. Start both instances
157|HERMES_HOME=~/.hermes-bot-a hermes gateway start
158|HERMES_HOME=~/.hermes-bot-b hermes gateway start
159|```
160|
161|### Examples
162|
163|| Example | Description | Signals Used |
164||---------|-------------|-------------|
165|| [Basic Collaboration](case-studies/basic-collaboration/) | Build two Bots from scratch | `[CONTINUE]` `[DONE]` |
166|| [Token Monitor](case-studies/token-monitor/) | Cross-machine Token usage Dashboard | `[INFO]` `[DONE]` |
167|| [Inbox System](case-studies/inbox-system/) | Offline messaging between Bots | `[CONTINUE]` `[INFO]` `[DONE]` |
168|
169|### Project Structure
170|
171|```
172|hermes-a2a/
173|├── README.md
174|├── LICENSE
175|├── ROADMAP.md
176|├── protocol/           # Protocol specs (spec, rules, anti-loop)
177|├── docs/               # Architecture, troubleshooting, collaboration guide
178|├── feishu/             # Feishu integration (setup, bot identity, config templates)
179|├── skills/             # Hermes Skill package
180|└── case-studies/           # Real-world examples
181|```
182|
183|### Documentation
184|
| I want to... | Go to |
|--------------|-------|
| **Getting Started (Recommended)** | **[docs/user-manual.md](docs/user-manual.md)** |
| Set up Feishu Bot-to-Bot from scratch | [feishu/setup-guide.md](feishu/setup-guide.md) |
| Read the protocol spec | [protocol/spec.md](protocol/spec.md) |
| Understand anti-loop mechanisms | [protocol/anti-loop.md](protocol/anti-loop.md) |
| See common pitfalls | [docs/troubleshooting.md](docs/troubleshooting.md) |
| Learn about multi-Agent architecture | [docs/architecture.md](docs/architecture.md) |
| Install the Skill package | [skills/hermes-a2a/SKILL.md](skills/hermes-a2a/SKILL.md) |
193|
194|### License
195|
196|[MIT](LICENSE)
197|
198|---
199|
200|*Built with ❤️ by the Hermes community*
201|