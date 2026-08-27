# Tauri 桌面客户端（macOS 优先）实现计划

> **面向 AI 代理的工作者：** 必需子技能：使用 superpowers:subagent-driven-development（推荐）或 superpowers:executing-plans 逐任务实现此计划。步骤使用复选框（`- [ ]`）语法来跟踪进度。

**目标：** 新增 `apps/desktop` Tauri 2 桌面壳，在原生 macOS 窗口内运行现有 Web GUI，并由壳托管 `dsh web` 后端子进程；浏览器版 Web GUI 行为不变。

**架构：** 壳不携带业务前端——Rust 编排器按 探测 → spawn → 就绪等待 三步拉起后端，就绪后用 `window.eval("window.location.replace(url)")` 把 WebView 导航到 `http://127.0.0.1:<port>`（origin 不变，服务端信任围栏天然通过）。失败与崩溃时 WebView 回到壳自带诊断页。规格见 [2026-08-27-tauri-desktop-macos-design.md](../specs/2026-08-27-tauri-desktop-macos-design.md)。

**技术栈：** Tauri 2（Rust）、tauri-plugin-single-instance、signal-hook、libc 进程组管理、pnpm workspace。

---

## 背景事实（工作者必读）

- 仓库是 pnpm workspace，`apps/*` 已被 glob 收纳：新增 `apps/desktop/package.json` 即成为成员。
- `dsh web` 是 `--profile web` 的硬编码别名（`apps/cli/src/args.ts:156`）；源码启动等价命令为根目录的 `node --import tsx/esm apps/cli/src/bin.ts web <flags>`。
- 后端把绑定地址打印到 stdout：`dsh web: http://127.0.0.1:<port>`（可能跟 ` (LAN: http://…)` 后缀），来自 `packages/bundle/web-app/src/index.ts` 的 `announceReady`；该行默认打印（`printUrl` 默认 true），`--no-open` 只禁浏览器接管。
- 前端静态资源要求 `@deepseek-ai/dsh-web-frontend/dist/index.html` 已构建；checkout 中若缺失会启动报错。
- 本机工具链（计划编写时实测）：node v22.22.1、pnpm 11.24.0、rustc/cargo 1.94.0、macOS 26.6.2。
- 系统提示约定：本项目 Agent Note 放 `.agents/notes/implemented/<kind>/YYYY-MM-DD-<slug>.md`。

## 文件结构

| 文件 | 创建/修改 | 职责 |
|---|---|---|
| `apps/desktop/package.json` | 创建 | workspace 成员声明；`dev`/`build`/`icon` 脚本 |
| `apps/desktop/.gitignore` | 创建 | 忽略 Rust 构建产物与 Tauri 生成物 |
| `apps/desktop/src/index.html` | 创建 | 诊断/状态薄前端：监听事件、显示日志尾部与重试按钮 |
| `apps/desktop/src-tauri/Cargo.toml` | 创建 | Rust 依赖清单 |
| `apps/desktop/src-tauri/build.rs` | 创建 | tauri-build 入口 |
| `apps/desktop/src-tauri/tauri.conf.json` | 创建 | macOS 优先窗口与打包配置 |
| `apps/desktop/src-tauri/capabilities/default.json` | 创建 | 最小能力集（core 默认权限） |
| `apps/desktop/src-tauri/src/probe.rs` | 创建 | 后端命令探测纯函数（依赖注入可测） |
| `apps/desktop/src-tauri/src/readiness.rs` | 创建 | 宣告行解析、连接参数拆分、超时读取 |
| `apps/desktop/src-tauri/src/supervisor.rs` | 创建 | 子进程 spawn / 进程组清理（SIGTERM→SIGKILL）、日志环形缓冲 |
| `apps/desktop/src-tauri/src/main.rs` | 创建 | Tauri 应用接线：编排主循环（宣告→导航→持续排水→崩溃回诊断页）、单实例、信号处理、restart 命令 |
| `apps/desktop/src-tauri/icons/` | 创建 | 由 `pnpm tauri icon` 从 `apps/web/public/favicon.svg` 生成 |
| `apps/desktop/README.md` | 创建 | 运行手册与手动验收清单 |
| `.agents/notes/implemented/architecture/2026-08-27-tauri-desktop-shell-managed-backend.md` | 创建 | 决策记录：托管薄壳 vs 反代/内嵌 |
| 根 `package.json` | 修改 | 追加 `"desktop"` 脚本别名 |

职责边界：probe/readiness 是无 IO 副作用的纯逻辑层；supervisor 是唯一触碰子进程生命周期的模块（日志缓冲也在其内）；main.rs 只做接线与输出编排，不做决策。关键并发模型：**stdout 主循环同时承担就绪检测与崩溃看门狗**（读到 EOF 即崩溃），stderr 由常驻排水线程喂同一份环形缓冲——这保证两条管道永远不会因无人读取而阻塞后端写入，也不需要额外的轮询线程。

---

### 任务 1：Tauri 脚手架与最小窗口

**文件：**
- 创建：`apps/desktop/package.json`
- 创建：`apps/desktop/.gitignore`
- 创建：`apps/desktop/src/index.html`
- 创建：`apps/desktop/src-tauri/Cargo.toml`
- 创建：`apps/desktop/src-tauri/build.rs`
- 创建：`apps/desktop/src-tauri/tauri.conf.json`
- 创建：`apps/desktop/src-tauri/capabilities/default.json`

- [ ] **步骤 1.1：写 package.json**

```json
{
  "name": "@deepseek-ai/dsh-desktop",
  "description": "Tauri desktop shell over the dsh-web GUI; owns and supervises a local dsh web backend process",
  "version": "0.1.1-rc.2",
  "private": true,
  "type": "module",
  "scripts": {
    "dev": "tauri dev",
    "build": "tauri build",
    "icon": "tauri icon ../web/public/favicon.svg"
  },
  "devDependencies": {
    "@tauri-apps/cli": "^2"
  }
}
```

- [ ] **步骤 1.2：写 .gitignore**

```gitignore
/target/
/gen/schemas/
```

Cargo.lock 保持入库（应用锁文件），不要加进 ignore。

- [ ] **步骤 1.3：写诊断页 src/index.html**

