# mini-watch 项目总览

把 macOS 主机的实时状态和 AI Agent 的运行状态，投射到一台旧 iPhone 上当**实体仪表盘**用：左滑是系统监控大屏，再滑是 Claude Code 的「红绿灯」，PC 控制台可远程遥控这台手机。

## 三个组成部分

| 模块 | 位置 | 技术栈 | 职责 |
| --- | --- | --- | --- |
| **手机 App** | `iapp/`（即本目录，`/workspace`） | Flutter / iOS | 仪表盘 UI、GLSL 着色器、接收 ws 指令 |
| **后端服务** | `iapp/server/` | Hono + Vue3 + pkg | 采集主机指标、聚合灯语、ws 广播、PC 控制台 |
| **Claude Hook** | `~/.claude-docker/.../scripts/signal_light_hook.py` | Python 标准库 | Claude Code 事件 → HTTP 上报后端 |

```
┌─────────────┐   采集/聚合    ┌──────────────┐   WebSocket    ┌──────────────┐
│  macOS 主机  │ ────────────▶ │   后端服务    │ ◀────────────▶ │  iPhone App   │
│ (CPU/温度…)  │  macmon/si    │ (Hono :4600) │   JSON 报文     │  (Flutter)    │
└─────────────┘               └──────────────┘                └──────────────┘
                                     ▲  ▲
              Claude hook HTTP 上报 ──┘  └── PC 控制台 (Vue 内嵌同端口)
```

## 手机 App（`iapp/`）

横向 `PageView` 三屏（`lib/pages/home_pager.dart`），切页会上报、也可被远程跳转：

1. **系统报告** `report_page` — CPU/GPU/内存/温度/磁盘/网络/风扇/健康度卡片，5s 刷新
2. **红绿灯** `traffic_light_page` — Agent 灯语（绿=空闲 / 黄呼吸=工作中 / 三灯灭=需关注 / 红闪=阻塞），背景点阵波浪 + 运行中外壳呼吸边框
3. **智能体** `agent_page` — 占位（TODO）

辅助：`settings_page`（主题切换）。

关键目录：
- `lib/components/` — 复用组件，含 4 个着色器封装（`aura_background` 点阵波浪背景、`light_shader` 单灯、`earth_sphere` 健康度地球、`fan_shader` 风扇）+ `breathing_container` 呼吸边框 + `top_bar`
- `lib/store/` — `ChangeNotifier` 单例：`ws_store`(连接/心跳/重连)、`system_stats_store`、`base_info_store`、`signal_light_store`
- `lib/styles/` — 5 套主题 + `ThemeScope`(InheritedWidget) 运行时切换
- `lib/routes/app_routes.dart` — **页面注册表**（id/中文名/path），增减页面只改这里
- `shaders/*.frag` — GLSL，新增需在 `pubspec.yaml` 的 `shaders:` 登记并重跑

## 后端服务（`iapp/server/`）

Hono API + 内嵌 Vue 控制台，最终用 pkg/esbuild 打成单文件。

- `src/lib/system-stats.ts` — `systeminformation` + `mac-extras`(macmon 免 sudo) 采集，历史落 `data/stats-history.json`，仅当有客户端在「系统报告」页时才采集
- `src/lib/mac-extras.ts` — 常驻 `macmon pipe` 子进程，解析温度/GPU/RAM/风扇
- `src/lib/base-info.ts` — 主机名/型号/芯片/内存/系统版本（启动时采集进 KV）
- `src/api/websocket.ts` — ws 中枢，`broadcastToAll` + 在线 peer 管理 + 远程指令转发
- `src/api/signal-light.ts` — Claude 灯语上报入口，多会话聚合
- `src/client/` — Vue3 + Pinia 控制台：主题切换、页面切换、亮度调节、在线状态

## Claude 信号灯链路

Claude Code 触发 hook → `signal_light_hook.py` 映射灯语 → `POST /api/signal-light/state` → 后端按多会话聚合（红>黄>工作中>空闲）→ ws 广播 → 手机红绿灯页点亮。详见 [signal-light.md](./signal-light.md)。

## WebSocket 报文

手机与后端单条长连接交换 JSON，类型含 `system_stats` / `base_info` / `theme_change` / `page_change` / `goto_page` / `brightness_change` / `signal_light` 等。完整格式见 [ws.md](./ws.md)。

## 构建 / 部署

项目根 `Makefile`（`mini-watch/Makefile`）：

```bash
make unsigned        # 出未签名 ipa -> dist/app.ipa（默认目标）
make ipa             # 出已签名 ipa（需开发证书）
make build-server    # 后端打成 macOS arm64 独立二进制 -> dist/
make deploy-server   # pnpm bundle + pm2 startOrReload（端口 4600）
make clean
```

pm2 配置为根目录 `ecosystem.config.cjs`（`mini-watch-server`，启动 `iapp/server/dist/bundle.cjs`）。

## 相关文档

- [ws.md](./ws.md) — WebSocket 报文协议
- [signal-light.md](./signal-light.md) — AI Agent 信号灯系统
