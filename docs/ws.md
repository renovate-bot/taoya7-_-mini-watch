# WebSocket 报文协议

手机端与服务端通过单一 WebSocket 长连接交换 JSON 消息。本文档列出报文格式。

## 1. 连接

| 项 | 值 |
| --- | --- |
| 测试环境 | `wss://test.t8s.ink/api/ws` |
| 本地开发 | `ws://localhost:3001/api/ws` |
| 子协议 | 无 |
| 鉴权 | 暂未启用（后续走 query `?token=` 或 Cookie） |

客户端配置见 `iapp/lib/config.dart`（`AppConfig.wsUrl`）。

## 2. 通用约定

- 所有消息为 **UTF-8 编码的 JSON 字符串**
- 顶层字段：
  - `type` *(string, 必填)* — 报文类型，决定 payload 结构
  - `ts` *(number, 必填)* — 发送时间戳，毫秒（`Date.now()`）
  - 其余字段视 `type` 而定
- 字段命名 **snake_case**
- 服务端会忽略无法解析 / 缺少 `type` 的消息，回复 `{"type":"error","error":"invalid_json"}`

## 3. 客户端 → 服务端

### 3.1 `ping` — 心跳

```json
{ "type": "ping", "ts": 1735000000000 }
```

- 客户端每 25 秒发一次；服务端回复 `pong`
- 超过 60s 未收到 `pong` 视为连接失效，客户端 3s 后重连

### 3.2 `page_change` — 页面切换

```json
{
  "type": "page_change",
  "page": {
    "id": "home",
    "name": "系统报告",
    "path": "/",
    "index": 0
  },
  "ts": 1735000000000
}
```

- App 启动首帧 + 用户左右滑切页时上报
- `page.id` 来源于 `iapp/lib/routes/app_routes.dart` 的注册表，stable 标识符
- `page.name` 中文显示名
- `page.path` Navigator 路由名
- `page.index` PageView 中的位置（仅滑动页有，modal 页可不带）

#### 当前已注册 page

| id | name | path | 类型 |
| --- | --- | --- | --- |
| `home` | 系统报告 | `/` | pager |
| `traffic_light` | 红绿灯 | `/traffic-light` | pager |
| `settings` | 设置 | `/settings` | modal |

### 3.3 `getBaseInfo` — 拉取主机基础信息

```json
{ "type": "getBaseInfo", "ts": 1735000000000 }
```

- 客户端连接成功后服务端已主动下发一次 `base_info`，此报文用于**主动重新拉取**（例如断线重连后）
- 服务端响应一条 `base_info`（见 4.5）

### 3.4 `goto_page` — 远程跳转手机页面

```json
{
  "type": "goto_page",
  "page_id": "traffic_light",
  "ts": 1735000000000
}
```

- PC 控制台下拉选目标页时发送
- 服务端不缓存（瞬时指令），直接广播给所有 peer
- 手机端 `HomePager` 收到后调用 `PageController.animateToPage` 跳转
- 跳转完成后手机会通过 `page_change` 回报实际页面状态

### 3.5 `brightness_change` — 屏幕亮度

```json
{
  "type": "brightness_change",
  "value": 70,
  "ts": 1735000000000
}
```

- PC 控制台滑动滑块松手时发送
- 服务端缓存 `mobile:current_brightness` + 广播
- 手机端收到后调用 `ScreenBrightness.setApplicationScreenBrightness(value/100)`
- `value` 范围 1..100，整数（服务端会 clamp）

### 3.6 `theme_change` — 主题切换

```json
{
  "type": "theme_change",
  "theme_id": "ocean",
  "ts": 1735000000000
}
```

- PC 控制台点击主题色按钮时发送
- 服务端缓存 `mobile:current_theme` + 广播给所有 peer
- 手机端收到 `theme_change` 自动调用 `themeController.setById(...)` 切换并持久化
- `theme_id` 取值：`amber` / `ocean` / `forest` / `carbon` / `sakura`（详见 `iapp/lib/styles/themes.dart`）

## 4. 服务端 → 客户端

### 4.1 `hello` — 连接握手

```json
{ "type": "hello", "ts": 1735000000000 }
```

服务端在 `onOpen` 时主动下发；客户端收到代表 WebSocket 已就绪。

### 4.2 `pong` — 心跳回应

```json
{ "type": "pong", "ts": 1735000000000 }
```

回应客户端的 `ping`。

### 4.3 `online` — 在线人数

```json
{ "type": "online", "count": 2, "ts": 1735000000000 }
```

任意 peer 上下线触发，广播给所有 peer。
`count` 当前活跃连接总数。

### 4.4 `broadcast` — 广播

```json
{
  "type": "broadcast",
  "data": { /* 原始客户端消息体 */ },
  "ts": 1735000000000
}
```

服务端把某客户端发来的非控制消息（非 ping）广播给所有在线 peer。
`data` 是原始 JSON 解析结果。

### 4.5 `base_info` — 主机基础信息

