# AI Agent 信号灯系统

把 Claude Code 的运行状态变成手机上的红绿灯：Agent 在干活、等授权、被阻塞时，iapp 的「红绿灯」页实时点亮对应灯色。

## 链路总览

```
Claude Code 触发 hook 事件
        │  (SessionStart / PreToolUse / Stop / Notification ...)
        ▼
~/.claude/scripts/signal_light_hook.py   ← settings.json hooks 配置
        │  事件 → 灯语 signal 映射 + 读 stdin payload
        ▼
POST https://mini-watch.t8s.ink/api/signal-light/state
        │  { signal, session_id, event, cwd }
        ▼
后端 server/src/api/signal-light.ts
        ├─ unstorage 维护 sessions 字典（24h TTL）
        ├─ 多会话聚合：blocked > attention > working > idle
        └─ broadcastToAll({ type:'signal_light', aggregate, ... })
        ▼
WebSocket 推送所有 peer（手机 + Web 控制台）
        ▼
iapp wsStore → signalLightStore.apply(...)
        ▼
TrafficLightPage 渲染 3 盏 GLSL 灯
```

## 1. Hook 脚本

位置（宿主机）：`~/.claude-docker/claude-home/scripts/signal_light_hook.py`
（容器内映射为 `~/.claude/scripts/signal_light_hook.py`）

- **纯 stdlib**，无第三方依赖，`#!/usr/bin/env python3` 可直接执行
- 读 stdin JSON + 命令行事件名 → 映射成 signal
- POST 到后端，任何网络/解析错误**一律静默**（hook 不能阻断 Claude）
- 临时日志写 `/tmp/signal-light-hook.log`（需 `SIGNAL_LIGHT_DEBUG=1`）

### 事件 → 灯语映射

| Claude 事件 | signal | 聚合后灯色 |
| --- | --- | --- |
| `SessionStart` | `session_start` | 绿 |
| `UserPromptSubmit` | `thinking` | 三色循环 |
| `PreToolUse` | `working` | 三色循环 |
| `PostToolUse` | `tool_done` | 三色循环 |
| `Stop` | `turn_end` | 清除该会话 |
| `Notification` | `attention` | 黄闪 |
| `PermissionRequest` | `permission` | 红闪 |
| `SessionEnd` | `session_end` | 清除该会话 |
| `Stop` + `stop_reason=error/max_tokens` | `blocked` | 红闪 |

### 环境变量

| 变量 | 默认 | 说明 |
| --- | --- | --- |
| `SIGNAL_LIGHT_API` | `https://mini-watch.t8s.ink/api/signal-light/state` | 上报地址 |
| `SIGNAL_LIGHT_TIMEOUT` | `2.0` | HTTP 超时秒数 |
| `SIGNAL_LIGHT_DEBUG` | — | `1` 时写 `/tmp` 调试日志 |

## 2. settings.json hook 配置

位置：`~/.claude-docker/claude-home/settings.json`

```json
{
  "env": {
    "SIGNAL_LIGHT_API": "https://mini-watch.t8s.ink/api/signal-light/state"
  },
  "hooks": {
    "SessionStart":     [{ "matcher": "", "hooks": [{ "type": "command", "command": "./scripts/signal_light_hook.py SessionStart" }] }],
    "UserPromptSubmit": [{ "matcher": "", "hooks": [{ "type": "command", "command": "./scripts/signal_light_hook.py UserPromptSubmit" }] }],
    "PreToolUse":       [{ "matcher": "", "hooks": [{ "type": "command", "command": "./scripts/signal_light_hook.py PreToolUse" }] }],
    "PostToolUse":      [{ "matcher": "", "hooks": [{ "type": "command", "command": "./scripts/signal_light_hook.py PostToolUse" }] }],
    "Stop":             [{ "matcher": "", "hooks": [{ "type": "command", "command": "./scripts/signal_light_hook.py Stop" }] }],
    "Notification":     [{ "matcher": "", "hooks": [{ "type": "command", "command": "./scripts/signal_light_hook.py Notification" }] }],
    "SessionEnd":       [{ "matcher": "", "hooks": [{ "type": "command", "command": "./scripts/signal_light_hook.py SessionEnd" }] }]
  }
}
```