```html
<!doctype html>
<html lang="zh-CN">
<head>
  <meta charset="utf-8" />
  <title>DSH Desktop</title>
  <style>
    body { font-family: -apple-system, sans-serif; margin: 2rem; background: #111; color: #eee; }
    pre { background: #1c1c1e; padding: 1rem; border-radius: 8px; white-space: pre-wrap; max-height: 45vh; overflow-y: auto; }
    button { font-size: 1rem; padding: .5rem 1.25rem; }
    #attempts li { margin: .25rem 0; }
  </style>
</head>
<body>
  <h1 id="title">正在启动后端…</h1>
  <ul id="attempts"></ul>
  <pre id="log">（暂无输出）</pre>
  <p><button id="retry" hidden>重启后端</button></p>
  <script>
    const { listen } = window.__TAURI__.event
    const { invoke } = window.__TAURI__.core
    const title = document.getElementById('title')
    const attempts = document.getElementById('attempts')
    const log = document.getElementById('log')
    const retry = document.getElementById('retry')
    listen('backend-status', (event) => {
      const s = event.payload
      title.textContent = s.phase === 'failed' ? '后端启动失败' : (s.phase === 'crashed' ? '后端已退出' : s.message)
      if (s.attempts) {
        attempts.innerHTML = ''
        for (const a of s.attempts) {
          const li = document.createElement('li')
          li.textContent = a
          attempts.append(li)
        }
      }
      if (s.log_tail && s.log_tail.length > 0) log.textContent = s.log_tail.join('\n')
      retry.hidden = !(s.phase === 'failed' || s.phase === 'crashed')
    })
    retry.addEventListener('click', () => invoke('restart_backend'))
  </script>
</body>
</html>
```

依赖 `"withGlobalTauri": true`（见步骤 1.6），使页面无需打包器即可拿到 `window.__TAURI__`。事件字段契约：`phase ∈ probing|starting|failed|crashed`，`message` 人类可读，`attempts` 仅失败时出现，`log_tail` 失败/崩溃时带尾部日志。

- [ ] **步骤 1.4：写 Cargo.toml**

```toml
[package]
name = "dsh-desktop"
version = "0.1.0"
edition = "2021"

[build-dependencies]
tauri-build = { version = "2", features = [] }

[dependencies]
libc = "0.2"
serde = { version = "1", features = ["derive"] }
serde_json = "1"
signal-hook = "0.3"
tauri = { version = "2", features = [] }
tauri-plugin-single-instance = "2"

[[bin]]
name = "dsh-desktop"
path = "src/main.rs"
```

- [ ] **步骤 1.5：写 build.rs**

```rust
fn main() {
    tauri_build::build()
}
```

- [ ] **步骤 1.6：写 tauri.conf.json**

```json
{
  "$schema": "https://schema.tauri.app/config/2",
  "productName": "DSH Desktop",
  "version": "0.1.0",
  "identifier": "ai.deepseek.dsh.desktop",
  "build": {
    "frontendDist": "../src"
  },
  "app": {
    "withGlobalTauri": true,
    "windows": [
      {
        "label": "main",
        "title": "DSH Desktop",
        "width": 1280,
        "height": 800,
        "minWidth": 960,
        "minHeight": 600
      }
    ],
    "security": { "csp": null }
  },
  "bundle": {
    "active": true,
    "targets": ["app", "dmg"],
    "icon": ["icons/icon.icns"]
  }
}
```

Windows/Linux 打包键保持不启用（非目标，见规格）。`frontendDist: ../src` 相对 `src-tauri/` 解析到诊断页。

- [ ] **步骤 1.7：写 capabilities/default.json**

```json
{
  "$schema": "../gen/schemas/desktop-schema.json",
  "identifier": "default",
  "windows": ["main"],
  "permissions": ["core:default"]
}
```

`core:default` 含 `core:event:default`，诊断页的 `listen` 因此可用；应用自有 command 无需额外授权。

- [ ] **步骤 1.8：生成图标**

```bash
cd apps/desktop && pnpm install && pnpm icon
```

预期：`src-tauri/icons/` 出现 `icon.icns` 等全套图标。（若 SVG 输入不被当前 CLI 版本接受，先用任一工具把 favicon 导出为 512×512 PNG 再 `pnpm exec tauri icon <png>`，目的不变。）

- [ ] **步骤 1.9：写占位 main.rs 让编译通过**

```rust
// 任务 5 会替换为本应用的完整接线；先保证脚手架可编译。
fn main() {
    tauri::Builder::default()
        .run(tauri::generate_context!())
        .expect("error while running dsh desktop");
}
```

- [ ] **步骤 1.10：验证编译**

```bash
cargo check --manifest-path apps/desktop/src-tauri/Cargo.toml
```

预期：首次拉取 crates 后 `Finished`，无 error。（首次构建耗时数分钟属正常。）

- [ ] **步骤 1.11：Commit**

```bash
git add apps/desktop
git commit -m "feat(desktop): scaffold tauri shell with minimal macOS window"
```

---

### 任务 2：后端探测纯函数 probe.rs

**文件：**
- 创建：`apps/desktop/src-tauri/src/probe.rs`
- 修改：`apps/desktop/src-tauri/src/main.rs`（顶部追加声明）

- [ ] **步骤 2.1：在 main.rs 追加模块声明并写失败的测试**

main.rs 顶部：

```rust
mod probe;
```

probe.rs 先写类型骨架与测试：

```rust
//! Resolve how this machine launches `dsh web`, in priority order.

use std::path::{Path, PathBuf};

/// 用户经此环境变量提供完整启动命令；空格分词，不支持引号转义。
pub const OVERRIDE_ENV: &str = "DSH_DESKTOP_CMD";

/// A fully-resolved backend launch command.
#[derive(Debug, Clone, PartialEq, Eq)]
pub struct ProbeOutcome {
    /// Human-readable resolution description for the diagnostics page.
    pub source_label: String,
    pub program: String,
    pub args: Vec<String>,
    /// Working directory for repository source mode; absent elsewhere.
    pub cwd: Option<PathBuf>,
}

/// Every probing path failed; entries are diagnostics-page lines.
#[derive(Debug, Clone)]
pub struct ProbeFailure {
    pub attempts: Vec<String>,
}

pub fn is_repo_root(_dir: &Path) -> bool { unimplemented!() }

pub fn find_repo_root(_start: &Path) -> Option<PathBuf> { unimplemented!() }

pub fn probe<E, P, R>(
    _read_env: E,
    _find_in_path: P,
    _repo_root_from: R,
) -> Result<ProbeOutcome, ProbeFailure>
where
    E: Fn(&str) -> Option<String>,
    P: Fn(&str) -> Option<PathBuf>,
    R: Fn() -> Option<PathBuf>,
{
    unimplemented!()
}

#[cfg(test)]
mod tests {
    use super::*;
    use std::path::PathBuf;

    #[test]
    fn override_env_wins_over_path_binary() {
        let outcome = probe(
            |k| (k == OVERRIDE_ENV).then(|| "/opt/wrapper/bin/dsh-web --no-open --port 0".to_string()),
            |_| Some(PathBuf::from("/usr/local/bin/dsh")),
            || None,
        )
        .unwrap();
        assert_eq!(outcome.program, "/opt/wrapper/bin/dsh-web");
        assert_eq!(outcome.args, vec!["--no-open", "--port", "0"]);
        assert_eq!(outcome.source_label, "DSH_DESKTOP_CMD override");
        assert!(outcome.cwd.is_none());
    }

    #[test]
    fn falls_back_to_path_binary_when_no_override() {
        let outcome = probe(
            |_| None,
            |_| Some(PathBuf::from("/usr/local/bin/dsh")),
            || None,
        )
        .unwrap();
        assert_eq!(outcome.program, "/usr/local/bin/dsh");
        // PATH 二进制补齐固定参数：profile 别名、浏览器交接关掉、随机端口。
        assert_eq!(outcome.args, vec!["web", "--no-open", "--port", "0"]);
        assert_eq!(outcome.source_label, "dsh on PATH (/usr/local/bin/dsh)");
    }

    #[test]
    fn repository_source_mode_uses_node_tsx_entry() {
        let root = PathBuf::from("/repo/deepseek-harness");
        let expected_root = root.clone();
        let outcome = probe(|_| None, |_| None, move || Some(expected_root)).unwrap();
        assert_eq!(outcome.program, "node");
        assert_eq!(
            outcome.args,
            vec![
                "--import", "tsx/esm",
                "apps/cli/src/bin.ts",
                "web", "--no-open", "--port", "0"
            ]
        );
        assert_eq!(outcome.cwd, Some(root));
    }

    #[test]
    fn all_three_paths_fail_with_recorded_attempts() {
        let failure = probe(|_| None, |_| None, || None).unwrap_err();
        assert_eq!(failure.attempts.len(), 3);
        assert!(failure.attempts[0].contains(OVERRIDE_ENV));
        assert!(failure.attempts[1].contains("PATH"));
        assert!(failure.attempts[2].contains("repository"));
    }

    #[test]
    fn repo_root_lookup_requires_both_markers() {
        assert!(!is_repo_root(&PathBuf::from("/")));
    }
}
```

