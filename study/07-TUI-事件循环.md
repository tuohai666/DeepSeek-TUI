# 07 · TUI 事件循环：键盘、引擎事件与 ratatui 渲染

> 这一节讲清楚为什么 UI **必须** 用一个 event loop 同时听键盘和引擎事件，以及作者用了哪些"小动作"让一台便宜笔记本也能跑出 120 FPS 的丝滑流。

---

## 1. UI 的两个并发源

UI 端有 *两个* 异步事件源要听：

```
┌─ crossterm: 键盘/鼠标/Resize/Paste ─────┐
│                                         │
│                                         ├──▶ run_event_loop
│                                         │
└─ engine_handle.rx_event: 流/工具/审批 ──┘
```

它们是 *硬性并发* 的：
- 用户可能正在打字时模型在流；
- 或者模型空闲时用户改字号；
- 或者用户按 `Esc` 中断模型。

UI 的核心挑战就是把这俩源 *公平* 地混进 ratatui 的"画一帧"节奏，**还不能阻塞**。

---

## 2. 主循环的骨架

来自 `@D:\1os\DeepSeek-TUI-tuohai666\crates\tui\src\tui\ui.rs:641-1900+`，去掉枝节后的骨架：

```rust
async fn run_event_loop(...) -> Result<()> {
    let mut frame_rate_limiter = FrameRateLimiter::default();

    loop {
        /* ── 1. 周期性后台任务 ── */
        drain_web_config_events(...).await;
        if last_task_refresh.elapsed() >= 2.5s { refresh_active_task_panel(...).await; }
        frame_rate_limiter.set_low_motion(app.low_motion);

        /* ── 2. 抽干引擎事件（非阻塞） ── */
        {
            let mut rx = engine_handle.rx_event.write().await;
            while let Ok(event) = rx.try_recv() {
                match event { /* 大 match：每种 EngineEvent 改 app 状态 */ }
            }
        }

        /* ── 3. 处理子代理 mailbox / steer ── */
        flush_subagent_inbox(app);
        flush_steer_queue(app, &engine_handle).await;

        /* ── 4. 是否要画一帧？ ── */
        let draw_wait = if app.needs_redraw {
            frame_rate_limiter.time_until_next_draw(Instant::now())
        } else {
            None
        };
        if app.needs_redraw && draw_wait.is_none() {
            terminal.draw(|f| render(f, app))?;
            frame_rate_limiter.mark_emitted(Instant::now());
            app.needs_redraw = false;
        }

        /* ── 5. 决定下一次唤醒时机 ── */
        let mut poll_timeout = if app.is_loading { ACTIVE_MS } else { IDLE_MS };
        if let Some(until_draw) = draw_wait { poll_timeout = poll_timeout.min(until_draw); }
        /* 还有几个其他 deadline 都 min 进去 */

        tokio::task::yield_now().await;

        /* ── 6. 阻塞 poll 键盘（最长 poll_timeout） ── */
        if event::poll(poll_timeout)? {
            let evt = event::read()?;
            app.needs_redraw = true;
            handle_terminal_event(app, evt, ...).await;
        }

        /* ── 7. 处理一些"自然 deadline"（比如把粘贴 burst 提交） ── */
        flush_paste_burst_if_due(app);
    }
}
```

**关键观察**：每次循环迭代做 4 件事：
1. **try_recv** 引擎事件（不阻塞）；
2. **try draw**（被 frame-rate limiter 卡住时不画）；
3. **算出最小 poll_timeout**；
4. **`event::poll(poll_timeout)`** 等键盘——这是循环里**唯一**的阻塞点。

因为 `event::poll` 阻塞最多 `poll_timeout` 毫秒，下一轮 try_recv 就能继续吃引擎事件。这是**手写的"伪 select"**。

---

## 3. 为什么不直接用 `tokio::select!`？

直觉上，把键盘事件和引擎事件 select 在一起更优雅：

```rust
// 看似优雅但会 出大问题
tokio::select! {
    Some(evt) = engine_rx.recv() => handle_engine_event(evt),
    Ok(true) = tokio::spawn_blocking(|| event::poll(...)) => handle_key(...),
}
```

**问题**：
1. `crossterm::event::poll` 是 *同步阻塞* 的，必须 `spawn_blocking` 包装；
2. ratatui 的 `terminal.draw` 也是同步的；
3. tokio 的 select 在 select 分支的 `cancel-safety` 上有约束——你 cancel 一个 `event::poll` 时已经接收的事件可能丢失。

