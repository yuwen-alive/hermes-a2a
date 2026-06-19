---
name: hermes-a2a
description: "Hermes Agent-to-Agent 协作框架 — 让多个 Hermes 实例在飞书群中协作对话"
version: 1.0.0
tags: [a2a, feishu, bot-to-bot, collaboration, multi-agent]
---

# Hermes A2A 协作 Skill

## 触发条件

当需要与其他 Hermes Agent 实例协作通信时自动生效。

## 核心协议

### @mention 格式

**必须使用飞书 `<at>` 标签**：
```
<at user_id="{open_id}">{Bot名称}</at> 消息内容

[信号标记]
```

### 信号标记（每条消息必须携带）

| 信号 | 含义 | 对方应 |
|------|------|--------|
| `[CONTINUE]` | 需要回复 | 回复 |
| `[DONE]` | 任务完成 | 不回复 |
| `[NEEDS_HUMAN]` | 需要人类 | 通知用户 |
| `[INFO]` | 纯通知 | 不回复 |

### 防死循环规则

1. **信号收敛**：收到 `[DONE]`/`[INFO]` → 不回复
2. **回复自检**：发送前问「提供了新信息吗？」→ 否则不发
3. **内容去重**：与上一条相似度 > 80% → 不发
4. **用户优先**：用户消息 → 立即打断 bot 对话
5. **轮次限制**：常规 ≤5 轮，复杂 ≤10 轮

### 协作铁律

1. **每条消息必须 @ 对方**，发完检查蓝色标
2. **进展消息发群里**，不走私发
3. **多轮协作可创建话题**，一两轮说完的不创建

## 角色定义

### 主协调人（Coordinator）
- 任务分解与分配
- 进度跟踪
- 结果汇总
- 任务完成后 @ 用户汇报

### 执行者（Executor）
- 接受任务
- 独立执行
- 汇报进度和结果
- 阻塞时升级给人类

## @mention 验证（每次必做）

发送后检查：
1. 飞书消息中的 @ 是否有**蓝色高亮**
2. 没有蓝色标 → `<at>` 标签没生效 → 重发

## 详细参考

- 协议规范：`references/protocol.md`
- 防循环机制：`references/anti-loop.md`
- 消息模板：`templates/message-template.md`
- 汇报模板：`templates/report-template.md`