- [ ] **步骤 2.2：运行测试确认失败**

```bash
cargo test --manifest-path apps/desktop/src-tauri/Cargo.toml probe::
```

预期：5 个测试全部 FAIL（panic: not implemented）。

- [ ] **步骤 2.3：实现**

把三个 `unimplemented!()` 与 `is_repo_root` 替换为：

```rust
/// Repository checkout marker: root manifest plus the tsx entry.
fn repo_markers(dir: &Path) -> bool {
    dir.join("package.json").is_file() && dir.join("apps/cli/src/bin.ts").is_file()
}

/// True when `dir` holds both repository markers.
pub fn is_repo_root(dir: &Path) -> bool {
    repo_markers(dir)
}

/// Walk up from `start` (exclusive) to find the checkout root.
pub fn find_repo_root(start: &Path) -> Option<PathBuf> {
    start.ancestors().skip(1).find(|dir| repo_markers(dir)).map(Path::to_path_buf)
}

fn source_mode_args() -> Vec<String> {
    ["web", "--no-open", "--port", "0"]
        .into_iter()
        .map(String::from)
        .collect()
}

/// Resolve the launch command: env override, then PATH binary, then source mode.
/// The closures keep this function pure — callers inject host access.
pub fn probe<E, P, R>(
    read_env: E,
    find_in_path: P,
    repo_root_from: R,
) -> Result<ProbeOutcome, ProbeFailure>
where
    E: Fn(&str) -> Option<String>,
    P: Fn(&str) -> Option<PathBuf>,
    R: Fn() -> Option<PathBuf>,
{
    let mut attempts = Vec::new();
    if let Some(raw) = read_env(OVERRIDE_ENV).filter(|raw| !raw.trim().is_empty()) {
        let mut tokens = raw.split_whitespace().map(String::from);
        let program = tokens.next().expect("filtered non-empty above");
        return Ok(ProbeOutcome {
            source_label: format!("{OVERRIDE_ENV} override"),
            program,
            args: tokens.collect(),
            cwd: None,
        });
    }
    attempts.push(format!("{}: 未设置或为空", OVERRIDE_ENV));

    if let Some(path) = find_in_path("dsh") {
        return Ok(ProbeOutcome {
            source_label: format!("dsh on PATH ({})", path.display()),
            program: path.to_string_lossy().into_owned(),
            args: source_mode_args(),
            cwd: None,
        });
    }
    attempts.push("PATH: 未找到 dsh 可执行文件".to_string());

    if let Some(root) = repo_root_from() {
        return Ok(ProbeOutcome {
            source_label: format!("仓库源码模式 ({})", root.display()),
            program: "node".to_string(),
            args: ["--import", "tsx/esm", "apps/cli/src/bin.ts"]
                .into_iter()
                .map(String::from)
                .chain(source_mode_args())
                .collect(),
            cwd: Some(root),
        });
    }
    attempts.push("repository: 可执行文件上层未找到 checkout".to_string());

    Err(ProbeFailure { attempts })
}
```

约定细节（写给步骤 2.4 之后仍要改代码的人）：失败 `attempts` 文案用中文以直接服务诊断页展示；`source_label` 保持英文断言短语与测试锁定一致。

- [ ] **步骤 2.4：运行测试确认通过**

```bash
cargo test --manifest-path apps/desktop/src-tauri/Cargo.toml probe::
```

预期：5 个测试全部 PASS。

- [ ] **步骤 2.5：Commit**

```bash
git add apps/desktop/src-tauri/src/probe.rs apps/desktop/src-tauri/src/main.rs
git commit -m "feat(desktop): injectable backend launch probe with recorded attempts"
```

---

### 任务 3：宣告行解析与超时读取 readiness.rs

**文件：**
- 创建：`apps/desktop/src-tauri/src/readiness.rs`
- 修改：`apps/desktop/src-tauri/src/main.rs`（顶部追加 `mod readiness;`）

- [ ] **步骤 3.1：写失败的测试与类型骨架**

readiness.rs：

```rust
//! Backend readiness primitives: announcement parsing, connect splits,
//! timeout bounds. Pure functions only — waiting lives in main.rs.

use std::time::Duration;

/** 唯一可信的就绪宣告来源：announceReady 打印行的首个空格分词。 */
pub const ANNOUNCE_PREFIX: &str = "dsh web: ";

/// Environment key overriding the readiness timeout, in milliseconds.
pub const TIMEOUT_ENV: &str = "DSH_DESKTOP_READY_TIMEOUT_MS";

/// Readiness bound when the environment does not name one.
pub const DEFAULT_READY_TIMEOUT_MS: u64 = 30_000;

/// Sleep granularity between readiness polls.
pub const POLL_INTERVAL: Duration = Duration::from_millis(250);

/// Host/port split used by the TCP readiness fallback.
pub struct ConnectParts {
    pub host: String,
    pub port: u16,
}

pub fn parse_announced_url(_line: &str) -> Option<String> { unimplemented!() }

pub fn ready_timeout_ms() -> u64 { unimplemented!() }

pub fn connect_parts(_url: &str) -> Option<ConnectParts> { unimplemented!() }

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn parses_announced_loopback_url_with_lan_suffix() {
        let line = "dsh web: http://127.0.0.1:41573 (LAN: http://192.168.1.8:41573)";
        assert_eq!(
            parse_announced_url(line).as_deref(),
            Some("http://127.0.0.1:41573")
        );
    }

    #[test]
    fn rejects_non_announce_lines_and_other_schemes() {
        assert_eq!(parse_announced_url("web-app: frontend dist ok"), None);
        assert_eq!(parse_announced_url("dsh web: https://example.invalid"), None);
        assert_eq!(parse_announced_url(""), None);
    }

    #[test]
    fn timeout_reads_env_with_default_fallback() {
        std::env::remove_var(TIMEOUT_ENV);
        assert_eq!(ready_timeout_ms(), DEFAULT_READY_TIMEOUT_MS);
        std::env::set_var(TIMEOUT_ENV, "1500");
        assert_eq!(ready_timeout_ms(), 1500);
        std::env::set_var(TIMEOUT_ENV, "not-a-number");
        assert_eq!(ready_timeout_ms(), DEFAULT_READY_TIMEOUT_MS);
        std::env::remove_var(TIMEOUT_ENV);
    }

    #[test]
    fn splits_url_into_connect_parts() {
        let parts = connect_parts("http://127.0.0.1:3080").unwrap();
        assert_eq!(parts.host, "127.0.0.1");
        assert_eq!(parts.port, 3080);
        assert!(connect_parts("http://bad-host-without-port").is_none());
    }
}
```