作者选了 *手写顺序* 的循环：每轮 *先 try_recv 全部引擎事件、再 poll 键盘*。这样：
- 引擎事件**永不丢失**（无 cancel 风险）；
- 键盘 poll 即使阻塞，也最多 `poll_timeout` ms 不会饿死引擎事件——下一轮就处理它；
- ratatui 调用都是直接 `?` 不必 `await`。

> **学习要点**：在 ratatui + tokio 这种"半同步半异步"场景下，*手写顺序循环 + 短 poll timeout* 比 `tokio::select!` 更稳。这是个反直觉但正确的工程选择。

---

## 4. `poll_timeout` 的"自适应"

来自 `@D:\1os\DeepSeek-TUI-tuohai666\crates\tui\src\tui\ui.rs:1554-1574`：

```rust
let mut poll_timeout = if app.is_loading || has_running_agents || app.is_compacting {
    Duration::from_millis(active_poll_ms(app))   // ~24ms (40 Hz)
} else {
    Duration::from_millis(idle_poll_ms(app))     // ~48ms (20 Hz)
};
if let Some(until_flush) = app.paste_burst_next_flush_delay_if_enabled(now) {
    poll_timeout = poll_timeout.min(until_flush);
}
if let Some(until_draw) = draw_wait {
    poll_timeout = poll_timeout.min(until_draw);
}
if web_config_session.is_some() {
    poll_timeout = poll_timeout.min(Duration::from_millis(WEB_CONFIG_POLL_MS));   // 16ms
}
if let Some(deadline) = app.quit_armed_until {
    poll_timeout = poll_timeout.min(deadline.saturating_duration_since(now).max(50ms));
}
```

也就是说：**有事就快、没事就慢**。具体规则：

| 情境 | poll 频率 |
|---|---|
| 空闲（无流、无后台代理） | ~20 Hz |
| 引擎活跃（流式 / 工具 / 子代理） | ~40 Hz |
| Web config session 开着（用户在改 settings） | ~60 Hz |
| 即将 frame draw 解锁 | min(到 draw 时刻) |
| 粘贴 burst 即将刷新 | min(到刷新时刻) |
| Quit armed（按一次 Ctrl+C 后的二次确认窗口） | min(到 deadline) |

**这是一种"事件驱动 + 截止时间感知"的混合**——既有事件驱动的"有事就唤醒"，又有 deadline-driven 的"该做的事到点就做"。

> **学习要点**：CPU 占用的"安静"就是在这种细节里磨出来的。空闲时 20 Hz 的循环，加 ratatui 的高效缓冲渲染，能让 deepseek-tui 待机时 CPU 占用 < 1%。

---

## 5. `FrameRateLimiter`：120 FPS 上限

来自 `@D:\1os\DeepSeek-TUI-tuohai666\crates\tui\src\tui\frame_rate_limiter.rs`。它的接口只有两个方法：

```rust
fn time_until_next_draw(&self, now: Instant) -> Option<Duration>
fn mark_emitted(&mut self, when: Instant)
```

**为什么需要它**？因为流式 chunk 来得比眼能看到的还快（DeepSeek 几十 Hz）。如果每个 chunk 都画一帧，CPU 全花在 ratatui buffer 重绘上，看起来反而卡。

策略：
1. **120 FPS 硬上限** —— 8.3ms 内最多画一帧；
2. **`low_motion = true`** 时降到 30 FPS（无障碍 / 录屏时启用）；
3. **draw 后记录 `last_drawn_at`**——下次 `time_until_next_draw` 减出还差多少。

主循环里这一段：

```rust
let draw_wait = if app.needs_redraw {
    frame_rate_limiter.time_until_next_draw(now)
} else {
    None
};
if app.needs_redraw && draw_wait.is_none() {
    terminal.draw(|f| render(f, app))?;
    frame_rate_limiter.mark_emitted(Instant::now());
    app.needs_redraw = false;
}
```

只在 *需要重画 且 已过最小帧间隔* 时才画。否则 `draw_wait` 被塞进 `poll_timeout.min(...)`——下次循环刚好踩到时间点。

---

## 6. 流式渲染：`StreamingState`

来自 `@D:\1os\DeepSeek-TUI-tuohai666\crates\tui\src\tui\streaming\mod.rs:248-300+`。

