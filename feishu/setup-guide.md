# 飞书 Bot-to-Bot 接入指南

> 本文档指导你从零配置两个 Hermes Agent 在飞书群中互相通信。

## 前置条件

- Hermes Agent ≥ v0.15.1（已内置 `FEISHU_ALLOW_BOTS`）
- 两个独立的飞书自建应用（各自有 App ID / App Secret）
- 一个飞书群，将两个 Bot 都添加进去

---

## 第一步：创建两个飞书应用

### 应用 A（主协调人，如"小马"）

1. 打开 [飞书开放平台](https://open.feishu.cn/app)
2. 创建自建应用 → 获取 `App ID` 和 `App Secret`
3. 添加能力 → 机器人
4. 权限管理 → 开通以下权限：
   - `im:message` — 获取与发送单聊、群组消息
   - `im:message.group_at_msg` — 接收群聊中 @ 机器人消息
   - `im:message.group_msg` — 接收群聊中所有消息（可选，看是否需要非 @ 消息）
   - `im:chat` — 获取群组信息
5. 事件订阅 → 添加事件：
   - `im.message.receive_v1` — 接收消息
6. 发布应用 → 审核通过

### 应用 B（执行者，如"小牛"）

重复上述步骤，使用**不同的** App ID / App Secret。

---

## 第二步：配置 Hermes 实例

### 方案 A：双实例部署（推荐）

每个 Bot 运行独立的 Hermes 实例，使用不同的配置目录。

```
# Bot A（小马）
~/.hermes-bot-a/
├── config.yaml
├── .env              # App A 的凭证
├── skills/
└── sessions/

# Bot B（小牛）
~/.hermes-bot-b/
├── config.yaml
├── .env              # App B 的凭证
├── skills/
└── sessions/
```

### 方案 B：单实例 + Profile

利用 Hermes 的 Profile 功能，在同一进程中隔离两个 Bot。

```bash
hermes --profile bot-a gateway start
hermes --profile bot-b gateway start
```

Profile 配置目录：`~/.hermes/profiles/bot-a/` 和 `~/.hermes/profiles/bot-b/`

---

## 第三步：环境变量配置

### Bot A 的 `.env`

```env
# 飞书凭证
FEISHU_APP_ID=cli_xxxxxxxxxxxxx
FEISHU_APP_SECRET=xxxxxxxxxxxxxxxxxxxxxxxx
FEISHU_ENCRYPT_KEY=（可选）
FEISHU_VERIFICATION_TOKEN=xxxxxxxxxxxxxxxx

# 🔑 关键：识别自身身份（防止自己回复自己）
FEISHU_BOT_OPEN_ID=ou_xxxxxxxxxxxxxxxxxxxxxxxx

# 🔑 关键：允许接收其他 Bot 的消息
FEISHU_ALLOW_BOTS=mentions

# LLM 配置
OPENAI_API_KEY=sk-xxx
OPENAI_MODEL=gpt-4o
```

### Bot B 的 `.env`

```env
FEISHU_APP_ID=cli_yyyyyyyyyyyyy
FEISHU_APP_SECRET=yyyyyyyyyyyyyyyyyyyyyyyyyy
FEISHU_ENCRYPT_KEY=（可选）
FEISHU_VERIFICATION_TOKEN=yyyyyyyyyyyyyyyy

FEISHU_BOT_OPEN_ID=ou_yyyyyyyyyyyyyyyyyyyyyyyy

FEISHU_ALLOW_BOTS=mentions

OPENAI_API_KEY=sk-yyy
OPENAI_MODEL=gpt-4o
```

### `config.yaml`（两个实例通用模板）

```yaml
feishu:
  allow_bots: mentions   # none | mentions | all
  require_mention: true   # 群聊中需要 @ 才响应
  group_policy: open      # open | allowlist | disabled

gateway:
  enabled: true
```

> **优先级规则**：环境变量 `FEISHU_ALLOW_BOTS` > `config.yaml` 中的 `feishu.allow_bots`

---

## 第四步：获取 Bot Open ID

`FEISHU_BOT_OPEN_ID` 是 Bot 识别自身身份的关键。获取方法：

### 方法 1：API 调用

```bash
curl -X POST 'https://open.feishu.cn/open-apis/auth/v3/tenant_access_token/internal' \
  -H 'Content-Type: application/json' \
  -d '{
    "app_id": "cli_xxxxxxxxxxxxx",
    "app_secret": "xxxxxxxxxxxxxxxxxxxxxxxx"
  }'
```

用返回的 `tenant_access_token`：

```bash
curl -X GET 'https://open.feishu.cn/open-apis/bot/v3/info' \
  -H 'Authorization: Bearer t-xxxxxxxxxxxxxxx'
```

返回的 `open_id` 就是 `FEISHU_BOT_OPEN_ID`。

### 方法 2：Hermes 日志

启动 Hermes 后，如果未设置 `FEISHU_BOT_OPEN_ID`，Hermes 会在启动时自动调用 `/bot/v3/info` 获取。查看日志：

```bash
tail -f ~/.hermes/logs/gateway.log | grep bot_open_id
```

---

## 第五步：启动并验证

```bash
# 终端 1：启动 Bot A
HERMES_HOME=~/.hermes-bot-a hermes gateway start

# 终端 2：启动 Bot B
HERMES_HOME=~/.hermes-bot-b hermes gateway start
```

### 验证清单

| 测试 | 预期结果 |
|------|---------|
| @Bot A 发消息 | Bot A 响应 |
| @Bot B 发消息 | Bot B 响应 |
| Bot A @Bot B | Bot B 收到并响应 |
| Bot B @Bot A | Bot A 收到并响应 |
| 群里普通聊天（不@） | 两个 Bot 都不响应 |

---

## FEISHU_ALLOW_BOTS 三个模式详解

| 模式 | 行为 | 适用场景 |
|------|------|---------|
| `none`（默认） | 忽略所有其他 Bot 的消息 | 单 Bot 部署 |
| `mentions` | 只处理 @ 自己的 Bot 消息 | **Bot-to-Bot 推荐** |
| `all` | 接受所有 Bot 消息 | 高级场景（慎用，有循环风险） |

### 内部实现原理

```python
# gateway/platforms/feishu.py — _admit() 方法
if is_bot:
    mode = self._allow_bots
    if mode != "mentions" and mode != "all":
        return "bots_disabled"       # ← 默认拒绝所有 Bot 消息
    if not self_ids or not sender_ids:
        return "self_ids_unknown"    # ← 未配置 BOT_OPEN_ID
    if mode == "mentions" and not self._mentions_self(message):
        return "bot_not_mentioned"   # ← mentions 模式下未 @ 自己
```

关键安全机制：
1. **自回环保护**：`sender_ids & self_ids` 检测，防止自己回复自己
2. **身份识别**：依赖 `FEISHU_BOT_OPEN_ID` 准确配置
3. **渐进放行**：`none` → `mentions` → `all`，逐级开放

---

## 常见问题

### Q: 启用后对端 Bot 消息仍被忽略？

**最常见原因**：未设置 `FEISHU_BOT_OPEN_ID`。

Hermes 需要知道"哪个 open_id 是自己"才能区分自身消息和对端 Bot 消息。

### Q: 两个 Bot 互相回复停不下来？

这是**死循环问题**，请配合使用 `hermes-a2a` 的防死循环协议。详见 [防死循环文档](../protocol/anti-loop.md)。

### Q: 可以在同一个飞书应用里用两个 Bot 吗？

不可以。每个 Bot 需要独立的飞书应用（独立的 App ID），否则无法区分消息来源。

---

*最后更新：2026-06-20*
