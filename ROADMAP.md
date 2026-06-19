# Roadmap

> hermes-a2a 项目的版本演进计划。

---

## V1.0 -- 飞书双 Bot 协作 (当前)

**目标**: 在飞书平台上实现两个 Hermes Agent 的安全协作通信。

### 已完成

- [x] A2A 通信协议规范 (protocol/spec.md)
- [x] 协作规则与信号标签 (protocol/rules.md)
- [x] 防死循环四层机制 (protocol/anti-loop.md)
- [x] 飞书接入配置指南 (feishu/setup-guide.md)
- [x] Bot 身份识别机制 (feishu/bot-identity-guide.md)
- [x] 话题与回复控制 (feishu/thread-and-reply-guide.md)
- [x] 多 Agent 扩展架构 (docs/architecture.md)
- [x] 踩坑记录与 FAQ (docs/troubleshooting.md)
- [x] 配置模板 (.env.example, config-bot-a/b.yaml)
- [x] 协作行为准则 (docs/collaboration-guide.md)
- [x] Skill 包 (skills/hermes-a2a/SKILL.md)

### 待完成

- [ ] 实战案例 (examples/)
  - [ ] 双 Bot 基础协作教程
  - [ ] Token Monitor 实战案例
  - [ ] 信箱系统案例
- [ ] README 最终版 (合并双方内容)
- [ ] GitHub 仓库创建与上传

---

## V1.1 -- 协作信号增强

**时间**: V1 发布后 2 周

- [ ] `[CONTINUE]` / `[DONE]` / `[NEEDS_HUMAN]` / `[INFO]` 信号的系统提示词自动注入
- [ ] Turn 计数器的自动追踪 (不需要 Bot 手动计数)
- [ ] 协作会话的状态机可视化

---

## V2.0 -- 多 Agent 协作 (3+ Bots)

**时间**: V1 发布后 3 周

- [ ] Agent 注册表 (自动发现 + 心跳)
- [ ] Leader 选举机制 (多 Leader 场景)
- [ ] 任务队列与分发器
- [ ] 能力发现 (Leader 根据能力标签选择 Executor)
- [ ] reply_to 话题控制 (飞书群聊不自动创建话题)
- [ ] 跨 Agent 的上下文传递

---

## V3.0 -- 多平台支持

**时间**: V2 发布后 4 周

- [ ] 平台适配器接口 (PlatformAdapter ABC)
- [ ] Discord 适配器
- [ ] Telegram 适配器
- [ ] Slack 适配器
- [ ] 企业微信适配器
- [ ] 跨平台消息格式转换
- [ ] 跨平台身份映射

---

## V4.0 -- 智能协作

**时间**: V3 发布后 4 周

- [ ] 基于 LLM 的任务自动分解
- [ ] Agent 自主协商 (无需 Leader 手动分配)
- [ ] 协作历史学习 (从过去的协作中优化策略)
- [ ] 自适应 Turn 限制 (根据任务复杂度动态调整)

---

## V5.0 -- 生态集成

**时间**: V4 发布后 4 周

- [ ] MCP Server 模式 (让外部工具通过 MCP 调用 Agent 协作)
- [ ] Kanban 集成 (多 Agent 看板协作)
- [ ] Web Dashboard (实时监控 Agent 协作状态)
- [ ] 插件市场 (第三方贡献 Agent 能力包)

---

*最后更新: 2026-06-20*
