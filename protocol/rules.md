# A2A Collaboration Rules

## Rule 1: Mandatory Signal Tags

Every bot-to-bot message MUST end with exactly one signal tag:
`[CONTINUE]` / `[DONE]` / `[NEEDS_HUMAN]` / `[INFO]`

See [spec.md](spec.md) for signal definitions.

---

## Rule 2: Hard Stop on [DONE] and [INFO]

Receiving `[DONE]` or `[INFO]` → **MUST NOT reply**.

No exceptions. Not even "收到", "OK", "understood", or "confirmed".

---

## Rule 3: Self-Check Before Sending

Before composing a reply, verify:

1. **Is there substantive new content?** (new info, decision, data, question)
2. If the reply would only be "收到" / "OK" / "understood" / "confirmed" → **do not send**
3. If you already said something very similar in the last 2 messages → **do not send**

**Test**: Could this message be replaced with `[INFO]` without losing information? If yes, use `[INFO]` instead or don't send at all.

---

## Rule 4: User Messages = Highest Priority

- Human user sends a message → **immediately drop any queued bot reply**
- Bot conversation pauses until human intent is resolved
- Human can interrupt at any point; bots resume only after human task is handled

---

## Rule 5: Turn Limits

| Context | Max Turns |
|---------|----------|
| Routine task | ≤ 5 |
| Complex task (debugging, research, multi-step) | ≤ 10 |
| Approaching limit | Last message MUST use `[DONE]` |

Turn counter resets when a human user sends a new message.

**What counts as a turn**: One bot-to-bot message exchange (A→B or B→A). `[INFO]` messages do NOT count.

---

## Rule 6: Anti Ping-Pong Detection

If the last 2 bot messages are semantically near-identical (both just "收到", both asking the same question), the sender MUST:

1. Stop repeating
2. Switch to `[DONE]` or `[NEEDS_HUMAN]`

---

## Rule 7: Error Recovery

If a bot reply fails (timeout, API error, delivery failure):

- Retry max **2 times** with exponential backoff
- After 2 failures → `[NEEDS_HUMAN]` and stop
- The other bot MUST NOT keep sending if it's getting no response

---

## Rule 8: [INFO] Doesn't Count

`[INFO]` messages are fire-and-forget status updates:

- Do NOT trigger replies
- Do NOT count toward turn limits
- Use for progress bars, log streaming, intermediate status

---

## Rule 9: Must @mention Other Agents

When sending a message intended for another agent, **always** use the platform's @mention mechanism.

Without @mention, the target agent may not receive the message.

**Correct**: `@AgentB 分析完了，根因是...`
**Wrong**: `AgentB，分析完了，根因是...` ← target may not receive

---

## Rule 10: Role Respect

- **Leader/Coordinator** assigns tasks and aggregates results
- **Workers/Executors** execute tasks and report back
- Workers should NOT reassign tasks to other workers (go through Coordinator)
- Any agent can escalate to human via `[NEEDS_HUMAN]`

---

## Rule 11: Multi-Agent Turn Coordination (3+ agents)

When 3+ agents are active:

1. **Coordinator speaks first** — assigns tasks to workers
2. **Workers reply independently** — can reply in any order
3. **Coordinator aggregates** — waits for all `[DONE]` signals, then summarizes
4. **No circular dependencies** — A should not wait on B which waits on A

Turn limits apply per-agent: no single agent should exceed 5 turns (10 for complex).

---

## System Prompt Template

Inject this into each agent's system prompt:

```
You are communicating with another AI agent in this group chat.
You MUST follow the A2A Anti-Loop Protocol:

1. End every bot-to-bot message with: [CONTINUE] / [DONE] / [NEEDS_HUMAN] / [INFO]
2. If you receive [DONE] or [INFO], do NOT reply.
3. Before replying, check: does my reply contain NEW substantive information?
   If not, do not send it. "收到"/"OK" is never a valid reply.
4. User messages always take priority over bot conversations.
5. Max 5 bot-to-bot turns per task (10 for complex tasks).
   Approaching the limit? Use [DONE] to wrap up.
6. Always @mention the target agent when sending a message.
```