每个流块（assistant 文本 / thinking）有自己的 `BlockState`，包含：
- **collector**：累计已收的字节
- **chunker**：决定何时把"已收的字"提交进可见 buffer
- **policy**：smooth / typewriter / immediate 几种节奏

挑 chunker 来讲：模型一次发 `"Hello"`，下一次发 `" world"`。如果直接打到屏幕，看起来是"瞬时跳跃"，不像在打字。chunker 会把它**拆成单字符**按节奏吐出来——一帧吐 1-3 字。视觉上就是打字机。

主循环里那一行：

```rust
// @D:\1os\DeepSeek-TUI-tuohai666\crates\tui\src\tui\ui.rs:715-719
app.streaming_state.push_content(0, &sanitized);
let committed = app.streaming_state.commit_text(0);
if !committed.is_empty() {
    append_streaming_text(app, index, &committed);
}
```

`push_content` 喂入 collector，`commit_text` 询问 chunker"现在能吐多少"，得到的字符串才追加到 history cell。

> **学习要点**：流式 UI 的"自然感" 不是来自模型流速，而是来自 *客户端节奏控制*。把"输入速率"和"输出速率"解耦，是高质量打字机效果的关键。

---

## 7. 引擎事件的大 match

`run_event_loop` 步骤 2 那个大 `match`（`@D:\1os\DeepSeek-TUI-tuohai666\crates\tui\src\tui\ui.rs:686-1500`）有约 30 种 `EngineEvent`。我们已经在 02、04 节走过几条，这里抽几条**有学习价值**的：

### 7.1 `EngineEvent::PauseEvents` / `ResumeEvents`

`@D:\1os\DeepSeek-TUI-tuohai666\crates\tui\src\tui\ui.rs:1154-1177`：

```rust
EngineEvent::PauseEvents => {
    if !event_broker.is_paused() {
        pause_terminal(terminal, ...)?;     // 退出 alt-screen / mouse capture
        event_broker.pause_events();
        terminal_paused_at = Some(Instant::now());
    }
}
EngineEvent::ResumeEvents => {
    if event_broker.is_paused() {
        resume_terminal(terminal, ...)?;    // 重进 alt-screen / mouse capture
        event_broker.resume_events();
        terminal_paused_at = None;
    }
}
```

这是给 *外部子进程* 用的——比如模型调 `exec_shell` 跑一个交互式 `vim`，引擎发 `PauseEvents`，UI 让出终端控制，让 vim 直接占用 TTY。vim 退出后引擎发 `ResumeEvents`，UI 收回终端。

> **学习要点**：TUI 程序如果只能"独占终端"，遇到 *它要调用的子程序也是 TUI* 时就会冲突。这个 pause/resume 协议是优雅的解法。

### 7.2 `EngineEvent::AgentSpawned` / `AgentProgress` / `AgentComplete`

子代理事件三连。每条都更新 `app.agent_progress: HashMap<String, String>`，footer 上的"sub-agent X: …"实时变化。

注意 `AgentComplete` 那段 (`@D:\1os\DeepSeek-TUI-tuohai666\crates\tui\src\tui\ui.rs:1203-1253`)：如果**所有** 子代理都完成了，会发 OSC 9 / BEL **桌面通知**——长跑任务结束时让你切回 terminal 的体验加分项。

### 7.3 `EngineEvent::ApprovalRequired`

`@D:\1os\DeepSeek-TUI-tuohai666\crates\tui\src\tui\ui.rs:1272-1330+`：

```rust
EngineEvent::ApprovalRequired { id, tool_name, description, approval_key } => {
    let session_approved = app.approval_session_approved.contains(&approval_key) || ...;
    let session_denied = app.approval_session_denied.contains(&approval_key) || ...;
    if session_denied {
        engine_handle.deny_tool_call(id.clone()).await;
    } else if session_approved || app.approval_mode == ApprovalMode::Auto {
        engine_handle.approve_tool_call(id.clone()).await;
    } else {
        /* 弹审批对话框，等用户按键 */
        app.pending_approval = Some(PendingApproval { id, tool_name, description, approval_key });
        app.view_stack.push(ApprovalView::new(...));
    }
}
```

用户在对话框里选"deny" 时不是只对这一次 deny，**而是把 `approval_key` 记到 `approval_session_denied`**。这样下次同一工具同样参数时**自动拒绝**，避免模型反复 retry 撞墙。

