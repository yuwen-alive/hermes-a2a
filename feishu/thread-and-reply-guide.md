# 飞书话题（Thread）与回复控制

> 理解 Hermes 在飞书群聊中如何处理话题和回复。

## 飞书消息中的 Thread 概念

飞书群聊支持**话题（Thread）**，即对某条消息的回复会形成一个话题链。

消息体中的关键字段：

```json
{
  "message_id": "om_xxxxxx",
  "root_id": "om_root_xxxxxx",    // 话题根消息 ID
  "parent_id": "om_parent_xxxxxx", // 直接父消息 ID
  "chat_id": "oc_xxxxxx"
}
```

## Hermes 的 Thread 处理

### Thread ID 提取

```python
# gateway/platforms/feishu.py 第 3099 行
thread_id = (
    getattr(message, "thread_id", None)
    or getattr(message, "root_id", None)
    or None
)
```

Hermes 优先使用 `thread_id`，其次 `root_id`。这决定了消息是否被归入同一个会话上下文。

### Thread 对 Session 的影响

- 同一 `thread_id` 的消息共享同一个 Session
- 不同 `thread_id` 的消息创建独立 Session
- 无 `thread_id` 的消息按 `chat_id` + `user_id` 路由

### Bot-to-Bot 场景下的 Thread 行为

当 Bot A @Bot B 时，Bot B 的回复默认在**同一个 Thread** 中。这有利于保持对话上下文，但也意味着：

1. ✅ 对话有上下文连续性
2. ⚠️ 如果 Thread 中消息过多，可能触发 token 限制
3. ⚠️ Thread 内的 Bot 对话可能打扰同一 Thread 中的人类用户

### `thread_sessions_per_user` 配置

```yaml
# config.yaml
feishu:
  thread_sessions_per_user: false   # 默认
```

| 值 | 行为 |
|----|------|
| `false`（默认） | 同一 Thread 中所有用户共享一个 Session |
| `true` | 同一 Thread 中每个用户独立 Session |

Bot-to-Bot 场景建议保持 `false`，让两个 Bot 共享上下文。

## 话题自动创建

飞书中，Bot 回复消息时可能自动创建新话题。目前 Hermes 不直接控制此行为——话题是否创建取决于飞书客户端和 API 的 `reply_to` 参数。

> **后续优化方向**：通过控制 `reply_to` 参数来决定是在原话题中回复还是创建新话题。此功能列入 `hermes-a2a` Roadmap V2。

## Bot-to-Bot 最佳实践

1. **使用 Thread 隔离**：Bot 间的任务对话在独立 Thread 中进行
2. **及时收敛**：任务完成后发送 `[DONE]` 信号，避免在 Thread 中持续对话
3. **人类打断**：用户在同一群中发新消息时，Bot 应暂停 Thread 中的对话

---

*最后更新：2026-06-20*
