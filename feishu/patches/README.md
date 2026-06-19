# 飞书平台补丁说明

> 本文档记录 hermes-a2a 项目需要的飞书相关改动及其上游状态。

## 当前状态：无需额外补丁 ✅

截至 **Hermes Agent v0.15.1**，Bot-to-Bot 通信所需的核心功能已全部内置：

| 功能 | 状态 | 配置方式 |
|------|------|---------|
| 接受其他 Bot 消息 | ✅ 内置 | `FEISHU_ALLOW_BOTS=mentions` |
| Bot 自身身份识别 | ✅ 内置 | `FEISHU_BOT_OPEN_ID` |
| 自回环保护 | ✅ 内置 | 自动检测 `sender.open_id == bot_open_id` |
| 消息 Thread 路由 | ✅ 内置 | 自动提取 `root_id` / `thread_id` |
| 群聊 @mention 过滤 | ✅ 内置 | `require_mention: true` |

## 历史补丁（已合入上游）

### 1. FEISHU_ALLOW_BOTS 环境变量

- **PR**: [#758](https://github.com/NousResearch/hermes-agent/pull/758)
- **版本**: v0.2.0 起可用
- **说明**: 添加了 `none` / `mentions` / `all` 三种模式控制 Bot 消息过滤

### 2. Discord DISCORD_ALLOW_BOTS（参考实现）

- **PR**: [#758](https://github.com/NousResearch/hermes-agent/pull/758)
- **说明**: 飞书的 `FEISHU_ALLOW_BOTS` 设计参考了 Discord 的同名功能

## 未来可能的补丁

| 功能 | 状态 | 说明 |
|------|------|------|
| reply_to 话题控制 | 📋 规划中 | 控制 Bot 回复是在原话题还是创建新话题 |
| Bot 间消息去重增强 | 📋 规划中 | 多实例场景下的跨进程去重 |
| 协作信号标签支持 | 📋 规划中 | `[CONTINUE]`/`[DONE]` 等协议标签的原生支持 |

## 补丁维护策略

1. **优先推上游**：所有通用功能优先向 Hermes 上游提 PR
2. **文档化**：每个补丁记录改了哪些文件、哪几行、为什么
3. **定期检查**：每个 Hermes 版本发布后检查补丁是否已被内置替代
4. **降级方案**：如果上游不收，评估长期 fork 的维护成本

---

*最后更新：2026-06-20*