这是把"用户决定"持久化到 session 内的小 trick，体验闭环。

---

## 8. 渲染层：`render(f, app)`

来自 `@D:\1os\DeepSeek-TUI-tuohai666\crates\tui\src\tui\renderer.rs`（或 `ui.rs::render` 函数附近）。

层次：

```
Frame
└── Layout (vertical split)
    ├── HeaderWidget          (model / cwd / token usage)
    ├── ChatWidget            (history cells)
    │   └── 多个 HistoryCell  (User / Assistant / Tool / System)
    ├── ComposerWidget        (input area)
    └── FooterWidget          (status / spinner / cost)
```

每个 widget 实现 ratatui 的 `Widget` trait（或自定义 `Renderable`）。**所有 widget 都是 stateless 的纯渲染函数**——状态全在 `App` 里。

### 8.1 `HistoryCell` 的多形态

每个聊天卡片是一个 `HistoryCell` enum：

```rust
enum HistoryCell {
    User { content: String, /* … */ },
    Assistant { content: String, streaming: bool, thinking: Option<String>, /* … */ },
    Tool(ToolCell),                    // ToolCell 内部又分 Generic / Edit / Shell / Subagent / Fanout 等
    System { content: String },
    /* … */
}
```

为啥 enum 而不是 trait object？因为 cell 数量是小封闭集，match 比 vtable 快、还能 derive `Clone`。

### 8.2 `ChatWidget` 的两次 layout

来自 `@D:\1os\DeepSeek-TUI-tuohai666\crates\tui\src\tui\widgets\mod.rs` (ChatWidget 区块)。
渲染聊天面板要解决一个特别的问题：*要在底部"贴墙"，新内容出现时整个上滚*。

ratatui 默认 widget 是从顶到下画的，所以 ChatWidget 自己**先把所有 cell 转成 Lines**、知道总高度后再决定 *从第 N 行开始画*。当 `total_height < panel_height` 时画到底部对齐；否则配合 `scroll` 状态画该露出的窗口。

这种"二次 layout"在 ratatui 里很常见——先 measure 再 paint。

---

## 9. 输入：`ComposerState` + `App::handle_input_submit`

`ComposerState` (`@D:\1os\DeepSeek-TUI-tuohai666\crates\tui\src\tui\app.rs:514-...`) 持有：

```rust
pub struct ComposerState {
    pub input: String,
    pub cursor: usize,
    pub history: Vec<String>,
    pub history_index: Option<usize>,
    pub menu: Option<ComposerMenu>,        // slash / mention 弹层
    /* … */
}
```

回车键的路径：
1. `crossterm::Event::Key(KeyCode::Enter)` 在 main loop 被识别；
2. `handle_input_submit(app)` 把 `composer.input` 取出来；
3. **判断**：
   - 是 `# foo` 形式 → 走 memory quick-add (`@D:\1os\DeepSeek-TUI-tuohai666\crates\tui\src\tui\ui.rs:516-530`)；
   - 是 `/cmd` 形式 → 进 commands 模块路由；
   - 是普通文本 → 调 `engine_handle.send(Op::SendMessage{ ... })`；
4. push 到 `composer.history` 让 ↑ 能重发；
5. 清空 input。

斜杠命令 `/help` 等的解析路径在 `crates/tui/src/commands/`（30+ 个文件），命令注册表在 `commands/mod.rs::COMMANDS`，每个返回 `CommandResult`，再被 UI 层翻译成 `AppAction` 处理。

---

## 10. `EngineHandle`：UI 端的 RPC stub

`@D:\1os\DeepSeek-TUI-tuohai666\crates\tui\src\core\engine_handle.rs`：

```rust
pub struct EngineHandle {
    pub tx_op: mpsc::Sender<Op>,
    pub rx_event: Arc<RwLock<mpsc::Receiver<Event>>>,
    pub tx_approval: mpsc::Sender<ApprovalDecision>,
    pub tx_user_input: mpsc::Sender<UserInputDecision>,
    pub tx_steer: mpsc::Sender<String>,
    pub shared_cancel_token: Arc<StdMutex<CancellationToken>>,
}

impl EngineHandle {
    pub async fn send(&self, op: Op) -> Result<()> { self.tx_op.send(op).await }
    pub async fn approve_tool_call(&self, id: String) -> Result<()> { /* … */ }
    pub async fn deny_tool_call(&self, id: String) -> Result<()> { /* … */ }
    pub fn cancel(&self) { self.shared_cancel_token.lock().unwrap().cancel(); }
    pub async fn steer(&self, msg: String) -> Result<()> { /* … */ }
}
```

