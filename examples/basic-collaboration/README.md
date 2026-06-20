# 双 Bot 基础协作教程

> 从零搭建两个 Hermes Agent，在飞书群中完成第一次 Bot-to-Bot 协作对话。

## 你会得到什么

完成本教程后，你将拥有：

- 两个独立的 Hermes 实例（Bot A + Bot B），运行在同一台机器上
- 一个飞书群，两个 Bot 互发消息协作完成任务
- 防死循环协议自动生效，不用担心 Bot 互相回复停不下来

## 前置条件

| 条件 | 说明 |
|------|------|
| Hermes Agent | >= v0.15.1（已内置 FEISHU_ALLOW_BOTS） |
| 飞书自建应用 | 两个（各自有 App ID / App Secret） |
| 飞书群 | 一个，将两个 Bot 都添加进去 |
| LLM API Key | 至少一个（OpenAI / DeepSeek / 其他兼容 OpenAI 格式的） |

> **为什么需要两个飞书应用？**
> 每个 Bot 需要独立的身份（App ID），否则飞书无法区分消息来源，Bot 会自己回复自己。

---

## 第一步：创建飞书应用

### 创建应用 A（Bot A）

1. 打开 [飞书开放平台](https://open.feishu.cn/app)
2. 点击「创建自建应用」
3. 填写应用名称（如 "Bot A"），创建完成
4. 记下 **App ID** 和 **App Secret**

### 开通权限

进入「权限管理」，开通以下权限：

- `im:message` — 获取与发送消息
- `im:message.group_at_msg` — 接收群聊 @ 消息
- `im:chat` — 获取群组信息

### 添加事件订阅

进入「事件订阅」：

- 添加事件：`im.message.receive_v1`（接收消息）
- 配置请求地址（如果用 WebSocket 模式可跳过）

### 添加机器人能力

进入「应用能力」-> 添加「机器人」

### 发布应用

提交审核 -> 审核通过

### 创建应用 B

重复上述步骤，使用**不同的**应用名称（如 "Bot B"）。你会得到另一组 App ID / App Secret。

---

## 第二步：创建飞书群并添加 Bot

1. 在飞书中创建一个新群（如 "Bot 协作测试"）
2. 将 Bot A 和 Bot B 都添加到群里
3. 确保两个 Bot 在群内都有发言权限

---

## 第三步：获取 Bot Open ID

每个 Bot 需要知道自己的 `open_id`（身份标识），用于防止自回环。

### 方法 1：通过 API 获取

```bash
# 第 1 步：获取 tenant_access_token
curl -s -X POST 'https://open.feishu.cn/open-apis/auth/v3/tenant_access_token/internal' \\
  -H 'Content-Type: application/json' \\
  -d '{"app_id": "你的APP_ID", "app_secret": "你的APP_SECRET"}'
```

```bash
# 第 2 步：用 token 查询 Bot 的 open_id
curl -s -X GET 'https://open.feishu.cn/open-apis/bot/v3/info' \\
  -H 'Authorization: Bearer 上一步返回的tena...oken'
```

返回 JSON 中的 `open_id` 字段就是 `FEISHU_BOT_OPEN_ID`。

### 方法 2：通过 Hermes 日志

先不设置 `FEISHU_BOT_OPEN_ID`，启动 Hermes 后查看日志：

```bash
hermes gateway start 2>&1 | grep -i "bot.*open_id"
```

Hermes 会自动调用 API 获取并打印。

> **重要**：两个 Bot 的 open_id 一定不同，不要搞混。记下来备用。

---

## 第四步：配置 Hermes 实例

### 目录结构

```
~/.hermes-bot-a/          # Bot A 的配置目录
+-- .env                  # 飞书凭证 + LLM 配置
+-- config.yaml           # Hermes 配置
+-- skills/
    +-- hermes-a2a/       # 协作协议 Skill（第五步安装）

~/.hermes-bot-b/          # Bot B 的配置目录
+-- .env
+-- config.yaml
+-- skills/
    +-- hermes-a2a/
```

### 创建目录

```bash
mkdir -p ~/.hermes-bot-a/skills ~/.hermes-bot-b/skills
```

### 编辑 .env 文件

从仓库中复制模板，然后编辑填入你的值：

```bash
cp feishu/config-templates/.env.example ~/.hermes-bot-a/.env
cp feishu/config-templates/.env.example ~/.hermes-bot-b/.env
```

每个 `.env` 文件需要填写以下关键配置：

**Bot 身份相关（两个 Bot 各自不同）：**

| 变量 | 说明 | 示例 |
|------|------|------|
| FEISHU_APP_ID | 飞书应用 ID | cli_a1b2c3d4e5 |
| FEISHU_BOT_OPEN_ID | Bot 的身份标识（第三步获取） | ou_a1b2c3d4e5... |

**通信开关（两个 Bot 相同）：**

```env
FEISHU_ALLOW_BOTS=mentions
```

这是启用 Bot-to-Bot 通信的关键开关。`mentions` 表示只处理 @ 自己的 Bot 消息（推荐）。

**其他飞书凭证和 LLM 配置：** 按 `.env.example` 模板中的注释填写即可。

> **易错点**：`FEISHU_BOT_OPEN_ID` 两个 Bot 一定不同！填错了会导致自回环（Bot 回复自己）。

### 编辑 config.yaml

从仓库复制模板：

```bash
cp feishu/config-templates/config-bot-a.yaml ~/.hermes-bot-a/config.yaml
cp feishu/config-templates/config-bot-b.yaml ~/.hermes-bot-b/config.yaml
```

编辑每个 `config.yaml`，把 `a2a.peers` 中的 `open_id` 替换为对端 Bot 的真实 open_id：

**Bot A 的 config.yaml**（关键部分）：

```yaml
a2a:
  role: leader
  name: "Bot A"
  peers:
    - name: "Bot B"
      open_id: "Bot-B 的 open_id"
```

**Bot B 的 config.yaml**（关键部分）：

```yaml
a2a:
  role: executor
  name: "Bot B"
  peers:
    - name: "Bot A"
      open_id: "Bot-A 的 open_id"
```

---

## 第五步：安装协作协议 Skill

让 Bot 知道协作规则（信号标签、防死循环等）：

```bash
# 从 hermes-a2a 仓库复制 skill 到两个 Bot
cp -r skills/hermes-a2a/ ~/.hermes-bot-a/skills/hermes-a2a/
cp -r skills/hermes-a2a/ ~/.hermes-bot-b/skills/hermes-a2a/
```

安装后，Bot 检测到需要与其他 Bot 协作时，会自动加载协议规则。

> **不装 Skill 行不行？** 技术上可以，但 Bot 不知道信号标签规则，容易死循环。强烈建议安装。

---

## 第六步：启动

打开两个终端窗口：

```bash
# 终端 1：启动 Bot A
HERMES_HOME=~/.hermes-bot-a hermes gateway start

# 终端 2：启动 Bot B
HERMES_HOME=~/.hermes-bot-b hermes gateway start
```

看到类似以下日志表示启动成功：

```
Feishu gateway started
Bot open_id: ou_xxx...
Allow bots: mentions
```

---

## 第七步：验证

在飞书群里逐项测试：

### 测试 1：用户 @ Bot A

```
@Bot A 你好，介绍一下你自己
```

**预期**：Bot A 回复一段自我介绍。

### 测试 2：用户 @ Bot B

```
@Bot B 你好，介绍一下你自己
```

**预期**：Bot B 回复一段自我介绍。

### 测试 3：Bot 间通信（核心！）

```
@Bot A 请让 Bot B 帮我查一下今天是星期几
```

**预期流程**：

```
用户:   @Bot A 请让 Bot B 帮我查一下今天是星期几
Bot A:  @Bot B 帮用户查一下今天是星期几 [CONTINUE]
Bot B:  @Bot A 今天是星期六 [DONE]
Bot A:  @用户 Bot B 说今天是星期六
```

**关键观察**：
- Bot B 的消息带 `[DONE]` 标签
- Bot A 收到 `[DONE]` 后**不再回复** Bot B
- Bot A 最终把结果汇报给用户

### 测试 4：验证防死循环

```
@Bot A 和 Bot B 讨论一下哪个编程语言最好，要求达成共识
```

**预期**：
- 两个 Bot 讨论几轮后带 `[DONE]` 结束
- 或者无法达成共识时带 `[NEEDS_HUMAN]`
- **不会无限循环**

### 测试 5：验证 @ 规则

在群里发一条不 @ 任何 Bot 的消息：

```
大家好，今天天气不错
```

**预期**：两个 Bot 都不响应。

---

## 验证清单

| # | 测试项 | 预期结果 | 通过 |
|---|--------|---------|------|
| 1 | @Bot A | Bot A 响应 | [ ] |
| 2 | @Bot B | Bot B 响应 | [ ] |
| 3 | Bot A -> Bot B | Bot B 收到并响应 | [ ] |
| 4 | Bot B -> Bot A | Bot A 收到并响应 | [ ] |
| 5 | 信号标签 | 消息末尾带 [CONTINUE]/[DONE] | [ ] |
| 6 | 防死循环 | 讨论最终终止 | [ ] |
| 7 | 不 @ 时 | 两个 Bot 都不响应 | [ ] |

全部打勾 = 搭建成功

---

## 常见问题

### Bot A 发消息后 Bot B 没反应？

1. 检查 Bot B 的 `.env` 中 `FEISHU_ALLOW_BOTS=mentions` 是否设置
2. 检查 Bot B 的 `FEISHU_BOT_OPEN_ID` 是否正确
3. 查看 Bot B 的日志是否有 `bot_not_mentioned` 或 `bots_disabled`

### 两个 Bot 互相回复停不下来？

1. 确认安装了 `hermes-a2a` skill
2. 检查消息是否带信号标签（`[CONTINUE]` / `[DONE]`）
3. 如果没装 skill，手动在系统提示词中加入协议规则（见 `protocol/rules.md`）

### Bot 报 429 错误？

两个 Bot 同时调 API 会触发限流。解决方案：
- 让一个 Bot 先完成，另一个再开始（错峰）
- 或者给两个 Bot 配不同的 API Key

### 启动报错说缺少环境变量？

`.env` 文件路径不对。确认：

```bash
ls -la ~/.hermes-bot-a/.env  # 文件存在？
cat ~/.hermes-bot-a/.env     # 内容正确？
```

---

## 下一步

- 了解完整协议规范：`protocol/spec.md`
- 学习防死循环机制：`protocol/anti-loop.md`
- 查看更多实战案例：`examples/`
- 部署到多台机器：`docs/architecture.md`

---

*本教程是 hermes-a2a 项目的一部分。*
