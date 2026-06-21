# hermes-a2a 产品手册

> **从零到跑通：让你的两个 AI Agent 在飞书群中安全协作**
>
> 本手册面向新手用户，手把手带你完成从安装到使用的全流程。预计 30 分钟内可以跑通。

---

## 目录

1. [hermes-a2a 是什么？](#1-hermes-a2a-是什么)
2. [你需要准备什么？](#2-你需要准备什么)
3. [第一步：创建飞书应用](#3-第一步创建飞书应用)
4. [第二步：安装 Hermes Agent](#4-第二步安装-hermes-agent)
5. [第三步：下载 hermes-a2a 项目](#5-第三步下载-hermes-a2a-项目)
6. [第四步：配置两个 Bot](#6-第四步配置两个-bot)
7. [第五步：安装协作 Skill](#7-第五步安装协作-skill)
8. [第六步：启动并验证](#8-第六步启动并验证)
9. [进阶：让两个 Bot 协作完成任务](#9-进阶让两个-bot-协作完成任务)
10. [常见问题排查](#10-常见问题排查)
11. [附录：配置速查表](#11-附录配置速查表)

---

## 1. hermes-a2a 是什么？

**一句话**：让两个（或更多）AI Agent 在同一个飞书群中互相协作，而不至于死循环、抢活、或打扰你。

### 举个例子 🌰

你有两个 Hermes Agent：
- **Bot A**（小马）：跑在 Windows 电脑上，擅长代码和本地操作
- **Bot B**（小牛）：跑在 Linux 服务器上，擅长部署和自动化

你想让它们一起帮你完成一个任务（比如"帮我分析这个数据集"）。没有 hermes-a2a 的话：
- 两个 Bot 可能**同时冲上去**做同一件事
- 一个回复另一个，另一个又回复回来——**无限循环**
- Token 费用飙升，你根本**插不上话**

有了 hermes-a2a：
- Bot A 作为 **Leader** 分配任务，Bot B 作为 **Executor** 执行
- 每条消息带信号标签 `[CONTINUE]`/`[DONE]`，说完了就**自动停止**
- 你随时说一句话，Bot 对话**立即暂停**，优先处理你的需求

---

## 2. 你需要准备什么？

在开始之前，请确认你有以下条件：

| # | 准备项 | 说明 |
|---|--------|------|
| ✅ | **两台设备**（或两台虚拟机） | 可以是一台 Linux 服务器 + 一台 Windows/Mac 电脑，也可以是两台服务器 |
| ✅ | **Hermes Agent ≥ v0.15.1** | 两台设备上都要装好 Hermes（如果还没装，见第 4 步） |
| ✅ | **一个飞书账号** | 用来创建飞书应用和建群 |
| ✅ | **一个飞书群** | 两个 Bot 都添加进去的测试群 |
| ✅ | **LLM API Key** | 比如 OpenAI、DeepSeek、小米 MiMo 等，Bot 需要调用大模型 |

> 💡 **新手提示**：如果你只有一台设备，也可以用 Hermes 的 Profile 功能在同一台机器上跑两个实例。详见[第 6 步中的"单机部署方案"](#方案-b单机部署用-profile-隔离)。

---

## 3. 第一步：创建飞书应用

你需要**两个独立的飞书自建应用**（两个 Bot 各一个）。

### 3.1 创建应用 A

1. 打开 [飞书开放平台](https://open.feishu.cn/app)，登录你的飞书账号
2. 点击 **创建自建应用**
3. 填写应用名称（如 `Bot-A`）和描述，点确认
4. 进入应用 → 记录 **App ID** 和 **App Secret**（后面要用）

### 3.2 配置应用 A 的权限

在应用详情页，依次操作：

1. **添加能力** → 选择 **机器人**
2. **权限管理** → 搜索并开通以下权限：
   - `im:message` — 收发消息
   - `im:message.group_at_msg` — 接收群聊中 @ 机器人的消息
   - `im:message.group_msg` — 接收群聊中所有消息（可选）
   - `im:chat` — 获取群组信息
3. **事件订阅** → 添加事件：
   - `im.message.receive_v1` — 接收消息事件
4. **发布应用** → 提交审核 → 等待通过

### 3.3 创建应用 B

**重复上述步骤**，创建第二个应用。注意：
- 使用**不同的**应用名称（如 `Bot-B`）
- 获取**不同的** App ID 和 App Secret
- 同样配置权限和事件订阅

### 3.4 建群并添加 Bot

1. 在飞书中创建一个新群（如 `A2A 测试群`）
2. 进入群设置 → 群机器人 → 添加 Bot A 和 Bot B
3. 记录**群的 chat_id**（可选，高级配置时用）

---

## 4. 第二步：安装 Hermes Agent

如果两台设备上已经安装了 Hermes Agent（≥ v0.15.1），跳过这步。

### 4.1 Linux / macOS 安装

```bash
# 一行命令安装
curl -fsSL https://raw.githubusercontent.com/NousResearch/hermes-agent/main/install.sh | bash

# 验证安装
hermes --version
```

### 4.2 Windows 安装

打开 PowerShell（管理员），执行：

```powershell
# 下载安装包
iwr -Uri "https://raw.githubusercontent.com/NousResearch/hermes-agent/main/install.ps1" -OutFile install.ps1
.\install.ps1

# 或者手动下载：https://github.com/NousResearch/hermes-agent/releases
# 解压到你喜欢的目录（比如 D:\Hermes）

# 验证安装
hermes --version
```

> 💡 **Windows 用户注意**：如果你不想装在 C 盘，可以把 Hermes 装到 `D:\Hermes`。安装后 `hermes` 命令需要在 PowerShell 中使用。

---

## 5. 第三步：下载 hermes-a2a 项目

在**两台设备上都执行**以下操作：

### Linux / macOS

```bash
# clone 项目
git clone https://github.com/yuwen-alive/hermes-a2a.git ~/hermes-a2a

# 确认文件完整
ls ~/hermes-a2a/
# 应该看到: README.md  protocol/  feishu/  skills/  docs/  case-studies/
```

### Windows（PowerShell）

```powershell
# clone 项目
git clone https://github.com/yuwen-alive/hermes-a2a.git "$env:USERPROFILE\hermes-a2a"

# 确认文件完整
ls "$env:USERPROFILE\hermes-a2a"
```

---

## 6. 第四步：配置两个 Bot

这是**最关键的一步**。你需要为每个 Bot 创建独立的配置目录和环境变量。

### 6.1 创建配置目录

**Bot A（Linux 示例）**：
```bash
mkdir -p ~/.hermes-bot-a
```

**Bot B（可以是同一台机器或另一台）**：
```bash
mkdir -p ~/.hermes-bot-b
```

### 6.2 创建环境变量文件

**Bot A 的 `.env`**（`~/.hermes-bot-a/.env`）：

```env
# ====== 飞书凭证（来自第 3 步） ======
FEISHU_APP_ID=cli_aaaaaaaaaaaaa
FEISHU_APP_SECRET=你的App-A-Secret
FEISHU_ENCRYPT_KEY=你的加密密钥（可选）
FEISHU_VERIFICATION_TOKEN=你的验证Token

# ====== Bot 身份标识（重要！） ======
# 不设置会导致自回环或无法识别对端 Bot
# 获取方法见下方说明
FEISHU_BOT_OPEN_ID=ou_aaaaaaaaaaaaaaaaaaaaaaa

# ====== Bot-to-Bot 通信开关 ======
# none=关闭（默认）| mentions=只响应@自己的Bot消息（推荐）| all=接受所有Bot消息
FEISHU_ALLOW_BOTS=mentions

# ====== LLM API ======
OPENAI_API_KEY=sk-你的API-Key
# 或者用其他兼容的 API：
# OPENAI_API_BASE=https://api.deepseek.com/v1
# OPENAI_API_KEY=sk-你的DeepSeek-Key
```

**Bot B 的 `.env`**（`~/.hermes-bot-b/.env`）：

```env
# ====== 飞书凭证（来自应用 B） ======
FEISHU_APP_ID=cli_bbbbbbbbbbbbb
FEISHU_APP_SECRET=你的App-B-Secret
FEISHU_ENCRYPT_KEY=你的加密密钥（可选）
FEISHU_VERIFICATION_TOKEN=你的验证Token

# ====== Bot 身份标识 ======
FEISHU_BOT_OPEN_ID=ou_bbbbbbbbbbbbbbbbbbbbbbb

# ====== Bot-to-Bot 通信开关 ======
FEISHU_ALLOW_BOTS=mentions

# ====== LLM API ======
OPENAI_API_KEY=sk-你的API-Key
```

### 6.3 如何获取 FEISHU_BOT_OPEN_ID？

这是**最容易被忽略但最关键**的一步。`FEISHU_BOT_OPEN_ID` 是 Bot 在飞书中的唯一身份标识，Hermes 用它来区分"自己发的消息"和"对端 Bot 发的消息"。

**方法一：API 调用（推荐）**

```bash
# 1. 获取 tenant_access_token
curl -X POST 'https://open.feishu.cn/open-apis/auth/v3/tenant_access_token/internal' \
  -H 'Content-Type: application/json' \
  -d '{"app_id":"cli_你的AppID","app_secret":"你的AppSecret"}'

# 返回 JSON 中找到 "tenant_access_token" 字段

# 2. 用 token 获取 Bot 信息
curl -X GET 'https://open.feishu.cn/open-apis/bot/v3/info' \
  -H 'Authorization: Bearer t-你的tenant_access_token'

# 返回 JSON 中找到 data.bot.open_id，这就是 FEISHU_BOT_OPEN_ID
```

**方法二：查看日志**

如果未手动设置 `FEISHU_BOT_OPEN_ID`，Hermes 会在启动时自动调用上述 API 获取。查看日志：

```bash
hermes gateway start 2>&1 | grep bot_open_id
```

> ⚠️ **建议手动设置**：自动发现在高并发启动时可能有竞态条件。

### 6.4 创建 config.yaml

**Bot A 的 config.yaml**（`~/.hermes-bot-a/config.yaml`）：

```yaml
feishu:
  allow_bots: mentions       # 接受 @ 自己的其他 Bot 消息
  require_mention: true       # 群聊中需要 @ 才响应
  group_policy: open          # 允许加入任何群

gateway:
  enabled: true
```

**Bot B 的 config.yaml**（`~/.hermes-bot-b/config.yaml`）：

```yaml
feishu:
  allow_bots: mentions
  require_mention: true
  group_policy: open

gateway:
  enabled: true
```

### 方案 B：单机部署（用 Profile 隔离）

如果你**只有一台机器**，可以用 Hermes 的 Profile 功能：

```bash
# 创建 Profile 目录
mkdir -p ~/.hermes/profiles/bot-a
mkdir -p ~/.hermes/profiles/bot-b

# 分别在两个 Profile 目录下放 .env 和 config.yaml
# 启动时：
hermes --profile bot-a gateway start   # 终端 1
hermes --profile bot-b gateway start   # 终端 2
```

---

## 7. 第五步：安装协作 Skill

Skill 包让 Bot "懂得"如何遵守 A2A 协作协议——信号标签、防死循环、角色分配等规则。

### 7.1 复制 Skill 到两个 Bot

**Bot A**：
```bash
cp -r ~/hermes-a2a/skills/hermes-a2a/ ~/.hermes-bot-a/skills/hermes-a2a/
```

**Bot B**：
```bash
cp -r ~/hermes-a2a/skills/hermes-a2a/ ~/.hermes-bot-b/skills/hermes-a2a/
```

> 💡 如果使用 Profile 模式：
> ```bash
> cp -r ~/hermes-a2a/skills/hermes-a2a/ ~/.hermes/profiles/bot-a/skills/hermes-a2a/
> cp -r ~/hermes-a2a/skills/hermes-a2a/ ~/.hermes/profiles/bot-b/skills/hermes-a2a/
> ```

### 7.2 验证 Skill 安装

```bash
# 确认文件存在
ls ~/.hermes-bot-a/skills/hermes-a2a/
# 应该看到: SKILL.md  references/  templates/

ls ~/.hermes-bot-b/skills/hermes-a2a/
# 同样
```

---

## 8. 第六步：启动并验证

### 8.1 启动两个 Bot

打开**两个终端窗口**：

**终端 1 — 启动 Bot A**：
```bash
HERMES_HOME=~/.hermes-bot-a hermes gateway start
```

**终端 2 — 启动 Bot B**：
```bash
HERMES_HOME=~/.hermes-bot-b hermes gateway start
```

> 💡 如果使用 Profile 模式：
> ```bash
> # 终端 1
> hermes --profile bot-a gateway start
> # 终端 2
> hermes --profile bot-b gateway start
> ```

### 8.2 看到什么算成功？

两个终端都应该显示类似：
```
✅ Gateway started
✅ Feishu platform connected
✅ Bot info loaded: open_id=ou_xxx
```

如果看到 `❌` 或 `ERROR`，请跳转到[第 10 节：常见问题排查](#10-常见问题排查)。

### 8.3 验证清单

在飞书群里逐项测试：

| # | 测试操作 | 预期结果 | 失败？看 |
|---|---------|---------|---------|
| 1 | `@Bot-A 你好` | Bot A 回复你 | 检查 App 权限和事件订阅 |
| 2 | `@Bot-B 你好` | Bot B 回复你 | 同上 |
| 3 | `@Bot-A @Bot-B 帮我查一下XX` | Bot A 收到后 @ Bot B 分配任务，Bot B 执行并回复 | 检查 FEISHU_ALLOW_BOTS 和 FEISHU_BOT_OPEN_ID |
| 4 | 在 Bot 对话中直接发一条消息 | Bot 对话立即暂停，优先处理你的消息 | 检查 require_mention 设置 |

---

## 9. 进阶：让两个 Bot 协作完成任务

### 9.1 基本协作模式

在飞书群里，像这样下达任务：

```
@小马 @小牛 帮我分析一下服务器上 ~/data/ 目录里的 CSV 文件，总结关键指标
```

**正常流程**：
1. **小马**（Leader）收到任务 → 分析任务 → @小牛 分配执行部分
2. **小牛**（Executor）收到指令 → 执行分析 → @小马 汇报结果
3. **小马** 收到结果 → 汇总 → @你 汇报最终结果
4. 每条消息带 `[CONTINUE]`/`[DONE]` 标签，任务完成后自动停止

### 9.2 信号标签说明

| 信号 | 含义 | 什么时候用 | 收到后怎么做 |
|------|------|-----------|-------------|
| `[CONTINUE]` | 需要你回复 | 还有后续工作需要讨论 | 回复并继续对话 |
| `[DONE]` | 任务完成 | 任务做完了 | **不要回复**，对话结束 |
| `[NEEDS_HUMAN]` | 需要人类介入 | 遇到无法自行决定的问题 | 通知用户处理 |
| `[INFO]` | 纯通知 | 告知状态/进度，不需要回复 | **不要回复** |

### 9.3 防死循环机制

hermes-a2a 内置四层防护，你**不需要手动干预**：

1. **信号收敛**：收到 `[DONE]`/`[INFO]` → 不回复
2. **回复自检**：发送前检查"这句话有新信息吗？"→ 没有就不发
3. **内容去重**：和上一条相似度 > 80% → 不发
4. **轮次限制**：常规任务 ≤5 轮，复杂任务 ≤10 轮

### 9.4 人类打断

任何时候你在群里发消息，Bot 对话**立即暂停**，优先处理你的需求。处理完后 Bot 协作才会恢复。

---

## 10. 常见问题排查

### Q1: 启用 FEISHU_ALLOW_BOTS 后对端 Bot 消息仍被忽略

**最常见原因**：未设置 `FEISHU_BOT_OPEN_ID`。

Hermes 需要知道"哪个 open_id 是自己"才能区分自身消息和对端 Bot 消息。如果不设置，它会把所有 Bot 消息都当成自己的，全部丢弃。

**解决**：按照[第 6.3 节](#63-如何获取-feishu_bot_open_id)获取并设置 `FEISHU_BOT_OPEN_ID`。

---

### Q2: 两个 Bot 同时收到任务，同时执行

**原因**：没有明确 @ 某个 Bot，导致两个 Bot 都收到并处理。

**解决**：
1. 下达任务时明确 @ 一个 Bot（Leader）
2. 使用 Skill 中的角色分配机制，让 Leader 分配任务给 Executor
3. 在 `.env` 中设置 `FEISHU_ALLOW_BOTS=mentions`（不要用 `all`）

---

### Q3: 两个 Bot 互相回复停不下来（死循环）

**原因**：未正确安装 hermes-a2a Skill，Bot 不知道信号标签规则。

**解决**：
1. 确认 `~/.hermes-bot-a/skills/hermes-a2a/SKILL.md` 文件存在
2. 确认 `~/.hermes-bot-b/skills/hermes-a2a/SKILL.md` 文件存在
3. 重启两个 Bot

---

### Q4: 飞书 Bot 消息不触发 Webhook

**原因**：飞书默认不会把 Bot 自己发的消息推送回来（防回环）。这对 Bot-to-Bot 场景是个问题。

**解决**：Hermes v0.15.1 已内置 `FEISHU_ALLOW_BOTS` 功能，会自动处理这个情况。确保：
1. Hermes 版本 ≥ v0.15.1
2. `.env` 中设置了 `FEISHU_ALLOW_BOTS=mentions`
3. 两个 Bot 的 `FEISHU_BOT_OPEN_ID` 都正确配置

---

### Q5: Bot A @ Bot B，但 Bot B 没反应

**排查步骤**：
1. 检查 Bot B 的 `.env` 中 `FEISHU_ALLOW_BOTS` 是否为 `mentions` 或 `all`
2. 检查 Bot B 的 `FEISHU_BOT_OPEN_ID` 是否正确
3. 检查 Bot B 的 `config.yaml` 中 `feishu.allow_bots` 是否为 `mentions`
4. 查看 Bot B 的终端日志是否有报错

---

### Q6: Windows 和 Linux 之间如何协作？

两台机器上的 Hermes 实例是**完全独立的**，通过飞书服务器中转消息。也就是说：
- Bot A 在 Windows 上跑 → 发消息到飞书 → 飞书推送给 Bot B → Bot B 在 Linux 上收到
- **不需要两台机器直连**，只需要都能访问飞书 API

---

## 11. 附录：配置速查表

### 环境变量速查

| 变量名 | 必填？ | 说明 | 示例值 |
|--------|-------|------|-------|
| `FEISHU_APP_ID` | ✅ | 飞书应用 ID | `cli_a1b2c3d4e5` |
| `FEISHU_APP_SECRET` | ✅ | 飞书应用密钥 | `xxxxxxxxxxxxxxxx` |
| `FEISHU_BOT_OPEN_ID` | ✅ | Bot 自身的 Open ID | `ou_xxxxxxxxxxxxxxxx` |
| `FEISHU_ALLOW_BOTS` | ✅ | Bot-to-Bot 通信策略 | `mentions`（推荐） |
| `FEISHU_ENCRYPT_KEY` | ❌ | 事件加密密钥 | `your_key` |
| `FEISHU_VERIFICATION_TOKEN` | ❌ | 事件验证 Token | `your_token` |
| `OPENAI_API_KEY` | ✅ | LLM API 密钥 | `sk-xxxx` |
| `OPENAI_API_BASE` | ❌ | 自定义 API 地址 | `https://api.deepseek.com/v1` |

### config.yaml 速查

```yaml
feishu:
  allow_bots: mentions       # none | mentions | all
  require_mention: true       # 群聊中是否需要 @ 才响应
  group_policy: open          # open | allowlist | disabled

gateway:
  enabled: true               # 启用网关
```

### FEISHU_ALLOW_BOTS 三种模式

| 模式 | 行为 | 适用场景 |
|------|------|---------|
| `none`（默认） | 忽略所有其他 Bot 的消息 | 单 Bot 部署 |
| `mentions` | 只处理 @ 自己的 Bot 消息 | **Bot-to-Bot 推荐** |
| `all` | 接受所有 Bot 消息 | 高级场景（慎用，有循环风险） |

---

## 文档导航

| 我想... | 去看 |
|---------|------|
| 了解协议规范（信号标签、消息格式） | [protocol/spec.md](../protocol/spec.md) |
| 深入理解防死循环机制 | [protocol/anti-loop.md](../protocol/anti-loop.md) |
| 学习 Bot 身份识别原理 | [feishu/bot-identity-guide.md](../feishu/bot-identity-guide.md) |
| 了解话题和回复控制 | [feishu/thread-and-reply-guide.md](../feishu/thread-and-reply-guide.md) |
| 看更多踩坑记录 | [docs/troubleshooting.md](troubleshooting.md) |
| 了解多 Agent 扩展架构 | [docs/architecture.md](architecture.md) |
| 查看实战案例 | [case-studies/](../case-studies/) |

---

*最后更新：2026-06-21*
*作者：小马 🐴 + 小牛 🐮，由雨文协调*
