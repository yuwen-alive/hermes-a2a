# 最简双 Bot 协作示例

> 从零搭建两个 Hermes Agent 在飞书群中协作对话

## 场景描述

两个 Hermes Agent（Bot A 和 Bot B）在同一个飞书群中，Bot A 作为协调人分配任务，Bot B 作为执行者完成任务。

## 前置条件

1. 两台机器（或同一台机器的两个环境）
2. 两个独立的飞书自建应用
3. 一个飞书群，两个 bot 都已加入

## 第一步：创建飞书应用

### Bot A（协调人）
1. 访问 https://open.feishu.cn/
2. 创建企业自建应用
3. 开启「机器人」能力
4. 记录 `App ID` 和 `App Secret`
5. 在「权限管理」中开通：
   - `im:message.group_at_msg.include_bot:readonly`
   - `im:message`

### Bot B（执行者）
重复上述步骤，创建第二个独立应用。

## 第二步：配置 Hermes 实例

### Bot A 的 .env
```bash
# 飞书配置
FEISHU_APP_ID=cli_xxxaaa
FEISHU_APP_SECRET=xxx

# 允许 bot 间通信
FEISHU_ALLOW_BOTS=mentions
```

### Bot B 的 .env
```bash
# 飞书配置
FEISHU_APP_ID=cli_yyybbb
FEISHU_APP_SECRET=yyy

# 允许 bot 间通信
FEISHU_ALLOW_BOTS=mentions
```

## 第三步：配置协作规则

### Bot A 的系统提示词（追加）

```
你是 Bot A，担任协调人角色。
你的协作伙伴是 Bot B（open_id: ou_xxx）。

协作规则：
1. 分配任务时 @ Bot B
2. 每条消息末尾加信号标记
3. 收到 [DONE] 不回复
4. 任务完成后 @ 用户汇报
```

### Bot B 的系统提示词（追加）

```
你是 Bot B，担任执行者角色。
你的协调人是 Bot A（open_id: ou_yyy）。

协作规则：
1. 接受 Bot A 分配的任务
2. 每条消息末尾加信号标记
3. 完成任务后发送 [DONE]
4. 遇到阻塞发送 [NEEDS_HUMAN]
```

## 第四步：测试

### 用户发起任务
```
用户：@BotA 帮我查一下今天的天气，然后让 Bot B 整理成表格
```

### Bot A 分配任务
```
Bot A：@BotB 请查一下今天北京的天气，整理成表格格式。

[CONTINUE]
```

### Bot B 执行并汇报
```
Bot B：@BotA 任务完成。北京今天天气：晴，25°C，湿度 40%，风力 2 级。
已整理成表格格式。

[DONE]
```

### Bot A 汇总汇报
```
Bot A：@用户 任务完成。今天北京天气：晴，25°C，已整理成表格。

[DONE]
```

## 关键验证点

1. ✅ Bot B 收到了 Bot A 的 @mention 消息
2. ✅ 消息中有蓝色高亮的 @ 标记
3. ✅ Bot B 发送 `[DONE]` 后 Bot A 没有继续回复
4. ✅ 最终结果汇报给了用户

## 常见问题

**Q: Bot B 收不到消息？**
A: 检查 `.env` 中 `FEISHU_ALLOW_BOTS=mentions` 是否设置，gateway 是否重启。

**Q: @ 没有蓝色高亮？**
A: 必须用 `<at user_id="...">` 标签格式，纯文本 `@` 不生效。

**Q: 两个 bot 无限对话？**
A: 确保每条消息都有信号标记，收到 `[DONE]` 不回复。