**它是 UI 端 *唯一*持有的引擎接口**。把所有引擎 channel 打包成 `Arc`-clonable 对象——传给 view、widget、命令处理器都行，无锁竞争。

> **学习要点**：把多 channel 打成一个 Handle 类型 —— 这是 actor 系统的标配模式。`tokio::select!` 多 channel 时只用 1 行 `handle.send(op).await`，整个 UI 才不会陷入 channel argument soup。

---

## 11. `commands` 模块：30+ 文件的小宇宙

`@D:\1os\DeepSeek-TUI-tuohai666\crates\tui\src\commands/`：

```
commands/
├─ mod.rs            (注册表 + CommandResult + 派发器)
├─ core.rs           (/help, /clear, /exit, /model, /models, /subagents 等)
├─ session.rs        (/session, /resume)
├─ goal.rs           (/goal)
├─ jobs.rs           (/jobs)
├─ memory.rs         (/memory)
├─ network.rs        (/network)
├─ note.rs           (/note)
├─ provider.rs       (/provider)
├─ queue.rs          (/queue)
├─ rename.rs         (/rename)
├─ restore.rs        (/restore)
├─ review.rs         (/review)
├─ skills.rs         (/skill)
├─ stash.rs          (/stash)
├─ task.rs           (/task)
├─ user_commands.rs  (用户自定义命令)
├─ … 还有更多
```

每个文件对应一个或几个 slash command。模板很统一（看 `commands/core.rs:13-44`）：

```rust
pub fn help(app: &mut App, topic: Option<&str>) -> CommandResult {
    /* … */
    CommandResult::message("...".to_string())   // 或 CommandResult::action(AppAction::xxx)
}
```

`CommandResult` 是 message + 可选 action 的二元体。**Action 是 UI 与 Engine 之间的中介**——比如 `AppAction::SyncSession{ … }` 会被 UI 层翻译成 `Op::SyncSession` 发给引擎。

这种"单文件单命令"的拆分让加新命令成本极低：写一个文件 + 在 mod.rs 注册一行。

---

## 12. 你修这一层时该改哪儿

| 你想改的事 | 改哪个文件 |
|---|---|
| 加新斜杠命令 | 在 `commands/` 下新建一个文件 + `commands/mod.rs::COMMANDS` 加一行 |
| 改聊天卡片的样式 | `tui/widgets/` 对应 cell 的 render 函数 |
| 改打字机节奏 | `tui/streaming/policy.rs` 的 `Smooth` / `Typewriter` 参数 |
| 改帧率上限 | `tui/frame_rate_limiter.rs` 的常量 |
| 加新 EngineEvent 处理 | `tui/ui.rs::run_event_loop` 的大 match |
| 改空闲/活跃 poll 频率 | `tui/ui.rs::idle_poll_ms` / `active_poll_ms` |
| 改主题色 | `tui/palette.rs` |
| 加全屏模态 | `tui/views/` 新增一个 View，view_stack push |

---

## 13. 自检小练习

1. 为什么不用 `tokio::select!` 把键盘和引擎事件混到一起？给出至少两个理由。
2. `frame_rate_limiter` 的 `time_until_next_draw` 返回 `Option<Duration>`——什么时候是 `None`？什么时候是 `Some`？
3. 流式打字时，`StreamingState` 在 collector / chunker / policy 三层里干什么？把 push_content / commit_text 的"流量传送"画一张图。
4. 用户按 Ctrl+C 中断模型时，UI 端调的是 `engine_handle.cancel()`，里面拿了一个 `StdMutex` 锁——为什么不是 `tokio::sync::Mutex`？请用一句话说清。
5. 接到 `EngineEvent::PauseEvents` 时 UI 必须 *退出 alt-screen + 关 mouse capture*。如果只关 alt-screen 不关 mouse capture 会出什么糗事？

---

**下一节**：[`08-会话与上下文管理.md`](./08-会话与上下文管理.md) 我们看一个 turn 之外的事情——session 如何持久化、checkpoint 如何救命、capacity 如何防爆 token、cycle 如何切片长任务。
