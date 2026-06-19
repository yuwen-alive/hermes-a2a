# 多 Agent 扩展架构设计

> 从双 Bot 到 N Bot 的架构演进路线。

## 一、架构概览

### V1: 双 Bot 直连 (当前)

```
+---------+    飞书群     +---------+
|  小马 🐴 | <----------> |  小牛 🐮 |
|  Leader  |   @mention   | Executor |
+---------+              +---------+
     ^                        ^
     |  HERMES_HOME=A         |  HERMES_HOME=B
     |  App ID: cli_aaa       |  App ID: cli_bbb
     +------------------------+
            同一飞书群
```

**特点**: 点对点通信，无中间层，简单可靠。

### V2: 多 Bot 星型拓扑

```
                    +----------+
                    | 协调中心  |
                    | (Router) |
                    +----+-----+
              +---------+---------+
              v         v         v
         +---------+ +---------+ +---------+
         | Agent A | | Agent B | | Agent C |
         |  Leader | |Executor | |Executor |
         +---------+ +---------+ +---------+
```

**特点**: 一个 Leader 协调多个 Executor，任务分配清晰。

### V3: 多平台 Mesh 拓扑

```
         飞书群                    Telegram
    +----+----+               +----+----+
    | Agent A |               | Agent D |
    | Agent B |               | Agent E |
    +----+----+               +----+----+
         |        +--------+       |
         +------->| Router |<------+
                  | Hub    |
    +------------>|        |<------------+
    |             +--------+             |
    |          Discord                   |
    |     +----+----+                   |
    |     | Agent F |                   |
    |     | Agent G |                   |
    |     +---------+                   |
    |                               Slack
    |                          +----+----+
    |                          | Agent H |
    +--------------------------| Agent I |
                               +---------+
```

---

## 二、核心组件设计

### 2.1 Agent 注册表 (Agent Registry)

每个 Agent 启动时向注册表注册自身信息:

```yaml
# 注册表结构 (可存储在文件/数据库/内存中)
agents:
  - id: "agent-xiaoma"
    name: "小马"
    role: leader
    platform: feishu
    capabilities: ["task-planning", "reporting", "coordination"]
    status: online
    last_heartbeat: "2026-06-20T12:00:00Z"
    peer_ids: ["agent-xiaoniu"]

  - id: "agent-xiaoniu"
    name: "小牛"
    role: executor
    platform: feishu
    capabilities: ["server-ops", "data-query", "code-execution"]
    status: online
    last_heartbeat: "2026-06-20T12:00:00Z"
    peer_ids: ["agent-xiaoma"]
```

### 2.2 消息路由 (Message Router)

消息路由的核心逻辑:

1. 解析 @mention -> 确定目标 Agent
2. 检查目标 Agent 状态 (是否在线)
3. 检查防循环规则 (是否会导致死循环)
4. 转发消息到目标 Agent

### 2.3 角色定义

| 角色 | 职责 | 数量 | 典型能力 |
|------|------|------|---------|
| **Leader** | 任务分解、分配、汇总、汇报用户 | 1-2 | 规划、协调、格式化输出 |
| **Executor** | 执行具体子任务、反馈结果 | 1-N | 代码执行、数据查询、API 调用 |
| **Monitor** | 监控系统状态、异常告警 | 0-N | 日志分析、指标监控、告警推送 |
| **Specialist** | 特定领域专家 | 0-N | 翻译、搜索、文档处理 |

### 2.4 任务分发协议

```
用户 -> Leader: @小马 帮我分析服务器状态和今天的 API 用量
Leader -> Leader: 分解任务为子任务 A、B
Leader -> Executor1: @小牛 [TASK:A] 查服务器状态
Leader -> Executor2: @小花 [TASK:B] 查 API 用量
Executor1 -> Leader: [DONE] 服务器状态正常, CPU 45%
Executor2 -> Leader: [DONE] API 用量 120k tokens
Leader -> 用户: 汇总报告
```

---

## 三、扩展点设计

### 3.1 平台适配层

