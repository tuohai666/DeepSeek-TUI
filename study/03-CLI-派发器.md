# 03 · CLI 派发器：deepseek 是怎么找到并启动 deepseek-tui 的

> 派发器（dispatcher）是项目最简单的 crate，但它的几个设计决定影响整个项目体验。本节带你**逐函数**读 `crates/cli`。
> 别小看这一节——读完你会理解 *"派发器→主进程"模式* 在 Rust CLI 里能怎么落地，工具链/CI 改造时这是必修课。

---

## 1. 文件全景

```
crates/cli/
├─ Cargo.toml
└─ src/
   ├─ main.rs       (4 行)
   ├─ lib.rs        (~2500 行，包含 #[cfg(test)] 单测)
   ├─ metrics.rs    (一个 deepseek metrics 子命令的实现)
   └─ update.rs     (deepseek update 命令)
```

`main.rs` 极简（`@D:\1os\DeepSeek-TUI-tuohai666\crates\cli\src\main.rs:1-4`）：

```rust
fn main() -> std::process::ExitCode {
    deepseek_tui_cli::run_cli()
}
```

**为什么把全部逻辑搬到 lib.rs**？因为单测要测 `Cli::parse()` 行为，必须能在 *库* 上挂 `#[cfg(test)]`，binary crate 不太方便。

---

## 2. clap 命令树

派发器用 `clap` 的派生宏定义命令树，`@D:\1os\DeepSeek-TUI-tuohai666\crates\cli\src\lib.rs:53-193`：

```rust
#[derive(Debug, Parser)]
#[command(name = "deepseek", version, bin_name = "deepseek", …)]
struct Cli {
    #[arg(long)] config: Option<PathBuf>,
    #[arg(long)] profile: Option<String>,
    #[arg(long)] provider: Option<ProviderArg>,
    #[arg(long)] model: Option<String>,
    /* 还有十几个全局开关 */
    #[arg(short = 'p', long = "prompt")] prompt_flag: Option<String>,
    #[arg(value_name = "PROMPT")] prompt: Option<String>,
    #[command(subcommand)] command: Option<Commands>,
}
```

最关键的是有 **两个 prompt 字段**：
- `prompt_flag` 是 `-p "你的问题"` 的形式；
- `prompt` 是位置参数 `deepseek "你的问题"`。

`#[arg(conflicts_with = "prompt")]` 保证两者只能给一个。这是为了同时支持两种用户习惯：UNIX 风（`-p`）和 Claude/Cursor 风（裸字符串）。

子命令枚举 `Commands` (`@D:\1os\DeepSeek-TUI-tuohai666\crates\cli\src\lib.rs:108-193`) 大致分三类：

| 类别 | 命令 | 行为 |
|---|---|---|
| **直通命令** | `Doctor`, `Models`, `Sessions`, `Resume`, `Fork`, `Setup`, `Exec`, `Review`, `Apply`, `Eval`, `Mcp`, `Features`, `Serve`, `Completions` | 派发器原样转发给 `deepseek-tui` |
| **派发器自营** | `Login`, `Logout`, `Auth`, `Config`, `Model`, `Thread`, `Sandbox`, `AppServer`, `Completion`, `Metrics`, `Update`, `McpServer` | 不需要 TUI 渲染，派发器自己跑完返回 |
| **隐式默认** | （无子命令时） | 进交互 TUI 模式 |

直通命令的标记是 `TuiPassthroughArgs(trailing_var_arg = true, allow_hyphen_values = true)`——把整段子参数原样收进 `Vec<String>`，再原样喂给伴生二进制。

---

## 3. 主入口 run()

来源：`@D:\1os\DeepSeek-TUI-tuohai666\crates\cli\src\lib.rs:394-523`。

```rust
pub fn run_cli() -> std::process::ExitCode {
    match run() {
        Ok(()) => std::process::ExitCode::SUCCESS,
        Err(err) => {
            eprintln!("error: {err}");
            for cause in err.chain().skip(1) {
                eprintln!("  caused by: {cause}");
            }
            std::process::ExitCode::FAILURE
        }
    }
}
```

**注意 anyhow chain 的展开**——这是 issue #767 的修复：默认 `Display` 只打顶层 context。如果你 `eprintln!("error: {err}")` 一行的话，TOML 解析失败时用户只会看到 `failed to parse config at /path/to.toml`，看不到底层 `TOML parse error at line X column Y`，体验非常糟。

来到 `run()`：

```rust
fn run() -> Result<()> {
    let mut cli = Cli::parse();
    let mut store = ConfigStore::load(cli.config.clone())?;
    let runtime_overrides = CliRuntimeOverrides {
        provider: cli.provider.map(Into::into),
        model: cli.model.clone(),
        api_key: cli.api_key.clone(),
        base_url: cli.base_url.clone(),
        /* … */
    };
    let command = cli.command.take();
    match command { /* 路由 */ }
}
```