> settings.json 启动时读取，改完需重启 Claude 会话。
> 命令用 `./scripts/...` 相对路径；若 cwd 不在家目录报错，改用 `~/.claude/scripts/...`。

## 3. 后端聚合

`server/src/api/signal-light.ts`

| 端点 | 方法 | 用途 |
| --- | --- | --- |
| `/api/signal-light/state` | POST | hook 上报事件 |
| `/api/signal-light/state` | GET | 查询当前聚合状态 |

### 聚合规则

多个 Claude 会话同时跑，取「最紧迫」的状态：

```
红 (blocked/permission)  >  黄 (attention/done)  >  working  >  idle (绿)
```

- 红：权限请求 / 阻塞 / 失败 —— 卡住流程，需立即处理
- 黄：纯通知 —— 等你看一眼但不阻塞

- 会话状态存 unstorage（内存），每会话独立 `{ signal, updated_at }`
- `session_end` / `turn_end` / `off` 删除会话条目
- 超过 24h 未更新的会话自动剪枝
- 聚合后通过 `broadcastToAll` 走 ws 推送（见 [ws.md §4.9](./ws.md)）

## 4. Flutter 端

| 文件 | 职责 |
| --- | --- |
| `lib/store/signal_light_store.dart` | 4 态 enum（idle/working/attention/blocked）+ ChangeNotifier |
| `lib/main.dart` | ws `signal_light` 报文 → `signalLightStore.apply` |
| `lib/pages/traffic_light_page.dart` | 3 盏 GLSL 灯渲染 + 动画 |
| `shaders/traffic_light.frag` | 复古单灯着色器（球面光 + 颗粒 + 扫描线） |

### 灯效

| 聚合状态 | 灯 | 灯效 |
| --- | --- | --- |
| `idle` | 绿 | 常亮 |
| `working` | 黄 | 慢闪呼吸（正弦 0.15↔1.0，约 1.2s 一个呼吸） |
| `attention` | 黄 | 快闪（方波 亮1.0/灭0.1，约 0.5s） |
| `blocked` | 红 | 快闪（同 attention 节奏） |

顶部状态条显示：当前状态文字 + 原始 signal 名 + 活跃会话数。

> working 与 attention 同为黄灯，靠**节奏**区分：工作中是平滑呼吸，需要关注是硬切快闪。

### 亮度驱动实现

`LightShader` 接收 `intensity: double`（0..1）而非 `active: bool`：

- shader 内 `intensity = mix(0.06, 1.0, uIntensity)`，传小数即得任意亮度（0=暗玻璃，1=全亮）
- `TrafficLightPage` 用单个 `AnimationController`（repeat 2s）派生两种波形：
  - `_breath`：`(sin(t·2π)+1)/2` 映射到 `0.15..1.0` → 呼吸
  - `_flash`：`(t·4)%1 < 0.5 ? 1.0 : 0.1` → 快闪
- 外层 `BoxShadow` 的 alpha / blur / spread 同步乘 `intensity`，光晕跟着明暗

## 5. 手动测试

不依赖 Claude，直接打 HTTP：

```bash
# 红灯闪（阻塞）
curl -X POST https://mini-watch.t8s.ink/api/signal-light/state \
  -H 'Content-Type: application/json' \
  -d '{"signal":"blocked","session_id":"test"}'

# 三色循环（工作中）
curl -X POST https://mini-watch.t8s.ink/api/signal-light/state \
  -H 'Content-Type: application/json' \
  -d '{"signal":"working","session_id":"test"}'

# 清除会话 → 回到绿灯
curl -X POST https://mini-watch.t8s.ink/api/signal-light/state \
  -H 'Content-Type: application/json' \
  -d '{"signal":"session_end","session_id":"test"}'

# 查询当前聚合状态
curl https://mini-watch.t8s.ink/api/signal-light/state
```

## 设计来源

灯语语义与事件映射沿用 [vibecoding-signal-light](https://github.com/) 项目，区别：

| | 原项目 | 本项目 |
| --- | --- | --- |
| 输出 | MCP2221A USB GPIO 实体灯 | iapp 手机端 GLSL 虚拟灯 |
| 聚合 | 本地 Python + `/tmp` 状态文件 | 后端 unstorage + 多端 ws 广播 |
| 传输 | 本地进程直驱 | HTTP 上报 → WebSocket 下发 |
