# DeepSeek-TUI 学习笔记

> 这是一份**面向源码学习者**的项目导读。它不是用户手册，也不是 API 参考，而是一位"老师"按"代码教学最佳实践"为你准备的导览：先建立大图，再钻细节，每一节都贴着真实代码讲清楚 *为什么这样写*。

---

## 这份学习笔记的目标读者

- 想理解一个**真实生产级 Rust 终端 Agent** 是如何被组织起来的
- 已经能读 Rust，但没系统地写过 `tokio` 并发 + `mpsc` 事件驱动 + `ratatui` 终端 UI
- 想搞清楚"LLM Agent"具体是怎么把 *用户输入 → 模型流式响应 → 工具调用 → 结果回灌* 串起来的
- 计划 fork 这个项目做改造、加 Provider、扩展工具，或者只是想偷师它的工程范式

如果只想"装上能用"，请直接看仓库根目录的 `@D:\1os\DeepSeek-TUI-tuohai666\README.md` 和 `@D:\1os\DeepSeek-TUI-tuohai666\docs\INSTALL.md`。

---

## 推荐学习路线

按下面的顺序读，**每一篇都建立在前一篇的认知上**，跳着读会反复卡壳：

| # | 文档 | 你将获得的认知 |
|---|---|---|
| 01 | [`01-项目概览.md`](./01-项目概览.md) | 这个项目到底是什么、和"裸调 OpenAI"差在哪、核心设计原则 |
| 02 | [`02-架构与数据流.md`](./02-架构与数据流.md) | 整体分层、一次用户输入会走过哪些组件、控制流 vs 数据流 |
| 03 | [`03-CLI-派发器.md`](./03-CLI-派发器.md) | `deepseek` 二进制如何 fork 出 `deepseek-tui`、为什么要这样拆 |
| 04 | [`04-引擎与轮次循环.md`](./04-引擎与轮次循环.md) | **本项目最关键的一篇**：`Engine` + `turn_loop` 如何驱动一轮对话 |
| 05 | [`05-工具体系.md`](./05-工具体系.md) | `ToolSpec` trait、Registry、审批闸门、并行执行、子代理、RLM |
| 06 | [`06-LLM-客户端.md`](./06-LLM-客户端.md) | OpenAI 兼容流式 SSE、retry、思考模式 (reasoning_content) 的坑 |
| 07 | [`07-TUI-事件循环.md`](./07-TUI-事件循环.md) | ratatui 渲染节奏、事件 → UI 状态、为什么把渲染和模型解耦 |
| 08 | [`08-会话与上下文管理.md`](./08-会话与上下文管理.md) | session、snapshot、capacity controller、compaction、cycle |
| 09 | [`09-实践手册.md`](./09-实践手册.md) | 如何打断点跟一次请求、加新工具、加新 Provider 的最小改动清单 |

> **节奏建议**：01–04 一口气读完（建立主干）；05–08 按需挑读（你关心哪块就先看哪块）；09 在动手改代码时再回来当对照表。

---

## 阅读约定

为了节省你的脑力，本笔记遵循下列约定：

- **代码引用一定带绝对路径 + 行号**：例如 `@D:\1os\DeepSeek-TUI-tuohai666\crates\tui\src\core\engine.rs:299-346`，可以直接在 IDE 里跳过去。
- **不抄长段代码**：只截最能说明问题的几行；其余请你自己跳进文件里读。把 IDE 当成第二屏。
- **每一节末尾都有"自检小练习"**：合上文档，能回答出来就过；答不出来再回来读。
- **优先讲"为什么"**：怎么写在源码里能看，但 *为什么这么拆 / 为什么这里要 clone / 为什么这个 channel 要 unbounded* 的设计意图，只能由人来讲。

---

## 这个项目的"五张关键代码图"

学习中你会反复回到这五张图，先在脑子里挂上：

1. **派发器图**：`crates/cli/src/lib.rs` 解析参数 → 设环境变量 → spawn `crates/tui/src/main.rs`。
2. **引擎心跳图**：`Engine::run` (`@D:\1os\DeepSeek-TUI-tuohai666\crates\tui\src\core\engine.rs:569`) 收 `Op` → 跑 `turn_loop` → 吐 `Event`。
3. **流式串图**：`handle_chat_completion_stream` (`@D:\1os\DeepSeek-TUI-tuohai666\crates\tui\src\client\chat.rs:137`) 把 SSE chunk 解码为 `StreamEvent`。
4. **工具图**：`ToolSpec` trait (`@D:\1os\DeepSeek-TUI-tuohai666\crates\tui\src\tools\spec.rs:548`) 是抽象，`ToolRegistry` 是分发器，`turn_loop` 里的 `plans → outcomes` 是调度器。
5. **TUI 双闭环图**：`run_event_loop` (`@D:\1os\DeepSeek-TUI-tuohai666\crates\tui\src\tui\ui.rs:641`) 一边 `try_recv` 引擎事件、一边 `crossterm::event::poll` 键盘事件。

---

## 一句话定位

> **DeepSeek-TUI 是一个把"模型 ↔ 工具 ↔ 终端 UI"三者用 `mpsc::channel` 串起来的、可中断、可流式、可恢复的 Agent 运行时。**
> 它的工程价值不在于"调通了 DeepSeek API"，而在于把 LLM Agent 落地成一个"可生产"程序所需要的所有边界条件——审批、沙箱、子代理、上下文压缩、崩溃恢复、并发 IO——都用最直接的 Rust 范式实现了一遍。

读完这九篇，你不只是会用它，你应该能自己写一个类似的。

---

*本笔记基于仓库 `feat/...`（study 分支）当时的源码状态生成。后续版本可能调整文件路径，请以代码为准。*