- [ ] **步骤 3.2：运行测试确认失败**

```bash
cargo test --manifest-path apps/desktop/src-tauri/Cargo.toml readiness::
```

预期：4 个测试全部 FAIL。注意：环境变量用例操纵进程级状态且默认并行执行，因此每个用例开头先 `remove_var` 兜底（骨架里已如此）；若未来并 行用例互相踩踏，用 `serial_test` 串行化这两个用例。

- [ ] **步骤 3.3：实现**

替换三个 `unimplemented!()`：

```rust
/// Extract the loopback URL from one line of backend output, else `None`.
pub fn parse_announced_url(line: &str) -> Option<String> {
    let rest = line.trim_start().strip_prefix(ANNOUNCE_PREFIX)?;
    let url = rest.split_whitespace().next()?;
    url.starts_with("http://").then(|| url.to_string())
}

/// Wait bound for backend startup: env-configured milliseconds, invalid values fall back.
pub fn ready_timeout_ms() -> u64 {
    std::env::var(TIMEOUT_ENV)
        .ok()
        .and_then(|raw| raw.parse::<u64>().ok())
        .unwrap_or(DEFAULT_READY_TIMEOUT_MS)
}

/// Parse `host:port` out of an http URL; `None` when malformed or port 0.
pub fn connect_parts(url: &str) -> Option<ConnectParts> {
    let rest = url.strip_prefix("http://")?;
    let authority = rest.split(['/', '?']).next()?;
    let (host, port_raw) = authority.rsplit_once(':')?;
    if host.is_empty() { return None; }
    let port = port_raw.parse::<u16>().ok()?;
    (port != 0).then_some(ConnectParts { host: host.to_string(), port })
}
```

- [ ] **步骤 3.4：运行测试确认通过**

```bash
cargo test --manifest-path apps/desktop/src-tauri/Cargo.toml readiness::
```

预期：4 个测试 PASS。

- [ ] **步骤 3.5：Commit**

```bash
git add apps/desktop/src-tauri/src/readiness.rs apps/desktop/src-tauri/src/main.rs
git commit -m "feat(desktop): parse backend URL announcement with configurable ready timeout"
```

---

### 任务 4：进程托管与清理 supervisor.rs

**文件：**
- 创建：`apps/desktop/src-tauri/src/supervisor.rs`
- 修改：`apps/desktop/src-tauri/src/main.rs`（顶部追加 `mod supervisor;`）

- [ ] **步骤 4.1：写失败的测试与骨架**

supervisor.rs：

```rust
//! Backend child ownership: spawn into its own process group, tear the whole
//! group down on quit (SIGTERM grace, then SIGKILL), and retain recent output
//! lines for diagnostics.

use std::process::{Child, Command};
use std::{io, time::Duration};

use crate::probe::ProbeOutcome;

/// SIGTERM grace period before the group gets SIGKILL.
const GRACE: Duration = Duration::from_secs(5);

/// Diagnostics log ring buffer keeping only the most recent lines.
pub struct LogTail;

impl LogTail {
    /// Buffer holding at most `cap` recent lines.
    pub fn new(_cap: usize) -> LogTail { unimplemented!() }
    /// Record one output line, evicting the oldest beyond capacity.
    pub fn push(&mut self, _line: String) {}
    /// Consume the buffer into plain lines, oldest first.
    pub fn into_vec(self) -> Vec<String> { Vec::new() }
}

/// Owns the spawned backend until `shutdown`; `Drop` is the last resort.
pub struct Supervisor;

impl Supervisor {
    /// Spawn `outcome` in a fresh process group capturing both output pipes.
    pub fn spawn(outcome: &ProbeOutcome) -> io::Result<Supervisor> {
        Self::spawn_command(build_command(outcome)?)
    }

    /// Same spawn contract for tests that build their own command.
    pub fn spawn_for_test(cmd: Command) -> io::Result<Supervisor> { unimplemented!() }

    /// Process-group id equals the leader pid (`process_group(0)`).
    pub fn pgid(&self) -> i32 { unimplemented!() }

    /// Stderr handle for the resident drain thread. Panics when called twice by misuse.
    pub fn take_stderr(&mut self) -> std::process::ChildStderr { unimplemented!() }

    /// SIGTERM the group, wait through the grace period, then SIGKILL survivors. Idempotent.
    pub fn shutdown(&mut self) {}

    pub(crate) fn build_command(outcome: &ProbeOutcome) -> io::Result<Command> { unimplemented!() }
}

fn spawn_command(mut cmd: Command) -> io::Result<Supervisor> { unimplemented!() }

impl Drop for Supervisor {
    fn drop(&mut self) {
        self.shutdown();
    }
}

/// Send `sig` to `-pgid`; ESRCH (group already gone) is success here.
fn signal_group(pgid: i32, sig: i32) { unimplemented!() }

#[cfg(test)]
mod tests {
    use super::*;
    use std::process::{Command, Stdio};

    fn sleep_cmd(seconds: &str) -> Command {
        let mut cmd = Command::new("/bin/sleep");
        cmd.arg(seconds);
        cmd
    }

    #[test]
    fn build_command_sets_args_and_cwd() {
        let outcome = ProbeOutcome {
            source_label: "test".into(),
            program: "/bin/echo".into(),
            args: vec!["hi".into()],
            cwd: Some(PathBuf::from("/tmp")),
        };
        let cmd = Supervisor::build_command(&outcome).unwrap();
        // Command 没有暴露参数访问器，用 Debug 输出断言。
        let debug = format!("{cmd:?}");
        assert!(debug.contains("\"hi\""));
        assert!(debug.contains("/tmp"));
        let _ = Stdio::null;
    }

    #[test]
    fn kill_group_sigkills_sleep_after_grace() {
        let mut sup = Supervisor::spawn_for_test(sleep_cmd("300")).unwrap();
        sup.shutdown();
        // 进程组已消亡：kill(-pgid, 0) 报 ESRCH。
        let alive = unsafe { libc::kill(-sup.pgid(), 0) };
        assert_eq!(alive, -1);
        assert_eq!(std::io::Error::last_os_error().raw_os_error(), Some(libc::ESRCH));
    }

    #[test]
    fn shutdown_takes_the_whole_process_group() {
        // 组内一个直接子进程（后台 sleep）加 shell 自身，cleanup 要全带走。
        let mut cmd = Command::new("/bin/sh");
        cmd.arg("-c").arg("sleep 300 & sleep 300; wait");
        let mut sup = Supervisor::spawn_for_test(cmd).unwrap();
        sup.shutdown();
        let alive = unsafe { libc::kill(-sup.pgid(), 0) };
        assert_eq!(alive, -1);
        assert_eq!(std::io::Error::last_os_error().raw_os_error(), Some(libc::ESRCH));
    }

    #[test]
    fn double_shutdown_is_idempotent() {
        let mut sup = Supervisor::spawn_for_test(sleep_cmd("300")).unwrap();
        sup.shutdown();
        sup.shutdown(); // 第二次必须安全且不 panic
    }

    #[test]
    fn log_tail_keeps_only_recent_lines() {
        let mut tail = LogTail::new(3);
        for i in 0..5 { tail.push(format!("line-{i}")); }
        assert_eq!(tail.into_vec(), vec!["line-2", "line-3", "line-4"]);
    }
}
```

