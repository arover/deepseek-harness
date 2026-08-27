# Tauri 桌面客户端（macOS 优先）设计

- 日期：2026-08-27
- 分支：`desktop/tauri-macos`
- 状态：已与用户逐节确认（集成模式、交付形态、后端来源、三节设计）

## 目标

在 DeepSeek Harness 上新增 Tauri 桌面客户端：原生 macOS 窗口内运行现有 Web GUI，由桌面壳托管 `dsh web` 后端进程。首期只保证 macOS 可用并可产出未签名 `.app`/`.dmg` 自用；浏览器版 Web GUI 行为完全不变。

## 非目标

- 不实现 Windows/Linux 打包（配置留占位，不启用）。
- 不做 Apple Developer ID 签名与公证。
- 不修改 `packages/**`、`apps/web`、`apps/cli` 的任何现有文件。
- 不复制或旁路 `dsh web` 的启动清单注入（`window.__DSH_BOOT__`）与 `/api` 浏览器信任围栏——两者仍是服务端独有职责。

## 架构

```
apps/desktop (Tauri 2 壳)
 ├─ WebView (WKWebView) ──导航──> http://127.0.0.1:<随机端口>   ← origin 不变，围栏天然通过
 └─ Rust 编排器 ──spawn──> dsh web --no-open --port 0 子进程     ← 现有 Node 后端，零改动
```

桌面壳是纯编排层：不携带业务前端资产，业务界面始终由 `dsh web` 服务并注入启动清单。

### 被否决的替代方案

- **自定义协议反代**（WebView 从 `tauri://localhost` 加载，Tauri 反向代理到本地后端）：必须给信任围栏增加 tauri 来源或放宽围栏，改动现有安全语义；代理层需自行处理 SSE 流式转发，重复实现 `dsh web` 已有行为。
- **在 `dsh --profile web` 内加 `--desktop` 由 Node 主导开窗**：与使用 Tauri 框架的要求相悖，且把桌面关注点混进 Web 服务包。

## 代码组织（全部为新增）

```
apps/desktop/
├── package.json          # @deepseek-ai/dsh-desktop；scripts: dev / build
├── src-tauri/            # Tauri 2 Rust crate：main.rs + backend.rs
├── src/                  # 极薄前端：就绪前显示状态页，随后由窗口导航接管
└── tauri.conf.json       # bundle targets 只开 app/dmg（macOS）
```

`apps/*` 已在 pnpm workspace glob 内，新增目录即自动成为成员。根 `package.json` 增加 `"desktop": "pnpm --filter @deepseek-ai/dsh-desktop dev"` 别名。桌面文档放 `apps/desktop/README.md`。

## 后端进程生命周期

### 探测顺序

首次找到即用；三条路径全部失败时进入诊断页：

1. 用户覆盖项：环境变量 `DSH_DESKTOP_CMD` 指定的自定义启动命令（显式 > 隐式）。
2. PATH 上的 `dsh` 可执行文件。
3. 本仓库源码模式：向上定位 checkout 根（以 `package.json` + `apps/cli/src/bin.ts` 为标识），执行 `node --import tsx/esm apps/cli/src/bin.ts --profile web --no-open --port 0`。

### 启动与就绪

- 统一追加 `--no-open --port 0`：不弹系统浏览器、由 OS 分配随机空闲端口，避免与已运行的 3080 实例冲突。
- 就绪判定双通道：解析子进程输出中的地址宣告行 `dsh web: http://...`，辅以 TCP 端口探测兜底；两路都确认端口后才导航 WebView。
- 就绪超时经环境变量 `DSH_DESKTOP_READY_TIMEOUT_MS` 配置，默认 30000；超时进入诊断页并保留子进程尾部日志。
- 诊断页是壳自带薄前端（`src/`）的状态视图：后端未就绪、启动失败、运行中崩溃三种状态都渲染在这里，同一组件显示探测结果与尾部日志并提供重试按钮。

### 退出清理

- 以独立进程组 spawn，退出顺序为 `killpg(SIGTERM)` 宽限、超时后 SIGKILL。
- 窗口关闭、`Cmd+Q`、桌面进程被信号杀死三条路径都收敛到同一清理逻辑（Rust Drop + 信号处理兜底）。
- 后端崩溃时 WebView 导航到诊断页，提供一键重启。

## macOS 适配

- `tauri.conf.json`：identifier、单窗口、标准红绿灯按钮、bundle targets 只开 `app` 与 `dmg`（未签名）；Windows/Linux 相关键不启用。
- `Cmd+Q` 退出；单实例：重复启动聚焦已有窗口并复用其后端。
- 窗口标题 "DSH Desktop"，构建标题遵循 `DSH_CLIENT_TITLE` 同源逻辑。
- 能力最小化：不申请 Tauri capability 之外的系统能力；WKWebView 访问 `127.0.0.1` 属于默认允许的本地出站连接，无额外 entitlement。
- 探测逻辑预留 Windows 差异（`where dsh` / `pnpm.cmd`）接口，首期不实现。

## 错误处理

| 故障 | 行为 |
|---|---|
| 探测不到 node/dsh | 启动即诊断页：列出三条探测路径各自的结果 |
| 后端启动超时/异常退出 | 诊断页显示子进程尾部日志 + 重试按钮 |
| 后端运行中崩溃 | WebView 导航到诊断页，提供一键重启 |

所有故障对用户可见，不静默吞掉。

## 测试策略

- Rust 侧核心逻辑（探测顺序、地址宣告行解析、清理顺序）拆成可单元测试的纯函数，`cargo test` 覆盖。
- 关键分支的正则与解析若落在 TS 侧则补 vitest 单测；纯 Rust 内部实现以 cargo test 为准。
- README 写入手动验收清单：`pnpm desktop dev` 起原生窗口 → 自动拉起后端 → 关窗后子进程树归零。
- 本变更不改 packages/*，仓库级门禁不受影响；跑 desktop 应用自身构建与测试即可。
