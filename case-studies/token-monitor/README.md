# Token Monitor 实战案例

> 两个 Bot 协作搭建 Token 用量监控系统（已脱敏）

## 背景

用户需要监控两个 Hermes 实例的 Token 使用情况，并在一个统一的 Dashboard 上展示。

## 角色分工

| Bot | 角色 | 负责 |
|-----|------|------|
| Bot A（协调人） | Windows 端 | 数据采集、定时上报 |
| Bot B（执行者） | 服务器端 | API 搭建、Dashboard 开发 |

## 协作过程

### 第 1 轮：任务分配

```
Bot A → Bot B：

## 任务：搭建 Token 用量监控

**目标**：搭建一个 Dashboard，展示两个 Hermes 实例的 Token 使用量

**你需要做**：
1. 搭建 API 接收端（POST /api/token-report）
2. 存储数据到数据库
3. 开发 Dashboard 页面（按日/周汇总展示）

**我来做**：
1. 采集 Windows 端 state.db 的 Token 数据
2. 编写定时上报脚本
3. 配置 Windows 计划任务

[CONTINUE]
```

### 第 2 轮：进度同步

```
Bot B → Bot A：

📊 进度更新：Token Monitor

- ✅ 已完成：API 接收端，POST /api/token-report
- 🔄 进行中：Dashboard 页面
- ⏳ 待开始：数据可视化

预计明天完成 Dashboard。

[INFO]
```

### 第 3 轮：Bot A 也完成采集端

```
Bot A → Bot B：

📊 进度更新：Token 数据采集

- ✅ 已完成：state.db 数据查询脚本
- ✅ 已完成：PowerShell 上报脚本
- ✅ 已完成：Windows 计划任务配置（每小时执行）

采集端就绪，等你 Dashboard 完成后一起测试。

[INFO]
```

### 第 4 轮：任务完成

```
Bot B → Bot A：

✅ 任务完成：Token Monitor

**产出物**：
- API 端点：POST http://xxx:8199/api/token-report
- Dashboard：http://xxx:8199
- 支持：按日汇总、双 Bot 合并/单独视图

[DONE]
```

### 第 5 轮：协调人汇总

```
Bot A → 用户：

## 📋 任务完成汇报

**目标**：搭建 Token 用量监控系统

**完成情况**：
- ✅ Windows 端数据采集（state.db → PowerShell 脚本 → 计划任务）
- ✅ 服务器端 API（POST /api/token-report）
- ✅ Dashboard 页面（按日汇总、双 Bot 视图）

**分工**：
- Bot A（Windows）：采集脚本 + 定时上报
- Bot B（服务器）：API + Dashboard

**产出物**：
- Dashboard：http://xxx:8199
- 上报脚本：windows-report.ps1
- 计划任务：Hermes_Token_Report

**遗留**：
- 无

[DONE]
```

## 技术要点（脱敏）

### 数据采集端
```
数据源：state.db → sessions 表
字段：input_tokens, output_tokens, cache_read_tokens, reasoning_tokens
上报：PowerShell 脚本 → HTTP POST JSON
频率：每小时
```

### API 端
```
端点：POST /api/token-report
格式：{"source": "windows", "data": [{"day": "2026-06-18", ...}]}
存储：SQLite / PostgreSQL
展示：Flask + Chart.js
```

## 经验总结

1. **任务拆分清晰** — 采集端和展示端独立，互不阻塞
2. **进度同步及时** — 用 `[INFO]` 信号同步进展
3. **最终汇报完整** — 包含目标、分工、产出物、遗留事项
4. **数据源选择** — state.db 比 sessions.json 更准确（自托管模型 token 不为 0）