实现真实 `spawn_for_test` 时需要 `use std::os::unix::process::CommandExt;` 与 `use std::process::Stdio;`（骨架省略了，落地时补上）。`LogTail` 测试引用 `ProbeOutcome.cwd` 类型依赖 `use std::path::PathBuf;`。

- [ ] **步骤 4.2：运行测试确认失败**

```bash
cargo test --manifest-path apps/desktop/src-tauri/Cargo.toml supervisor::
```

预期：进程组消除断言 FAIL（空 shutdown 不杀进程，`kill` 仍返回 0）；`build_command`/`pgid` 用例 panic。

- [ ] **步骤 4.3：实现**

```rust
/// Diagnostics log ring buffer keeping only the most recent lines.
pub struct LogTail {
    cap: usize,
    lines: std::collections::VecDeque<String>,
}

impl LogTail {
    /// Buffer holding at most `cap` recent lines.
    pub fn new(cap: usize) -> LogTail {
        LogTail { cap, lines: std::collections::VecDeque::with_capacity(cap.max(1)) }
    }

    /// Record one output line, evicting the oldest beyond capacity.
    pub fn push(&mut self, line: String) {
        if self.cap == 0 { return; }
        if self.lines.len() == self.cap {
            self.lines.pop_front();
        }
        self.lines.push_back(line);
    }

    /// Consume the buffer into plain lines, oldest first.
    pub fn into_vec(self) -> Vec<String> {
        self.lines.into_iter().collect()
    }
}

/// Owns the spawned backend until `shutdown`; `Drop` is the last resort.
pub struct Supervisor {
    child: Child,
    pgid: i32,
    shut_down: bool,
}

impl Supervisor {
    /// Spawn `outcome` in a fresh process group capturing both output pipes.
    pub fn spawn(outcome: &ProbeOutcome) -> io::Result<Supervisor> {
        spawn_command(Self::build_command(outcome)?)
    }

    /// Same spawn contract for tests that build their own command.
    pub fn spawn_for_test(mut cmd: Command) -> io::Result<Supervisor> {
        use std::os::unix::process::CommandExt;
        use std::process::Stdio;
        cmd.stdin(Stdio::null()).stdout(Stdio::piped()).stderr(Stdio::piped());
        cmd.process_group(0);
        let child = cmd.spawn()?;
        Ok(Supervisor { pgid: child.id() as i32, child, shut_down: false })
    }

    /// Process-group id equals the leader pid (`process_group(0)`).
    pub fn pgid(&self) -> i32 { self.pgid }

    /// Stderr handle for the resident drain thread. Panics when called twice by misuse.
    pub fn take_stderr(&mut self) -> std::process::ChildStderr {
        self.child.stderr.take().expect("stderr taken twice")
    }

    /// SIGTERM the group, wait through the grace period, then SIGKILL survivors. Idempotent.
    pub fn shutdown(&mut self) {
        if self.shut_down { return; }
        self.shut_down = true;
        signal_group(self.pgid, libc::SIGTERM);
        let deadline = std::time::Instant::now() + GRACE;
        while std::time::Instant::now() < deadline {
            match self.child.wait() {
                Ok(Some(_)) => return,
                Ok(None) => std::thread::sleep(Duration::from_millis(50)),
                Err(_) => break,
            }
        }
        signal_group(self.pgid, libc::SIGKILL);
        let _ = self.child.wait();
    }

    pub(crate) fn build_command(outcome: &ProbeOutcome) -> io::Result<Command> {
        let mut cmd = Command::new(&outcome.program);
        cmd.args(&outcome.args);
        if let Some(cwd) = &outcome.cwd {
            cmd.current_dir(cwd);
        }
        Ok(cmd)
    }
}

fn spawn_command(cmd: Command) -> io::Result<Supervisor> {
    Supervisor::spawn_for_test(cmd)
}

/// Send `sig` to `-pgid`; ESRCH (group already gone) is success here.
fn signal_group(pgid: i32, sig: i32) {
    let rc = unsafe { libc::kill(-pgid, sig) };
    if rc == -1 {
        let err = std::io::Error::last_os_error();
        if err.raw_os_error() != Some(libc::ESRCH) {
            eprintln!("dsh desktop: killpg({pgid}, {sig}) failed: {err}");
        }
    }
}
```

要点：stdout/stderr 都置 `piped`——调用方（任务 5）保证两条管道都有读者；`process_group(0)` 使子进程成为新进程组组长，`kill(-pgid)` 才能命中整组。

- [ ] **步骤 4.4：运行测试确认通过**

```bash
cargo test --manifest-path apps/desktop/src-tauri/Cargo.toml supervisor::
```

预期：5 个测试 PASS（含整组消亡断言）。

- [ ] **步骤 4.5：Commit**

```bash
git add apps/desktop/src-tauri/src/supervisor.rs apps/desktop/src-tauri/src/main.rs
git commit -m "feat(desktop): supervise backend process group with graceful teardown"
```

---

### 任务 5：main.rs 编排接线

**文件：**
- 修改：`apps/desktop/src-tauri/src/main.rs`（整体替换占位内容）
- 修改：`apps/desktop/src-tauri/Cargo.toml`（无需改动；确认 `serde_json` 已在依赖中即可）

- [ ] **步骤 5.1：整体替换 main.rs 为完整实现**