每条路由前都要 `resolve_runtime_for_dispatch(&mut store, &runtime_overrides)`：把 *配置文件 + 环境变量 + CLI 覆盖* 三层叠加成一个 `ResolvedRuntimeOptions`，这就是要传给 TUI 的"权威配置"。

---

## 4. 核心函数：`delegate_to_tui`

派发器最重要的一段代码。逐行读 `@D:\1os\DeepSeek-TUI-tuohai666\crates\cli\src\lib.rs:1203-1214`：

```rust
fn delegate_to_tui(
    cli: &Cli,
    resolved_runtime: &ResolvedRuntimeOptions,
    passthrough: Vec<String>,
) -> Result<()> {
    let mut cmd = build_tui_command(cli, resolved_runtime, passthrough)?;
    let tui = PathBuf::from(cmd.get_program());
    let status = cmd
        .status()
        .map_err(|err| anyhow!("{}", tui_spawn_error(&tui, &err)))?;
    exit_with_tui_status(status)
}
```

四步：
1. 构造 `Command`；
2. spawn 子进程（`status()` 阻塞等它退出）；
3. 失败时给一段**带恢复建议**的错误（不是裸 `IOError`）；
4. 用子进程的退出码作为自己的退出码。

### 4.1 `build_tui_command` 做了什么

来自 `@D:\1os\DeepSeek-TUI-tuohai666\crates\cli\src\lib.rs:1267-1360`。

第一步定位伴生二进制：

```rust
let tui = locate_sibling_tui_binary()?;
```

第二步把全局开关原样加到 args：

```rust
let mut cmd = Command::new(&tui);
if let Some(config) = cli.config.as_ref() {
    cmd.arg("--config").arg(config);
}
if let Some(profile) = cli.profile.as_ref() {
    cmd.arg("--profile").arg(profile);
}
if cli.no_alt_screen { cmd.arg("--no-alt-screen"); }
/* ……一堆 */
cmd.args(passthrough);
```

第三步**把解析好的运行时配置塞到环境变量**，让 TUI 进程不必重复解析：

```rust
cmd.env("DEEPSEEK_MODEL", &resolved_runtime.model);
cmd.env("DEEPSEEK_BASE_URL", &resolved_runtime.base_url);
cmd.env("DEEPSEEK_PROVIDER", resolved_runtime.provider.as_str());
if !resolved_runtime.http_headers.is_empty() {
    let encoded = resolved_runtime.http_headers.iter()
        .map(|(name, value)| format!("{}={}", name.trim(), value.trim()))
        .collect::<Vec<_>>()
        .join(",");
    cmd.env("DEEPSEEK_HTTP_HEADERS", encoded);
}
if let Some(api_key) = resolved_runtime.api_key.as_ref() {
    cmd.env("DEEPSEEK_API_KEY", api_key);
    let source = resolved_runtime.api_key_source
        .unwrap_or(RuntimeApiKeySource::Env)
        .as_env_value();
    cmd.env("DEEPSEEK_API_KEY_SOURCE", source);
}
```

> **学习要点**：环境变量是 *跨进程* 的天然胶水。这种"把配置打包成环境变量传子进程"的写法，比让子进程重新读一遍配置文件快、也更安全（不用担心配置文件中途被改）。

不支持的 Provider 直接 `bail!` 阻塞（如 `deepseek --provider openrouter` 还没把 OpenRouter 的 TUI 路线跑通时）。

### 4.2 `locate_sibling_tui_binary` 的查找顺序

来自 `@D:\1os\DeepSeek-TUI-tuohai666\crates\cli\src\lib.rs:1407-1462`。

```rust
fn locate_sibling_tui_binary() -> Result<PathBuf> {
    if let Ok(override_path) = std::env::var("DEEPSEEK_TUI_BIN") {
        let candidate = PathBuf::from(override_path);
        if candidate.is_file() {
            return Ok(candidate);
        }
        bail!("DEEPSEEK_TUI_BIN points at {}, which is not a regular file.", candidate.display());
    }
    let current = std::env::current_exe().context("failed to locate current executable path")?;
    if let Some(found) = sibling_tui_candidate(&current) {
        return Ok(found);
    }
    /* 大段错误提示 */
    bail!(/* … */);
}
```

查找优先级：
1. **`DEEPSEEK_TUI_BIN` 环境变量**（最高优先级，给 CI / 异构安装用）；
2. **同目录的 `deepseek-tui` / `deepseek-tui.exe`**（默认）。

为什么要 Windows 兜底找 *无后缀* 文件？issue #247：早期 npm 包错把 `.exe` 后缀脱了，用户改名为 `deepseek-tui`。修复后保留兜底逻辑兼容老安装：

