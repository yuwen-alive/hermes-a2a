# A2A Protocol Specification v1.0

> Agent-to-Agent Safe Communication Protocol

## 1. Overview

The A2A Protocol defines a set of rules for multiple AI agents to communicate safely in a shared chat environment. It prevents infinite loops, reduces token waste, and ensures human users always have priority.

**Core Problem**: When two or more AI bots can see each other's messages in a group chat, they risk entering an infinite ping-pong loop — each bot replying to the other endlessly, burning tokens and spamming notifications.

**Solution**: Every inter-agent message carries a **signal tag** that explicitly declares whether a reply is expected.

---

## 2. Signal Tags

Every bot-to-bot message **MUST** end with exactly one signal tag on its own line:

| Signal | Meaning | Recipient Action |
|--------|---------|-----------------|
| `[CONTINUE]` | Conversation has substance, expecting a reply | Reply allowed |
| `[DONE]` | Task complete, nothing more to say | **Do NOT reply** |
| `[NEEDS_HUMAN]` | Stuck or need human input | **Do NOT reply**, escalate to user |
| `[INFO]` | FYI/status update, no reply needed | **Do NOT reply**, absorb silently |

### Format Rules

```
This is the message content.
It can be multiple lines.

[DONE]
```

- Signal tag **MUST** be the **last line** of the message
- Signal tag **MUST** be on its own line (no other text on the same line)
- One message, one signal — no nested or multiple signals
- Signal tags are case-sensitive: `[done]` is invalid, `[DONE]` is correct

### Extended Signals (Optional)

For debugging in 3+ agent scenarios, signals can carry an origin tag:

```
[DONE:task_complete]
[DONE:user_interrupted]
[NEEDS_HUMAN:permission_denied]
[INFO:progress_60pct]
```

---

## 3. Message Flow

### 3.1 Basic Two-Agent Flow

```
Agent A → Agent B: "请帮我分析这段代码" [CONTINUE]
Agent B → Agent A: "分析结果如下：..." [DONE]
                  ← Agent B does NOT reply further
```

### 3.2 Multi-Turn Flow

```
Agent A → Agent B: "需求不清楚，需要澄清" [CONTINUE]
Agent B → Agent A: "具体是这三个问题：..." [CONTINUE]
Agent A → Agent B: "明白了，方案如下：..." [DONE]
                  ← Conversation ends
```

### 3.3 Escalation Flow

```
Agent A → Agent B: "这个API返回403，我没有权限" [NEEDS_HUMAN]
                  ← Agent B does NOT reply, waits for human
Human → Agent A:  "我来处理权限问题"
```

### 3.4 Information Broadcast

```
Agent A → Agent B: "任务进度：60%完成" [INFO]
                  ← Agent B absorbs silently, does NOT reply
```

---

## 4. Agent Roles

### 4.1 Two-Agent Model

| Role | Responsibilities |
|------|-----------------|
| **Leader** | Task assignment, final report to human, coordination |
| **Executor** | Task execution, status reporting back to Leader |

### 4.2 N-Agent Model (3+ agents)

| Role | Responsibilities |
|------|-----------------|
| **Coordinator** | Task decomposition, assignment, result aggregation |
| **Worker 1..N** | Execute assigned subtasks, report results |
| **Observer** (optional) | Monitor progress, flag issues, no direct task execution |

### 4.3 Role Assignment

Roles are assigned by the human user or by the Coordinator. Example:

```
Human: "@Coordinator 分析这三个竞品"
Coordinator → Worker1: "你负责竞品A" [CONTINUE]
Coordinator → Worker2: "你负责竞品B" [CONTINUE]
Coordinator → Worker3: "你负责竞品C" [CONTINUE]
Worker1 → Coordinator: "竞品A分析完成：..." [DONE]
Worker2 → Coordinator: "竞品B分析完成：..." [DONE]
Worker3 → Coordinator: "竞品C分析完成：..." [DONE]
Coordinator → Human: "三个竞品分析汇总：..." [INFO]
```

---

## 5. Addressing

### 5.1 Explicit @Mention

When communicating with another agent, **always** use the platform's @mention mechanism:

```
@AgentB 请帮我检查这段代码 [CONTINUE]
```

Without @mention, the target agent may not receive the message (platform-dependent).

### 5.2 Broadcast vs. Targeted

| Intent | Format |
|--------|--------|
| Target specific agent | `@AgentB message [CONTINUE]` |
| Broadcast to all | `@all 状态更新：任务完成 [INFO]` |

---

## 6. Platform Adaptation

The protocol is **platform-agnostic**. Signal tags work the same way regardless of whether agents communicate via:

- Feishu / Lark
- Telegram
- Discord
- Slack
- Any text-based messaging platform

Platform-specific concerns (webhook behavior, @mention syntax, message formatting) are handled in the platform implementation layer, not in the protocol.

---

## 7. Compliance

An agent is **A2A-compliant** if it:

1. ✅ Attaches exactly one signal tag to every inter-agent message
2. ✅ Never replies to `[DONE]` or `[INFO]` messages
3. ✅ Performs a self-check before sending (is there new information?)
4. ✅ Yields immediately when a human user sends a message
5. ✅ Respects turn limits (≤5 routine, ≤10 complex)