```rust
//! DSH desktop shell: manage a local `dsh web` backend and mirror it in a
//! native macOS webview window.

mod probe;
mod readiness;
mod supervisor;

use std::sync::atomic::{AtomicBool, Ordering};
use std::sync::{Arc, Mutex};

use serde::Serialize;
use tauri::{AppHandle, Emitter, Manager, WindowEvent};

use supervisor::{LogTail, Supervisor};

/// Shared supervisor slot owned by the app.
struct BackendState(Mutex<Option<Supervisor>>);

/// Set while orchestration runs so diagnostics-page restarts cannot overlap.
static ORCHESTRATING: AtomicBool = AtomicBool::new(false);

/// Event payload rendered by the bundled diagnostics page. Field names are
/// part of the shell contract with src/index.html.
#[derive(Serialize, Clone)]
struct BackendStatus {
    phase: String,
    message: String,
    #[serde(skip_serializing_if = "Option::is_none")]
    attempts: Option<Vec<String>>,
    #[serde(skip_serializing_if = "Vec::is_empty")]
    log_tail: Vec<String>,
}

fn report(handle: &AppHandle, phase: &str, message: impl Into<String>) {
    let payload = BackendStatus {
        phase: phase.to_string(),
        message: message.into(),
        attempts: None,
        log_tail: Vec::new(),
    };
    if let Some(window) = handle.get_webview_window("main") {
        let _ = window.emit("backend-status", payload);
    }
}

fn report_failure(
    handle: &AppHandle,
    phase: &str,
    message: impl Into<String>,
    attempts: Vec<String>,
    log_tail: Vec<String>,
) {
    let payload = BackendStatus {
        phase: phase.to_string(),
        message: message.into(),
        attempts: Some(attempts),
        log_tail,
    };
    if let Some(window) = handle.get_webview_window("main") {
        let _ = window.emit("backend-status", payload);
    }
}

fn navigate(handle: &AppHandle, js: String) {
    if let Some(window) = handle.get_webview_window("main") {
        let _ = window.eval(&js);
    }
}

/// Stop and clear the supervisor slot; safe to call repeatedly.
fn teardown_backend(handle: &AppHandle) {
    let state = handle.state::<BackendState>();
    if let Some(mut sup) = state.0.lock().unwrap().take() {
        sup.shutdown();
    }
}

/// One attempt cycle: probe, spawn, then run the stdout service loop that
/// doubles as readiness detection and crash watchdog. Any exit path either
/// navigates the webview forward (ready) or back to the diagnostics page.
fn orchestrate(handle: AppHandle, first_run: bool) {
    if !first_run {
        // Re-run from the diagnostics page: reload it before reporting anew.
        navigate(&handle, "window.location.replace('tauri://localhost/')".to_string());
    }
    report(&handle, "probing", "正在定位后端命令…");

    let exe = std::env::current_exe().ok();
    let outcome = match probe::probe(
        |key| std::env::var(key).ok(),
        |name| which_dsh(name),
        || exe.as_deref().and_then(probe::find_repo_root),
    ) {
        Ok(outcome) => outcome,
        Err(failure) => {
            report_failure(&handle, "failed", "未找到可用的 dsh 后端命令", failure.attempts, Vec::new());
            ORCHESTRATING.store(false, Ordering::SeqCst);
            return;
        }
    };

    report(&handle, "starting", format!("启动后端：{}", outcome.source_label));

    let sup = match Supervisor::spawn(&outcome) {
        Ok(sup) => sup,
        Err(err) => {
            report_failure(
                &handle,
                "failed",
                format!("无法启动后端进程：{err}"),
                vec![outcome.source_label.clone()],
                Vec::new(),
            );
            ORCHESTRATING.store(false, Ordering::SeqCst);
            return;
        }
    };

    // 读走 stdout/stderr 句柄；此时还没有任何线程持有它们。
    let mut sup = sup;
    let stdout = sup.take_stdout();
    let stderr = sup.take_stderr();
    *handle.state::<BackendState>().0.lock().unwrap() = Some(sup);

    // stderr 常驻排水线程：只追加共享尾部缓冲，别的什么都不判断。
    let err_tail: Arc<Mutex<LogTail>> = Arc::new(Mutex::new(LogTail::new(40)));
    let drain_tail = err_tail.clone();
    std::thread::spawn(move || {
        use std::io::BufRead;
        for line in std::io::BufReader::new(stderr).lines() {
            match line {
                Ok(line) => drain_tail.lock().unwrap().push(line),
                Err(_) => break,
            }
        }
    });

    serve_stdout(handle, stdout, Arc::clone(&err_tail));
}

/// Drain stdout until EOF. Inside the loop the same line stream drives
/// readiness (announce parse + TCP probe) and afterwards keeps draining so
/// the backend can never block on a full pipe. EOF routes the UI: it either
/// reports an unfinished start failure or a crash of an already-ready
/// backend, then tears down.
fn serve_stdout(handle: AppHandle, stdout: std::process::ChildStdout, err_tail: Arc<Mutex<LogTail>>) {
    use std::io::BufRead;
    let mut tail: Vec<String> = Vec::with_capacity(40);
    let started = std::time::Instant::now();
    let timeout = std::time::Duration::from_millis(readiness::ready_timeout_ms());
    // ready=false 时尚未宣告；announced=Some(url) 记录宣告值。
    let mut announced: Option<String> = None;
    let mut ready = false;

    for line in std::io::BufReader::new(stdout).lines() {
        let line = match line {
            Ok(line) => line,
            Err(_) => break, // EOF/error → 进程组已终止，走退出路径。
        };
        if tail.len() == 40 { tail.remove(0); }
        tail.push(line.clone());

        if ready { continue }                       // 就绪后专心排水直到 EOF。
        if announced.is_none() {
            announced = readiness::parse_announced_url(&line);
        }
        let Some(url) = announced.clone() else {
            if started.elapsed() < timeout { continue }
            report_failure(&handle, "failed", "后端在超时窗口内未就绪",
                vec!["backend never announced a usable URL within the timeout".to_string()],
                combined(&err_tail, &tail));
            navigate(&handle, DIAGNOSTICS_PAGE_JS.to_string());
            teardown_backend(&handle);
            ORCHESTRATING.store(false, Ordering::SeqCst);
            return;
        };

        // 宣告已到手：等它可连。可连即就绪——导航、复位标志、转入纯排水。
        if readiness::connect_parts(&url)
            .and_then(|parts| std::net::TcpStream::connect((parts.host.as_str(), parts.port)).ok())
            .is_some()
        {
            ready = true;
            ORCHESTRATING.store(false, Ordering::SeqCst);
            navigate(&handle, format!("window.location.replace('{url}')"));
            continue;
        }
        if started.elapsed() >= timeout {
            report_failure(&handle, "failed", "后端宣告的地址始终无法建立连接",
                vec![format!("announced: {url}")], combined(&err_tail, &tail));
            navigate(&handle, DIAGNOSTICS_PAGE_JS.to_string());
            teardown_backend(&handle);
            ORCHESTRATING.store(false, Ordering::SeqCst);
            return;
        }
        std::thread::sleep(readiness::POLL_INTERVAL); // 通告节奏兜底，避免空转烧 CPU。
    }

    // 循环结束 = 后端进程组消亡（含被外部 kill 的场景）。
    let log_tail = combined(&err_tail, &tail);
    if ready {
        let attempts = vec!["backend crashed after becoming ready".to_string()];
        report_failure(&handle, "crashed", "后端进程意外退出", attempts, log_tail);
    } else {
        let attempts = vec!["backend exited before announcing a usable URL".to_string()];
        report_failure(&handle, "failed", "后端进程未能完成启动", attempts, log_tail);
    }
    navigate(&handle, DIAGNOSTICS_PAGE_JS.to_string());
    teardown_backend(&handle);
    ORCHESTRATING.store(false, Ordering::SeqCst);
}

/** 壳自带诊断页地址：frontendDist 静态服务的 origin。 */
const DIAGNOSTICS_PAGE_JS: &str = "window.location.replace('tauri://localhost/')";

/// Merge stderr ring buffer and recent stdout lines, oldest first.
fn combined(err_tail: &Mutex<LogTail>, out_tail: &[String]) -> Vec<String> {
    let mut merged = err_tail.lock().unwrap().clone().into_vec();
    merged.extend(out_tail.iter().cloned());
    merged
}

/// Resolve `name` on $PATH exactly like a shell would.
fn which_dsh(name: &str) -> Option<std::path::PathBuf> {
    let path = std::env::var_os("PATH")?;
    std::env::split_paths(&path)
        .map(|dir| dir.join(name))
        .find(|candidate| candidate.is_file())
}

/// Restart command invoked from the diagnostics page.
#[tauri::command]
fn restart_backend(handle: AppHandle) {
    if ORCHESTRATING.swap(true, Ordering::SeqCst) { return; }
    teardown_backend(&handle);
    std::thread::spawn(move || orchestrate(handle, false));
}

fn main() {
    tauri::Builder::default()
        .plugin(tauri_plugin_single_instance::init(|app, _args, _cwd| {
            if let Some(window) = app.get_webview_window("main") {
                let _ = window.set_focus();
            }
        }))
        .manage(BackendState(Mutex::new(None)))
        .invoke_handler(tauri::generate_handler![restart_backend])
        .setup(|app| {
            let signal_handle = app.handle().clone();
            let term_signals = signal_hook::iterator::Signals::new(signal_hook::consts::term_signals())
                .expect("install termination handlers");
            std::thread::spawn(move || {
                for _sig in term_signals.forever() {
                    teardown_backend(&signal_handle);
                    std::process::exit(0);
                }
            });
            let boot_handle = app.handle().clone();
            std::thread::spawn(move || orchestrate(boot_handle, true));
            Ok(())
        })
        .on_window_event(|window, event| {
            if matches!(event, WindowEvent::Destroyed) {
                teardown_backend(window.app_handle());
            }
        })
        .run(tauri::generate_context!())
        .expect("error while running dsh desktop");
}
```