```json
{
  "type": "base_info",
  "data": {
    "hostname": "Taoya",
    "platform": "darwin",
    "arch": "arm64",
    "osName": "macOS",
    "osVersion": "26.4",
    "model": "Mac15,9",
    "chip": "Apple M4 Pro",
    "cpuCores": 12,
    "memoryGB": 24,
    "bootTime": 1735000000000,
    "uptimeMs": 1080000000,
    "serverStartedAt": 1735200000000
  },
  "ts": 1735200000000
}
```

- 服务端启动时通过 Node `os` 模块 + macOS `sysctl` 采集，写入 `system:base_info`
- 每个新连接 `onOpen` 主动下发一次
- 客户端发 `getBaseInfo` 也可主动拉取
- `uptimeMs` 在响应时按 `Date.now() - bootTime` 实时重算，长跑不过期

### 4.6 `theme_change` — 主题下发

```json
{
  "type": "theme_change",
  "theme_id": "ocean",
  "ts": 1735000000000
}
```

- 服务端把 PC 端发起的 `theme_change` 广播给所有连接（包括手机）
- 新 peer 连接握手后，服务端会主动下发缓存中的最近一次主题，保证刷新后状态同步

### 4.7 `system_stats` — 系统状态快照

```json
{
  "type": "system_stats",
  "data": {
    "cpu":     { "usage": 32.9, "loadAvg": 4.2, "cores": 12, "tempC": 38 },
    "gpu":     { "usage": 30, "tempC": 53, "cores": 16, "model": "Apple M4 Pro" },
    "memory":  { "pressure": 53, "usedGB": 18.07, "totalGB": 24, "swapUsedGB": 6 },
    "temp":    { "cpu": 38, "gpu": 53, "disk": null, "ambient": null },
    "disk":    { "totalGB": 1830, "usedGB": 1020, "availGB": 810, "usedPct": 55.7 },
    "network": { "rxKBs": 124, "txKBs": 8, "iface": "en0" },
    "fan":     { "rpm": null, "load": null }
  },
  "ts": 1735000000000
}
```

- 服务端基于 `systeminformation` 包采集
- 第一个 peer 上线时启动 5 秒一次的定时广播；最后一个 peer 离线时自动停
- 新连接握手后立即推送一帧，避免首次等 5s
- 数值字段类型见 `iapp/lib/types/system_stats.dart` 与 `server/src/lib/system-stats.ts`，缺采集（macOS 无 sudo 拿不到 GPU/Temp/Fan）字段为 `null`

### 4.8 `stats_history` — 历史 stats（断线/重连恢复 chart）

```json
{
  "type": "stats_history",
  "data": [
    { "ts": 1735000000000, "cpu": 32.9, "gpu": 30, "mem": 53, "rx": 124, "tx": 8 },
    { "ts": 1735000005000, "cpu": 30.1, "gpu": 28, "mem": 52, "rx": 100, "tx": 6 }
  ],
  "ts": 1735000010000
}
```

- 持久化在 `data/stats-history.json`，重启后保留（节流 5s 写盘 1 次）
- 容量上限 240 点（5s × 240 = 20 分钟）
- 新连接 onOpen 时**总是**发一次，让 sparkline 立刻有数据可画
- 字段说明：`ts` 毫秒时间戳；`cpu/mem` 百分比；`gpu` 百分比或 `null`；`rx/tx` KB/s

### 4.9 `signal_light` — AI Agent 灯语

```json
{
  "type": "signal_light",
  "aggregate": "working",
  "signal": "PreToolUse",
  "session_id": "abc-123",
  "active_sessions": 2,
  "ts": 1735000000000
}
```

- 由 Claude Code hook 通过 `POST /api/signal-light/state` 触发，服务端聚合多会话后广播
- `aggregate` 全局聚合后的状态（4 种）：
  - `idle` → 绿灯常亮
  - `working` → 黄灯慢闪呼吸（thinking/working/tool_done）
  - `attention` → 黄灯快闪（attention/done）
  - `blocked` → 红灯快闪（blocked/permission/error）
- `signal` 本次触发的原始 signal 名（thinking/working/blocked/...）
- `active_sessions` 当前还活着的 session 数
- Flutter 端 `TrafficLightPage` 用此状态控制 3 盏灯

#### HTTP 端点

```bash
# POST 上报
curl -X POST http://localhost:3001/api/signal-light/state \
  -H 'Content-Type: application/json' \
  -d '{"signal":"working","session_id":"test","event":"PreToolUse"}'

# GET 查询当前状态
curl http://localhost:3001/api/signal-light/state
```

### 4.10 `error` — 错误

```json
{ "type": "error", "error": "invalid_json" }
```

| `error` | 说明 |
| --- | --- |
| `invalid_json` | 客户端报文不是合法 JSON |
| `unknown_type` | `type` 字段未识别 |

## 5. 后续待加报文（规划）

| type | 方向 | 用途 |
| --- | --- | --- |
| `auth` | C→S | 鉴权 token 上送 |
| `system_stats` | S→C | 服务端推送实时 CPU/GPU/内存指标，给报告页用 |
| `light_state` | C→S | 红绿灯页用户选灯 |
| `notification` | S→C | 系统通知 / toast |