```python
class PlatformAdapter(ABC):
    """平台适配器接口。"""

    @abstractmethod
    async def send_message(self, chat_id: str, text: str, reply_to: str = None):
        """发送消息。"""

    @abstractmethod
    async def extract_mention(self, message) -> Optional[str]:
        """提取 @mention 目标。"""

    @abstractmethod
    def get_bot_id(self) -> str:
        """获取自身 Bot ID。"""
```

已实现:
- `FeishuAdapter` -- 飞书

待实现:
- `TelegramAdapter` -- Telegram
- `DiscordAdapter` -- Discord
- `SlackAdapter` -- Slack
- `WeChatWorkAdapter` -- 企业微信

### 3.2 能力发现机制

```yaml
# 每个 Agent 声明自己的能力
capabilities:
  - name: server-ops
    description: "SSH to server and execute commands"
    parameters:
      - host: string
      - command: string
    examples:
      - "查询服务器磁盘使用情况"
      - "重启 nginx 服务"
```

Leader 可以根据能力标签自动选择合适的 Executor。

### 3.3 共享状态层

```
+-------------------------------------+
|          Shared State Layer          |
+---------+-----------+---------------+
| 任务队列 | 结果缓存  |  Agent 心跳   |
| task:// | cache://  | heartbeat://  |
+---------+-----------+---------------+
     ^          ^            ^
  Agent A    Agent B     Agent C
```

---

## 四、防死循环在多 Agent 场景下的扩展

### 双 Agent -> N Agent 的变化

| 维度 | 双 Agent | N Agent |
|------|---------|---------|
| 通信路径 | 1 条 (A<->B) | N*(N-1) 条 |
| 循环风险 | 低 | 指数增长 |
| Turn 控制 | 简单 (A/B 交替) | 需要令牌环或协调者 |
| 资源争抢 | 2 方争抢 | N 方争抢 |

### 解决方案: 协调者模式

```
所有 Agent 间的通信都经过协调者 (Leader):
  Agent A -> Leader -> Agent B
  Agent B -> Leader -> Agent A
  Agent C -> Leader -> Agent A

好处:
1. Leader 统一管理 Turn 分配
2. Leader 可以检测全局循环
3. Leader 可以做资源调度
```

---

## 五、部署方案

### 5.1 双实例 systemd 配置

```ini
# /etc/systemd/system/hermes-bot-a.service
[Unit]
Description=Hermes Bot A (小马)
After=network.target

[Service]
Type=simple
User=hermes
Environment=HERMES_HOME=/opt/hermes-bot-a
ExecStart=/opt/hermes/.venv/bin/hermes gateway start
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```

```ini
# /etc/systemd/system/hermes-bot-b.service
[Unit]
Description=Hermes Bot B (小牛)
After=network.target

[Service]
Type=simple
User=hermes
Environment=HERMES_HOME=/opt/hermes-bot-b
ExecStart=/opt/hermes/.venv/bin/hermes gateway start
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```

### 5.2 进程管理对比

| 方案 | 优点 | 缺点 | 推荐场景 |
|------|------|------|---------|
| systemd | 系统原生、自动重启、日志集成 | 需要 root 或 sudo | 服务器部署 |
| supervisor | 配置简单、Web 管理界面 | 额外依赖 | 开发环境 |
| s6 | 轻量级、容器友好 | 学习曲线 | 容器化部署 |
| Docker Compose | 隔离性好、一键部署 | 网络配置复杂 | 生产环境 |

### 5.3 Docker Compose 示例

```yaml
version: '3.8'
services:
  bot-a:
    image: hermes-agent:latest
    environment:
      - HERMES_HOME=/app/data
      - FEISHU_APP_ID=cli_aaa
      - FEISHU_APP_SECRET=<secret_a>
      - FEISHU_BOT_OPEN_ID=ou_aaa
      - FEISHU_ALLOW_BOTS=mentions
    volumes:
      - ./data-bot-a:/app/data
    restart: always

  bot-b:
    image: hermes-agent:latest
    environment:
      - HERMES_HOME=/app/data
      - FEISHU_APP_ID=cli_bbb
      - FEISHU_APP_SECRET=<secret_b>
      - FEISHU_BOT_OPEN_ID=ou_bbb
      - FEISHU_ALLOW_BOTS=mentions
    volumes:
      - ./data-bot-b:/app/data
    restart: always
```

---

*最后更新: 2026-06-20*