**工作者须知（并发契约，实现必须原样满足）：** stdout 由 `serve_stdout` 独占读取直至进程组消亡，stderr 由常驻排水线程独占读取——两条管道从 spawn 起到进程组终止永远各有一个存活读者，后端不会因管道写满而阻塞。所有失败/崩溃路径都 emit 恰好一次终态事件（`failed` 或 `crashed`）并把 `ORCHESTRATING` 复位；就绪路径导航前先复位 `ORCHESTRATING`。窗口销毁引发的 teardown 会杀掉进程组、使两处读到 EOF，随后 emit 的事件发往已销毁窗口属预期无害。

- [ ] **步骤 5.2：编译检查**

```bash
cargo check --manifest-path apps/desktop/src-tauri/Cargo.toml
```

预期：`Finished` 无 error。Tauri 小版本若调整了 API 名称（如 `get_webview_window`、`WindowEvent::Destroyed`），以 `cargo doc --open` 实际签名为准修正 import；行为契约以上文并发契约为准，不得为编译便利删除排水线程或就绪双通道中的任一路径。

- [ ] **步骤 5.3：跑全部单元测试回归**

```bash
cargo test --manifest-path apps/desktop/src-tauri/Cargo.toml
```

预期：任务 2/3/4 的 14 个用例全部 PASS。

- [ ] **步骤 5.4：Commit**

```bash
git add apps/desktop/src-tauri/src/main.rs apps/desktop/src-tauri/src/supervisor.rs
git commit -m "feat(desktop): wire orchestration loop, signals, single instance, restart command"
```

---

### 任务 6：dev 冒烟验证与 README

**文件：**
- 创建：`apps/desktop/README.md`
- 修改：根 `package.json`（`scripts` 内紧跟 `"dev:web"` 一行后追加 `"desktop"` 别名）

- [ ] **步骤 6.1：先决条件就位**

```bash
pnpm install
pnpm --filter @deepseek-ai/dsh-web-frontend build
```

预期：dist 生成，无错误。缺 dist 时后端会拒绝启动（见背景事实最后一条）。

- [ ] **步骤 6.2：dev 冒烟**

```bash
pnpm --filter @deepseek-ai/dsh-desktop dev
```

预期逐项记录：
1. 出现"DSH Desktop"原生窗口，初始显示诊断页"正在定位后端命令…"；
2. 数秒内自动切到 Web GUI 界面；
3. 关闭窗口（红绿灯）后终端 dev 进程结束；
4. 另开终端执行 `pgrep -fl "bin.ts.*--profile web"` 预期无残留输出；
5. 运行期间浏览器打开 `http://127.0.0.1:<同端口>` 也能看到同一 GUI（证明 origin 围栏未被绕过）。

若第 4 步有残留：说明 teardown 未生效，回查任务 5 的 Destroyed 分支与 Drop。

- [ ] **步骤 6.3：写 README**

````markdown
# @deepseek-ai/dsh-desktop

DeepSeek Harness 的 Tauri 桌面壳（macOS 优先）：原生窗口内运行现有 Web GUI，由壳探测、拉起并守护本地 `dsh web` 后端。业务界面与启动清单注入始终归 `dsh web` 服务，壳只做进程编排。

## 前置条件

- 本仓库依赖照常安装：`pnpm install`
- 前端产物：`pnpm --filter @deepseek-ai/dsh-web-frontend build`
- Rust 工具链（`rustup` 安装的 stable）

## 日常使用

```bash
pnpm desktop        # 等价于 pnpm --filter @deepseek-ai/dsh-desktop dev
```

后端命令按序探测，首个命中即用：
1. 环境变量 `DSH_DESKTOP_CMD`（完整命令，空格分词，须自带 `--no-open --port 0`）
2. PATH 上的 `dsh`（自动附加 `web --no-open --port 0`）
3. 本仓库源码模式（`node --import tsx/esm` 经 `apps/cli/src/bin.ts web` 启动）

