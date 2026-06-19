# 信箱系统实战案例

> 两个 Bot 协作搭建跨实例消息信箱（已脱敏）

## 背景

两个 Hermes Agent 运行在不同机器上，需要一种机制让一个 Bot 可以给另一个 Bot "留言"，即使对方当前不在线。

## 角色分工

| Bot | 角色 | 负责 |
|-----|------|------|
| Bot A（协调人） | Windows 端 | 信箱客户端设计、发送/接收逻辑 |
| Bot B（执行者） | 服务器端 | 信箱服务端 API、消息队列 |

## 协作过程

### 第 1 轮：需求讨论

```
Bot A → Bot B：

## 任务：搭建 Bot 间信箱系统

**需求**：
1. Bot A 可以给 Bot B 发送"留言"
2. Bot B 上线后自动拉取未读留言
3. 支持消息状态（已读/未读）
4. 留言内容支持文本和文件引用

**分工建议**：
- 我做：客户端 SDK（send/read 操作）
- 你做：服务端 API + 存储

[CONTINUE]
```

### 第 2 轮：方案确认

```
Bot B → Bot A：

方案确认，补充几点：

**API 设计**：
- POST /api/inbox/send — 发送留言
- GET /api/inbox/{bot_id}/unread — 获取未读留言
- PUT /api/inbox/{msg_id}/read — 标记已读

**存储**：SQLite，轻量级够用

开始开发。

[CONTINUE]
```

### 第 3 轮：各自进展

```
Bot B → Bot A：

📊 进度：信箱服务端

- ✅ API 框架搭建
- ✅ POST /send 接口
- 🔄 GET /unread 接口
- ⏳ 标记已读接口

[INFO]
```

```
Bot A → Bot B：

📊 进度：信箱客户端

- ✅ send() 函数封装
- ✅ read() 函数封装
- 🔄 自动拉取定时任务
- ⏳ 与 Hermes Skill 集成

[INFO]
```

### 第 4 轮：联调

```
Bot A → Bot B：

联调发现一个问题：POST /send 的 content_type 字段你们返回的和我们发的不一致。
我们发 `text`，你们存成 `plain`。统一用 `text` 吧？

[CONTINUE]
```

```
Bot B → Bot A：

已修复，统一用 `text`。现在返回格式：
```json
{"ok": true, "msg_id": "xxx", "content_type": "text"}
```

[CONTINUE]
```

```
Bot A → Bot B：

联调通过 ✅ 信箱系统可用了。

[DONE]
```

### 第 5 轮：汇报

```
Bot A → 用户：

## 📋 信箱系统完成汇报

**目标**：搭建 Bot 间离线消息信箱

**完成情况**：
- ✅ 服务端 API（发送/接收/标记已读）
- ✅ 客户端 SDK（send/read 封装）
- ✅ 自动拉取定时任务
- ✅ 联调测试通过

**产出物**：
- 服务端：POST/GET/PUT /api/inbox/*
- 客户端：inbox_client.py
- Skill：hermes-inbox（可选安装）

**遇到的问题**：
- 字段命名不一致 → 统一为 text/plain

**遗留**：
- 文件附件支持（V2 实现）

[DONE]
```

## 技术要点（脱敏）

### API 设计
```
POST /api/inbox/send
  Body: {"from": "bot_a", "to": "bot_b", "content": "...", "content_type": "text"}
  Response: {"ok": true, "msg_id": "xxx"}

GET /api/inbox/{bot_id}/unread
  Response: {"messages": [{"msg_id", "from", "content", "created_at"}, ...]}

PUT /api/inbox/{msg_id}/read
  Response: {"ok": true}
```

### 客户端封装
```python
# 伪代码
def send_inbox(to_bot, content):
    return requests.post(f"{API_URL}/api/inbox/send", json={
        "from": MY_BOT_ID,
        "to": to_bot,
        "content": content,
        "content_type": "text"
    })

def read_unread():
    return requests.get(f"{API_URL}/api/inbox/{MY_BOT_ID}/unread").json()
```

## 经验总结

1. **联调是关键** — 各自开发完成后必须联调，发现字段不一致等问题
2. **API 设计先行** — 先确认接口格式再开发，减少返工
3. **轻量级够用** — SQLite 满足初期需求，不需要 Redis/Kafka
4. **渐进式功能** — V1 只支持文本，V2 再加文件附件
