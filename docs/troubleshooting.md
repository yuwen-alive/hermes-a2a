# 踩坑记录 & FAQ

> 我们在搭建 Bot-to-Bot 通信过程中遇到的所有坑，以及解决方案。

---

## 坑 1: 启用 FEISHU_ALLOW_BOTS 后对端 Bot 消息仍被忽略

**症状**: 设置了 `FEISHU_ALLOW_BOTS=mentions`，但对端 Bot 的消息完全不触发任何响应。

**根因**: 未设置 `FEISHU_BOT_OPEN_ID`。Hermes 无法识别"哪个 open_id 是自己"，在自回环检测时将所有 Bot 消息(包括对端的)都当成了自己的。

**解决**:

```bash
# 查询 Bot 自身的 open_id
curl -X POST 'https://open.feishu.cn/open-apis/auth/v3/tenant_access_token/internal' \
  -H 'Content-Type: application/json' \
  -d '{"app_id":"cli_xxx","app_secret":"xxx"}' \
  | jq .tenant_access_token

curl -X GET 'https://open.feishu.cn/open-apis/bot/v3/info' \
  -H 'Authorization: Bearer t-xxx' \
  | jq .data.bot.open_id
```

将返回的 `open_id` 写入 `.env`:

```env
FEISHU_BOT_OPEN_ID=ou_xxxxxxxxxxxxxxxx
```

---

## 坑 2: 两个 Bot 同时收到任务同时执行 (资源争抢)

**症状**: 用户 @all 或者不明确 @ 某个 Bot，导致两个 Bot 同时处理同一个任务，同时撞 API 限流。

**真实案例**:
- 小马: `Rate limited. Waiting 5.9s (attempt 3/3)...` -> HTTP 429
- 小牛: `Rate limited. Waiting 2.3s (attempt 2/3)...` -> HTTP 429
- 两个人同时冲，同时撞墙，配额耗尽

**根因**: 没有 Turn 控制机制，多个 Bot 可以同时处理同一个任务。

**解决**: 使用 `hermes-a2a` 协议中的角色分配和 Turn 控制:
1. Leader 分配任务 -> Executor 执行
2. 一个任务同一时间只有一个 Agent 在处理
3. 使用 `[CONTINUE]`/`[DONE]` 信号控制对话流程

---

## 坑 3: 飞书 Bot 消息不触发 Webhook

**症状**: 飞书开放平台配置了事件订阅，但 Bot 发出的消息没有触发 `im.message.receive_v1` 事件。

**根因**: 飞书的**默认行为**是不将 Bot 自身发送的消息作为事件推送回来(防回环)。这对单 Bot 场景是好事，但对 Bot-to-Bot 场景造成了困惑。

**关键认知**:
- Bot A 发的消息 -> Bot A **不会**收到事件 (预期行为)
- Bot A 发的消息 -> Bot B **会**收到事件 (前提: `FEISHU_ALLOW_BOTS` 已启用)

这不是 Bug，是飞书的设计。你不需要自己发的消息触发自己的 webhook。

---

## 坑 4: 飞书群聊自动创建话题

**症状**: Bot 回复消息时，飞书自动创建了一个新话题(Thread)，而不是在原消息下回复。

**根因**: 飞书 API 的 `reply` 接口行为取决于 `message_id` 参数和消息是否已在话题中。

**当前状态**: 这是飞书平台行为，Hermes 尚未提供配置项控制。列入 Roadmap V2。

**Workaround**: 使用 `send_message` 而非 `reply`(如果不需要话题关联)。

---

## 坑 5: config.yaml 和 .env 配置冲突

**症状**: 在 `config.yaml` 中设置了 `feishu.allow_bots: mentions`，但不生效。

**根因**: 环境变量优先于 config.yaml。如果 `.env` 中有 `FEISHU_ALLOW_BOTS=none`，会覆盖 config.yaml 的设置。

**优先级**(从高到低):
1. 命令行参数
2. 环境变量 (`.env`)
3. `config.yaml`
4. 默认值

**解决**: 只在一个地方配置，推荐用 `.env`。

---

## 坑 6: 两个 Hermes 实例用同一个 App ID

**症状**: 消息路由混乱，A 实例收到了应该给 B 的消息。

**根因**: 飞书通过 App ID 区分应用。两个实例必须使用不同的飞书应用。

**解决**: 确保每个 Hermes 实例使用独立的飞书应用(独立 App ID / App Secret)。

---

## 坑 7: Bot @另一个 Bot 但对方不响应

**症状**: Bot A 在群里 @Bot B，但 Bot B 没有任何反应。

**排查清单**:

| 检查项 | 命令 |
|--------|------|
| `FEISHU_ALLOW_BOTS` 是否设置? | `grep ALLOW_BOTS .env` |
| `FEISHU_BOT_OPEN_ID` 是否正确? | 对比 API 返回值 |
| `require_mention` 是否为 true? | 检查 config.yaml |
| Bot B 是否在群里? | 飞书群设置 -> 机器人列表 |
| Bot B 的 gateway 是否在运行? | `ps aux | grep hermes` |
| 日志有没有 reject 原因? | `grep "reject\|admit\|bots_disabled" gateway.log` |

---

## FAQ

### Q: 三个以上的 Bot 怎么办?

当前协议设计支持 N 个 Agent。关键是:
1. 每个 Bot 使用独立的飞书应用
2. 所有 Bot 都添加到同一个群
3. Leader 负责协调，Executor 各司其职

### Q: 不同平台 (飞书 + Telegram) 能互通吗?

目前不支持，列入 Roadmap V3。核心难点是消息格式转换和身份映射。

### Q: Bot 对话会消耗大量 Token 吗?

会的。每轮对话都是一次 LLM API 调用。防死循环协议中的 Turn 限制(默认 5 轮)就是为了控制 Token 消耗。

### Q: 如何监控 Bot-to-Bot 的 Token 消耗?

建议搭建 Token Monitor(参考 `examples/token-monitor/`)，记录每次 API 调用的 Token 用量。

---

*最后更新: 2026-06-20*