```rust
fn sibling_tui_candidate(dispatcher: &Path) -> Option<PathBuf> {
    let primary = dispatcher.with_file_name(format!("deepseek-tui{}", std::env::consts::EXE_SUFFIX));
    if primary.is_file() {
        return Some(primary);
    }
    if cfg!(windows) {
        let suffixless = dispatcher.with_file_name("deepseek-tui");
        if suffixless.is_file() {
            return Some(suffixless);
        }
    }
    None
}
```

> **学习要点**：把"找伴生二进制"这种**纯函数逻辑** 抽成 `sibling_tui_candidate` 而不是直接长在 `locate_sibling_tui_binary` 里——为了能写单元测试（`@D:\1os\DeepSeek-TUI-tuohai666\crates\cli\src\lib.rs:2415-2464` 三个测试）。这是工程学的一个重要范式：**纯函数边界 = 测试边界**。

### 4.3 错误信息也是产品

`tui_spawn_error` (`@D:\1os\DeepSeek-TUI-tuohai666\crates\cli\src\lib.rs:1381-1396`) 构造的失败提示长这样：

```
failed to spawn companion TUI binary at C:\tools\deepseek-tui.exe: access is denied

The `deepseek` dispatcher found a `deepseek-tui` file, but the OS refused
to execute it. Common fixes:
  - Reinstall with `npm install -g deepseek-tui`, or run `deepseek update`.
  - On Windows, run `where deepseek` and `where deepseek-tui`; both should
    come from the same install directory.
  - If you downloaded release assets manually, keep both `deepseek` and
    `deepseek-tui` binaries together and make sure the TUI binary is executable.
  - Set DEEPSEEK_TUI_BIN to the absolute path of a working `deepseek-tui` binary.
```

不是简单 `Err(io::Error)`，而是把"发生了什么 / 该怎么排查 / 三种修复路径"都写进去了。这是工程师对用户的尊重。

---

## 5. 派发器自营命令

不是所有命令都需要 TUI。比如 `auth set`、`config set` 直接读写 `~/.deepseek/config.toml`，不需要起渲染线程。这些路径走"派发器自营":

### 5.1 `auth` 命令：API Key 多层存储

来自 `@D:\1os\DeepSeek-TUI-tuohai666\crates\cli\src\lib.rs:760-1100`（`run_auth_command_with_secrets`）。

API Key 有三层来源，**优先级从高到低**：

```rust
// @D:\1os\DeepSeek-TUI-tuohai666\crates\cli\src\lib.rs:773
println!("credential precedence: config -> keyring -> env");
```

1. **配置文件**：`~/.deepseek/config.toml` 里 `[providers.deepseek] api_key = "sk-..."`
2. **OS Keyring**：macOS Keychain / Linux Secret Service / Windows Credential Manager
3. **环境变量**：`DEEPSEEK_API_KEY`

**这是为什么**？
- 配置文件高优——便于 *跨终端共用* 同一个 key（你打开 10 个 terminal tab 不必每次输 key）。
- Keyring 中等优——**安全存储**的标准做法，但跨终端用不友好（需要 OS 提示）。
- 环境变量最低优——CI 友好，但容易被 `.zshrc` 旧 export 卡住（issue #767 的另一面）。

精彩的是 `dispatch_keyring_recovery_self_heals_into_config_file` (`@D:\1os\DeepSeek-TUI-tuohai666\crates\cli\src\lib.rs:2108-2155`) 测试演示的"自愈"行为：如果只在 keyring 里有 key、配置文件里没有，派发器会自动把它复制进配置文件，*下次启动就不用再问 OS keyring*。

```rust
if resolved.api_key_source == Some(RuntimeApiKeySource::Keyring)
    && !provider_config_set(store, resolved.provider)
    && let Some(api_key) = resolved.api_key.clone()
{
    write_provider_api_key_to_config(store, resolved.provider, &api_key);
    /* save & inform user */
}
```

> **学习要点**：在派发器层做这种"配置自愈"，不污染 TUI 进程的逻辑。一个进程死亡时另一个进程能拿到稳定状态——这是为什么"两段式 binary"在工程上是有红利的。

### 5.2 Windows-specific resume picker

`@D:\1os\DeepSeek-TUI-tuohai666\crates\cli\src\lib.rs:1216-1265`：

```rust
fn run_resume_command(...) {
    let passthrough = tui_args("resume", args);
    if should_pick_resume_in_dispatcher(&passthrough, cfg!(windows)) {
        return run_dispatcher_resume_picker(cli, resolved_runtime);
    }
    delegate_to_tui(cli, resolved_runtime, passthrough)
}
```

**为什么 Windows 要特殊处理**？因为旧 Windows 终端在 raw mode 下交互式选择体验很糟。派发器先用普通 stdin 列出会话让用户选编号，再带着选定 ID delegate。