## 配置

| 环境变量 | 默认 | 说明 |
|---|---|---|
| `DSH_DESKTOP_CMD` | 未设置 | 覆盖后端启动命令（完整 argv） |
| `DSH_DESKTOP_READY_TIMEOUT_MS` | `30000` | 就绪等待上限 |

## 打包（未签名，自用）

```bash
pnpm --filter @deepseek-ai/dsh-desktop build
```

产出位于 `apps/desktop/src-tauri/target/release/bundle/{macos,dmg}/`。未签名应用首次打开需右键→打开绕过 Gatekeeper。

## 手动验收清单

1. `pnpm desktop` 起原生窗口，自动进入 Web GUI。
2. 关窗后 `pgrep -fl "bin.ts.*--profile web"` 无残留。
3. 设 `DSH_DESKTOP_CMD=/nonexistent/wrapper` 后窗口显示"无法启动后端进程"，取消该变量并点重启恢复。
4. 杀掉后端进程（`pkill -f "profile web"`）后窗口回到诊断页并可一键重启。

## 单元测试

```bash
pnpm --filter @deepseek-ai/dsh-desktop test   # 如无此脚本则直接：
cargo test --manifest-path src-tauri/Cargo.toml
```

设计规格见 [docs/superpowers/specs](../../docs/superpowers/specs/2026-08-27-tauri-desktop-macos-design.md)。
````

- [ ] **步骤 6.4：追加根脚本别名**

根 `package.json` 的 `scripts` 对象中，紧跟 `"dev:web"` 一行添加：

```json
"desktop": "pnpm --filter @deepseek-ai/dsh-desktop dev",
```

同步验证 JSON 合法：

```bash
node -e "JSON.parse(require('fs').readFileSync('package.json','utf8'))" && echo OK
```

- [ ] **步骤 6.5：Commit**

```bash
git add apps/desktop/README.md package.json
git commit -m "docs(desktop): runtime guide and acceptance checklist; add desktop alias"
```

---

### 任务 7：Agent Note 与收尾验证

**文件：**
- 创建：`.agents/notes/implemented/architecture/2026-08-27-tauri-desktop-shell-managed-backend.md`

- [ ] **步骤 7.1：写 Agent Note**（描述已落地现实，含 why/放弃项/必要验证）

```markdown
# Tauri 桌面壳托管既有 dsh web 后端

桌面客户端以纯编排薄壳落地（`apps/desktop`，Tauri 2）：Rust 侧按
`DSH_DESKTOP_CMD` → PATH `dsh` → 仓库源码模式 的顺序解析后端命令，
以独立进程组拉起 `dsh web --no-open --port 0`，等待地址宣告行
（`dsh web: <url>` 解析 + TCP 探测兜底，超时经
`DSH_DESKTOP_READY_TIMEOUT_MS` 配置）后把 WKWebView 导航到
`http://127.0.0.1:<随机端口>`。stdout 主循环兼作崩溃看门狗并在全程
保持管道排水，stderr 由常驻线程喂同一份环形诊断缓冲。退出路径
（关窗/Cmd+Q/外部信号/SIGTERM）统一走 SIGTERM→SIGKILL 的进程组清理。

选择 WebView 直连回环地址而非自定义协议反代，是因为服务端的 `/api`
浏览器信任围栏按 origin 判定；直连即天然通过，反代则要求放宽围栏并
自复制流式转发。也没有把桌面能力放进 `bundle/web-app`：启动清单注入
与前端服务只留一个归属地。放弃的打包面（签名/公证、Windows/Linux）
记为明确的非目标；探测逻辑保留平台差异接口但首期只在 macOS 实现。

验证：`cargo test --manifest-path apps/desktop/src-tauri/Cargo.toml`
覆盖探测顺序、宣告解析、超时回退、build_command 构造、进程组全组
消亡与幂等 shutdown；手动验收清单在 `apps/desktop/README.md`。
```

- [ ] **步骤 7.2：跑相关检查**

新成员入工作区后跑一次全仓 lint 确认 knip/workspace 约束对新包无告警；有告警按其指示补充 knip.json 或工作区配置白名单条目，不绕过门禁：

```bash
pnpm run lint
pnpm run typecheck
```

预期：typecheck 不受影响（desktop 无 TS 源）；lint 若报 `@deepseek-ai/dsh-desktop` 未用导出/脚本类条目，将其加入 knip 白名单并在提交说明注明。

- [ ] **步骤 7.3：最终提交**

```bash
git add .agents/notes/implemented/architecture/2026-08-27-tauri-desktop-shell-managed-backend.md
git commit -m "docs(agents-note): record managed-backend desktop shell decision"
```

- [ ] **步骤 7.4：打包冒烟（交付形态验收）**

```bash
pnpm --filter @deepseek-ai/dsh-desktop build
ls src-tauri/target/release/bundle/macos/ src-tauri/target/release/bundle/dmg/
open "src-tauri/target/release/bundle/macos/DSH Desktop.app"
```

预期：产出 `.app` 与 `.dmg`；打开 .app 后同样走诊断页→GUI 流程，退出后第 6.2 节第 4 条检查依旧零残留。

---

## 自检结论（计划编写者已执行）

1. **规格覆盖度：** 规格各节对应关系——代码组织→任务 1；探测顺序（含 DSH_DESKTOP_CMD 显式优于隐式）→任务 2；启动与就绪双通道（宣告解析 + TCP 兜底）与超时配置→任务 3+5；退出清理三条路径（关窗/Cmd+Q/信号）收敛同一 teardown→任务 4 Drop/shutdown + 任务 5 Destroyed 分支与信号线程；macOS 适配（单实例聚焦、红绿灯标准窗、标题、targets 只开 app/dmg）→任务 1 conf + 插件；错误处理表三行（探测失败/启动超时/运行中崩溃）→任务 5 的 failed/crashed 事件与诊断页；测试策略→任务 2/3/4 单测、任务 6 验收清单、任务 7 cargo test 全量回归；Agent Note 要求→任务 7。无遗漏。
2. **占位符扫描：** 无 TODO/待定/"类似任务 N"；类型骨架中的 `unimplemented!()` 是 TDD 步骤设计的一部分，每处都在下一步给出完整替换实现。`take_stderr` 已在任务 4 的骨架与实现两步中成对出现，任务 5 直接使用。
3. **类型一致性：** `ProbeOutcome.source_label/program/args/cwd`（任务 2 定义，任务 4 build_command 与任务 5 orchestrate 引用一致）；`ConnectParts.host/port`（任务 3 定义，任务 5 TCP 兜底使用）；`LogTail.new/push/into_vec`（任务 4 定义，任务 5 排水线程与 combined 使用）；`BackendStatus` 字段 `phase/message/attempts/log_tail` 与任务 1 前端读取键一一对应；事件名 `backend-status` 两处相同；command 名 `restart_backend` 在 capabilities-independent 注册与前端 invoke 一致。已核对一致。
