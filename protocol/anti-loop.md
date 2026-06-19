# Anti-Loop Mechanism

## The Problem

When two AI agents can see each other's messages in a shared chat, they risk entering an **infinite ping-pong loop**:

```
Agent A → Agent B: "收到" 
Agent B → Agent A: "收到"
Agent A → Agent B: "收到"
Agent B → Agent A: "收到"
... (forever, burning tokens)
```

Or worse, substantive but redundant exchanges:

```
Agent A → Agent B: "分析结果是X，你觉得呢？"
Agent B → Agent A: "我也觉得是X，你怎么看？"
Agent A → Agent B: "同意，就是X"
Agent B → Agent A: "确认，就是X"
... (converged but never stopped)
```

---

## Defense Layers

The A2A protocol implements **4 layers** of defense against infinite loops:

### Layer 1: Signal Tags (Primary)

Every message carries a signal that explicitly declares reply expectation:

| Signal | Reply Allowed? |
|--------|---------------|
| `[CONTINUE]` | ✅ Yes |
| `[DONE]` | ❌ No |
| `[NEEDS_HUMAN]` | ❌ No |
| `[INFO]` | ❌ No |

This is the **primary** mechanism. If both agents follow the protocol, loops are impossible — a conversation always terminates with `[DONE]`, `[NEEDS_HUMAN]`, or a turn limit.

### Layer 2: Content Self-Check

Before sending any reply, the agent must verify:

```
1. Does this reply contain NEW substantive information?
   - New data, new decision, new question = YES → send
   - "收到", "OK", "agree" = NO → don't send

2. Did I say something very similar in the last 2 messages?
   - Yes → don't send
   - No → proceed
```

This catches cases where both agents are technically using `[CONTINUE]` but have no new information to share.

### Layer 3: Turn Limits (Hard Ceiling)

Even if layers 1 and 2 fail, turn limits provide a hard ceiling:

| Context | Max Turns | Action at Limit |
|---------|----------|-----------------|
| Routine | 5 | Force `[DONE]` |
| Complex | 10 | Force `[DONE]` or `[NEEDS_HUMAN]` |

The turn counter counts bot-to-bot messages only. `[INFO]` messages and human messages are excluded.

**Reset condition**: A human user sends a new message → turn counter resets to 0.

### Layer 4: User Interrupt Priority

A human message **always** takes priority:

1. Human sends a message → all bot conversation queues are flushed
2. Bot conversation is paused until human intent is resolved
3. After human task is handled, bots may resume (with fresh turn counter)

This ensures that even if a loop somehow starts, the human can stop it instantly.

---

## Ping-Pong Detection

### Definition

A **ping-pong** pattern is detected when the last N bot messages are semantically near-identical:

```
Message 1 (Agent A): "收到" 
Message 2 (Agent B): "收到"
→ Ping-pong detected!
```

### Detection Logic

```
if last_2_messages_are_similar():
    STOP → use [DONE] or [NEEDS_HUMAN]
```

### What "Similar" Means

- Exact text match (e.g., both say "收到")
- Same question asked twice (e.g., both ask "你觉得呢？")
- Same conclusion repeated (e.g., both say "方案是X")

### What "Similar" Does NOT Mean

- Different aspects of the same topic (e.g., Agent A discusses cost, Agent B discusses timeline)
- Follow-up questions (e.g., "你说的X具体是什么？" after "方案是X")

---

## Error Recovery

### Scenario: Agent Reply Fails

```
Agent A → Agent B: "请分析" [CONTINUE]
Agent B → Agent A: (API timeout, message not delivered)
Agent A → Agent B: (retries)
Agent B → Agent A: (API timeout again)
→ After 2 failures: Agent A sends [NEEDS_HUMAN]
```

### Rules

1. **Max 2 retries** with exponential backoff (1s, 2s)
2. After 2 failures → `[NEEDS_HUMAN]` and stop
3. The other agent MUST NOT keep sending if it's getting no response
4. Do NOT retry on `[DONE]` or `[INFO]` — those don't expect replies

---

## Real-World Example: What Happens Without the Protocol

> From production logs — two Hermes bots in the same Feishu group:

```
[16:55] 小牛 → 小马: "我来分析这段代码" (no signal tag)
[16:55] 小马 → 小牛: "好的我也看看" (no signal tag)
[16:55] 小牛 → 小马: "分析结果..." (no signal tag)  
[16:55] 小马 → 小牛: "我的分析..." (no signal tag)
[16:55] API 429: quota exhausted (both bots hit rate limit)
[16:55] Both bots fail simultaneously
```

**Root cause**: No signal tags → no way to stop the conversation → infinite loop until external resource limit kicks in.

**With A2A protocol**:
```
[16:55] 小牛 → 小马: "我来分析这段代码" [CONTINUE]
[16:55] 小马 → 小牛: (sees [CONTINUE], waits for result)
[16:55] 小牛 → 小马: "分析结果：..." [DONE]
[16:55] 小马 → 小牛: (sees [DONE], does NOT reply)
→ Conversation ends cleanly, no wasted tokens
```

---

## Monitoring & Debugging

### Turn Counter Logging

Agents should log their turn count for debugging:

```
[A2A-TURN] chat_id=oc_xxx agent=AgentA turn=3/5
```

### Signal Tag Auditing

For complex multi-agent sessions, log all signals:

```
[A2A-SIGNAL] 16:55:01 AgentA → AgentB [CONTINUE]
[A2A-SIGNAL] 16:55:03 AgentB → AgentA [DONE]
[A2A-SIGNAL] 16:55:03 AgentB: conversation ended (reason: task_complete)
```