这就是"派发器作为 *预处理层*"的价值：可以做一些 TUI 不擅长的事。

---

## 6. 派发器与 TUI 的"接口契约"

派发器把下面这些环境变量当成"私有 ABI"传给 TUI：

| 变量 | 含义 |
|---|---|
| `DEEPSEEK_API_KEY` | API Key 明文（会被 TUI 在错误日志里 redact） |
| `DEEPSEEK_API_KEY_SOURCE` | `config` / `keyring` / `env` / `cli` —— 用于诊断错误时告诉用户从哪儿读的 |
| `DEEPSEEK_BASE_URL` | API 端点 |
| `DEEPSEEK_PROVIDER` | provider 字符串 |
| `DEEPSEEK_MODEL` | 默认模型 |
| `DEEPSEEK_HTTP_HEADERS` | `name1=val1,name2=val2` 编码的额外 header |
| `DEEPSEEK_OUTPUT_MODE` | `text` / `json` |
| `DEEPSEEK_LOG_LEVEL` | tracing 日志级别 |
| `DEEPSEEK_TELEMETRY` | bool |
| `DEEPSEEK_APPROVAL_POLICY` | 审批策略 |
| `DEEPSEEK_SANDBOX_MODE` | 沙箱模式 |
| `DEEPSEEK_TUI_BIN` | 测试/CI 时覆盖伴生二进制路径 |

> **学习要点**：环境变量在这里**充当 IPC 协议**。这种契约一旦定下来，两个 binary 就可以独立演化，只要环境变量名/语义不变。这跟"前后端用 OpenAPI Schema 协约"一回事。

---

## 7. 隐藏命令：`mcp-server`

派发器还有一个直营路径 `Commands::McpServer` (`@D:\1os\DeepSeek-TUI-tuohai666\crates\cli\src\lib.rs:148-149`)：

```rust
/// Run MCP server mode over stdio.
McpServer,
```

它会让 `deepseek` 进程**自己变成一个 MCP 服务器**（通过 stdio JSON-RPC 暴露内置工具），让其他 LLM 客户端把它当成工具源。这是 MCP 生态的"反向"用法。

具体实现走 `run_mcp_server_command(&mut store)`，从 `[mcp.server_definitions]` 配置加载工具定义并起 stdio server。学习时可以略过这条路径——它是给"DeepSeek 是被调用的工具，不是调用方"那种场景用的。

---

## 8. 派发器视角下的"启动流"

把全部串起来，命令 `deepseek -p "hello"` 的完整流程：

```
1. main.rs           → run_cli() → run()
2. lib.rs:run        → Cli::parse() 解析参数
3. lib.rs:run        → ConfigStore::load() 读 ~/.deepseek/config.toml
4. lib.rs:run        → resolve_runtime_for_dispatch() 三层叠加配置
5. lib.rs:run        → match command 落到 None 分支（无子命令）
6.                   → forwarded.push("--prompt"); forwarded.push("hello");
7. delegate_to_tui   → build_tui_command()
8. build_tui_command → locate_sibling_tui_binary() 找 deepseek-tui[.exe]
9. build_tui_command → cmd.env(...) 注入 12 个环境变量
10. delegate_to_tui  → cmd.status() 起子进程，阻塞等
11.                  ┌── deepseek-tui 进程 ──────────────┐
                     │ #[tokio::main] async fn main()    │
                     │ → 检测 DEEPSEEK_API_KEY 等环境量  │
                     │ → 进 run_one_shot 或 run_interactive │
                     │ → 退出，返回退出码                │
                     └────────────────────────────────────┘
12. exit_with_tui_status → std::process::exit(子进程退出码)
```

整个派发器进程的生命周期通常**< 50 ms**（不算子进程）。它是一个真正的 launcher。

---

## 9. 自检小练习

1. 为什么把 `Cli` 解析、`fn run()`、`tests` 全放在 `lib.rs` 而不是 `main.rs`？
2. `DEEPSEEK_TUI_BIN` 环境变量在什么场景下必须用？为什么不直接 hardcode 路径？
3. `auth migrate` 命令在 `dry_run = true` 时应该具体做什么、不做什么？尝试在源码里找到对应的测试。
4. 你想新增一个子命令 `deepseek dump-prompt`，让它打印出当前 mode 的系统 prompt。它应该是"派发器直营"还是"TUI 直通"？请给出推理。
5. `Persistence Actor` 在 TUI 进程里。如果派发器进程崩溃了（比如 `cmd.status()` 失败），TUI 子进程 *会立刻退出* 吗？为什么？

---

**下一节**：[`04-引擎与轮次循环.md`](./04-引擎与轮次循环.md)——本系列**最关键的一节**。我们要带你从 `Engine::run` 开始，逐行读懂"一轮对话"是如何在 90% 的工程难题里穿行的。
