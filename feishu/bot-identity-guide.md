# Bot 身份识别机制详解

> 理解 Hermes 如何在飞书中识别"自己"和"别人"。

## 为什么需要身份识别？

当群里有多个 Bot 时，每个 Bot 必须能区分：
- **自己发的消息** → 忽略（防止自回环）
- **人类用户的消息** → 正常处理
- **其他 Bot 的消息** → 根据 `allow_bots` 策略决定

## 身份标识符

| 标识符 | 环境变量 | 说明 |
|--------|---------|------|
| Open ID | `FEISHU_BOT_OPEN_ID` | Bot 在飞书中的唯一标识（推荐） |
| User ID | `FEISHU_BOT_USER_ID` | 可选，仅当应用使用 `sender_id_type=user_id` 时需要 |
| App ID | `FEISHU_APP_ID` | 应用级别标识（飞书 SDK 自动使用） |

## 自回环检测流程

```
收到消息 → sender.open_id = ?
                ↓
    ┌─── 等于 FEISHU_BOT_OPEN_ID？ ───┐
    │                                  │
    是                                 否
    ↓                                  ↓
  丢弃（self_echo）              继续处理
                                      ↓
                          sender.sender_type = "app"？
                                      ↓
                           是              否
                            ↓              ↓
                    allow_bots 策略    正常人类消息
```

## 启动时自动发现

如果未手动设置 `FEISHU_BOT_OPEN_ID`，Hermes 会在启动时：

1. 调用 `POST /open-apis/auth/v3/tenant_access_token/internal` 获取 token
2. 调用 `GET /open-apis/bot/v3/info` 获取 Bot 信息
3. 提取 `open_id` 字段作为自身标识

> ⚠️ **推荐手动设置**：自动发现在高并发启动时可能有竞态条件。

## 多 Bot 场景下的 ID 关系

```
飞书应用 A                    飞书应用 B
├── App ID: cli_aaa           ├── App ID: cli_bbb
├── App Secret: xxx           ├── App Secret: yyy
└── Bot Open ID: ou_aaa       └── Bot Open ID: ou_bbb
         ↑                            ↑
    Hermes 实例 A                Hermes 实例 B
    FEISHU_BOT_OPEN_ID=         FEISHU_BOT_OPEN_ID=
    ou_aaa                       ou_bbb
```

两个实例在同一个飞书群里，通过 `open_id` 互相区分。

---

*最后更新：2026-06-20*
