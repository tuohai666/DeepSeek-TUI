# 06 · LLM 客户端：把 SSE 拼成 StreamEvent

> 引擎与外部世界之间的"传输层"。本节聚焦三件事：**HTTP/SSE 流式解码、重试与健康度、thinking-mode 的反复打补丁**。
> 这一节的代码读起来一开始有点琐碎，但每一段琐碎背后都有真实事故——读完你能感受到一个生产 LLM 客户端要扛多少边角。

---

## 1. 模块地图

```
crates/tui/src/
├─ client.rs                  ← DeepSeekClient struct + 重试 + 健康度
├─ client/chat.rs             ← Chat Completions 路径（流式 + 非流式）
├─ llm_client.rs              ← LlmClient trait（抽象 Provider）
├─ models.rs                  ← MessageRequest / MessageResponse / ContentBlock 等数据型
├─ retry_status.rs            ← UI 顶部"retrying… (3/5)"提示状态
└─ retry/                     ← 通用 with_retry helper
```

**两个层次**：
1. **trait `LlmClient`** —— 抽象层，让引擎不关心具体是哪家 API；
2. **`DeepSeekClient`** —— 实现层，OpenAI 兼容的 Chat Completions 客户端，能跑 DeepSeek / NVIDIA NIM / Fireworks / SGLang / vLLM / Ollama / OpenRouter。

```rust
// @D:\1os\DeepSeek-TUI-tuohai666\crates\tui\src\client.rs:701-741
impl LlmClient for DeepSeekClient {
    fn provider_name(&self) -> &'static str { /* … */ }
    fn model(&self) -> &str { /* … */ }
    async fn health_check(&self) -> Result<bool> { /* … */ }
    async fn create_message(&self, request: MessageRequest) -> Result<MessageResponse> {
        self.create_message_chat(&request).await
    }
    async fn create_message_stream(&self, request: MessageRequest) -> Result<StreamEventBox> {
        self.handle_chat_completion_stream(request).await
    }
}
```

---

## 2. `DeepSeekClient` 的构造

来自 `@D:\1os\DeepSeek-TUI-tuohai666\crates\tui\src\client.rs:450-484`：

```rust
pub fn new(config: &Config) -> Result<Self> {
    let api_key = config.deepseek_api_key()?;
    let base_url = config.deepseek_base_url();
    let api_provider = config.api_provider();
    validate_base_url_security(&base_url)?;
    let retry = config.retry_policy();
    let default_model = config.default_model();
    let http_headers = config.http_headers();

    let http_client = Self::build_http_client(&api_key, &http_headers)?;

    Ok(Self {
        http_client,
        api_key,
        base_url,
        api_provider,
        retry,
        default_model,
        connection_health: Arc::new(AsyncMutex::new(ConnectionHealth::default())),
        rate_limiter: Arc::new(AsyncMutex::new(TokenBucket::from_env())),
    })
}
```

挑两点细节：

### 2.1 `validate_base_url_security`

不允许 `http://` 而非 `https://`（除非显式 localhost 调试）；不允许私网地址（防 SSRF）。这是基础卫生。

### 2.2 `connection_health` + `rate_limiter`

两个 `Arc<AsyncMutex<…>>`：
- **`ConnectionHealth`** —— 滑动窗口里数失败次数，连续失败到阈值时进 *degraded* 状态，下一次成功调用前会跑 `maybe_probe_recovery` 探测恢复。
- **`TokenBucket`** —— 客户端侧的 rate limiter，`from_env()` 读 `DEEPSEEK_RATE_LIMIT_RPS` 等。每次请求前 `wait_for_rate_limit()` 阻塞，避免触发服务端 429。

> **学习要点**：客户端侧主动 rate-limit 比"被服务端 429 后重试"更友好——避免抖动尖峰。这是 production 客户端的标志特征。

---

## 3. HTTP 客户端的"特殊化"

来自 `@D:\1os\DeepSeek-TUI-tuohai666\crates\tui\src\client.rs:486-508`：

```rust
fn build_http_client(api_key: &str, extra_headers: &HashMap<String, String>) -> Result<reqwest::Client> {
    let headers = build_default_headers(api_key, extra_headers)?;
    let mut builder = reqwest::Client::builder()
        .default_headers(headers)
        .connect_timeout(Duration::from_secs(30))
        .tcp_keepalive(Some(Duration::from_secs(30)))
        .http2_keep_alive_interval(Some(Duration::from_secs(15)))
        .http2_keep_alive_timeout(Duration::from_secs(20))
        .min_tls_version(reqwest::tls::Version::TLS_1_2);
    if force_http1_from_env() {
        builder = builder.http1_only();
    }
    if let Ok(cert_path) = std::env::var("SSL_CERT_FILE")
        && !cert_path.is_empty()
    {
        builder = add_extra_root_certs(builder, &cert_path);
    }
    builder.build().map_err(Into::into)
}
```

每一行都是一次教训：

| 配置 | 抗的什么坑 |
|---|---|
| `connect_timeout(30s)` | 公司内网代理慢，避免无限挂起 |
| `tcp_keepalive(30s)` | 一些 NAT 会在 60s 内回收无活动连接，keepalive 让它存活 |
| `http2_keep_alive_interval/timeout` | HTTP/2 上 SSE 长流容易被 idle middlebox 砍，定期 PING |
| `min_tls_version(TLS_1_2)` | 拒绝过时 TLS，issue #67 |
| `force_http1_from_env` | 个别 corporate 代理把 HTTP/2 解析坏，给用户一把 escape hatch |
| `SSL_CERT_FILE` | 自签 CA 场景（公司 MITM 网关） |

---

## 4. SSE 流式解码：`handle_chat_completion_stream`

这是项目最长、最关键的几百行之一，来自 `@D:\1os\DeepSeek-TUI-tuohai666\crates\tui\src\client\chat.rs:137-413`。

### 4.1 请求构造

把 `MessageRequest` 翻成 OpenAI Chat Completions 形态（`@D:\1os\DeepSeek-TUI-tuohai666\crates\tui\src\client\chat.rs:142-185`）：

```rust
let messages = build_chat_messages_for_request(&request);
let mut body = json!({
    "model": request.model,
    "messages": messages,
    "max_tokens": request.max_tokens,
    "stream": true,
    "stream_options": { "include_usage": true },
});

if let Some(tools) = request.tools.as_ref() {
    body["tools"] = json!(tools.iter().map(tool_to_chat).collect::<Vec<_>>());
}
apply_reasoning_effort(&mut body, request.reasoning_effort.as_deref(), self.api_provider);

let replay_input_tokens = sanitize_thinking_mode_messages(
    &mut body, &request.model, request.reasoning_effort.as_deref()
);
```

注意 `stream_options.include_usage: true` —— DeepSeek 的 OpenAI 兼容路径默认 *不返回 usage*，必须显式开。**没开就拿不到 token 计数，就没法做 capacity 控制**。

### 4.2 流的生产者：`async_stream::stream!`

来自 `@D:\1os\DeepSeek-TUI-tuohai666\crates\tui\src\client\chat.rs:213-413`：

```rust
let stream = async_stream::stream! {
    yield Ok(StreamEvent::MessageStart { message: /* synthetic */ });

    let mut line_buf = String::new();
    let mut byte_buf = acquire_stream_buffer();
    /* ... */
    let mut byte_stream = std::pin::pin!(byte_stream);
    let idle = stream_idle_timeout();

    loop {
        let chunk_result = match tokio_timeout(idle, byte_stream.next()).await {
            Ok(Some(result)) => result,
            Ok(None) => break,
            Err(_elapsed) => {
                yield Err(anyhow::anyhow!("SSE stream idle timeout after {}s", idle.as_secs()));
                break;
            }
        };
        let chunk = match chunk_result {
            Ok(bytes) => bytes,
            Err(e) => { /* 链式打错误源 + yield Err */ break; }
        };

        byte_buf.extend_from_slice(&chunk);
        // 防爆缓冲（10MB 上限）
        if byte_buf.len() > MAX_SSE_BUF { yield Err(...); break; }
        // 背压（高水位 sleep 一会儿）
        if byte_buf.len() > SSE_BACKPRESSURE_HIGH_WATERMARK { tokio::time::sleep(...).await; }

        // 按 \n 切行，按 SSE 协议处理
        while let Some(newline_pos) = byte_buf.iter().position(|&b| b == b'\n') {
            let line = ...;
            byte_buf.drain(..newline_pos + 1);

            if line.is_empty() {
                // 空行 = event 边界
                if !line_buf.is_empty() {
                    let data = std::mem::take(&mut line_buf);
                    if data.trim() == "[DONE]" { /* 完事 */ }
                    else if let Ok(chunk_json) = serde_json::from_str::<Value>(&data) {
                        for event in parse_sse_chunk(&chunk_json, &mut content_index, ...) {
                            yield Ok(event);
                        }
                    }
                }
            } else if let Some(rest) = line.strip_prefix("data:") {
                line_buf.push_str(rest.trim_start());
            }
            /* 忽略其他 SSE 字段：event:, id:, retry: */
        }
    }
};
```

骨架就是 **"按 \n 切行 → 累积到 line_buf → 遇到空行就 parse → 喷出 StreamEvent"**。`async_stream::stream!` 宏让你能在 async 函数里 `yield`，省掉手写 `Stream` impl 的样板。

### 4.3 五个真实事故的烙印

读这段代码时注意以下五个细节，每个都对应一次故障排查：

| 行 | 设计 | 防的什么坑 |
|---|---|---|
| `tokio_timeout(idle, byte_stream.next())` | idle timeout（默认 60s 无字节） | 服务端死链没发 RST，连接一直挂 |
| `MAX_SSE_BUF = 10 MB` | 缓冲上限 | 故意发畸形流来 OOM 你 |
| `SSE_BACKPRESSURE_HIGH_WATERMARK` | 大缓冲时 sleep 让消费者追上 | UI 绘帧慢导致 buffer 堆积 |
| `error_chain` 走 `Error::source` 链 | 链式打错误 | reqwest 默认 `Display` 只说 "error decoding response body"，不告诉你是 HTTP/2 RST 还是 gzip 损坏 |
| `format_stream_headers` 记录响应头 | 错误日志里写头 | 区分 chunked / brotli / hpack 异常 |

> **学习要点**：高质量的"日志可诊断性"是生产级流式客户端的护城河。把 *failure 时刻的所有可观察上下文* 一并记下来，一年里那一两次诡异 bug 才能定位。

### 4.4 `parse_sse_chunk` —— DeepSeek 流的反序列化

来自 `@D:\1os\DeepSeek-TUI-tuohai666\crates\tui\src\client\chat.rs:1100-1300+`。它把一条 OpenAI 格式的 chunk JSON 翻译成 0..N 个 `StreamEvent`。

OpenAI 的 chunk 长这样：

```json
{
  "choices": [{
    "index": 0,
    "delta": {
      "content": "Hello",                  // 文本增量
      "reasoning_content": "Let me think", // 思考增量（DeepSeek 扩展）
      "tool_calls": [                       // 工具调用增量
        { "index": 0, "id": "call_xxx", "function": { "name": "...", "arguments": "{\"path\":\"" } }
      ]
    },
    "finish_reason": null
  }],
  "usage": { /* 仅在 stream_options.include_usage=true 才出现 */ }
}
```

`parse_sse_chunk` 把它翻译成项目内统一的 `StreamEvent`：
- `delta.content` → `ContentBlockDelta { Delta::TextDelta { … } }`
- `delta.reasoning_content` → `ContentBlockDelta { Delta::ThinkingDelta { … } }`
- `delta.tool_calls[i].function.arguments` → `ContentBlockDelta { Delta::InputJsonDelta { partial_json } }`
- `delta.tool_calls[i]` 第一次出现 → `ContentBlockStart { ContentBlockKind::ToolUse }`
- `finish_reason: "stop"` → `ContentBlockStop` + `MessageStop`
- `usage` → `MessageDelta { usage }`

**为什么要做这一步翻译？** 因为引擎不想关心 OpenAI 协议的细节。同一套 `StreamEvent` 也能给未来"非 OpenAI 协议"的 Provider 用——比如 Anthropic、Google 等。这是 **anti-corruption layer**。

---

## 5. Thinking Mode 的"三层防御"

DeepSeek V4 的 thinking 模式有个让人发指的限制：

> **每个带 `tool_calls` 的 assistant message 必须 *也带* `reasoning_content`**

否则 API 直接 400。在多轮对话里这个非常容易踩坑——如果上一轮的 `reasoning_content` 没保留，下一轮发请求就死。

项目里有 **三层防御**：

### 层 1：引擎侧补占位

在 `turn_loop.rs` 里 (`@D:\1os\DeepSeek-TUI-tuohai666\crates\tui\src\core\engine\turn_loop.rs:739-750`)：

```rust
let needs_thinking_block = !tool_uses.is_empty() || tool_parser::has_tool_call_markers(...);
let thinking_to_persist = if !current_thinking.is_empty() {
    Some(current_thinking.clone())
} else if needs_thinking_block {
    Some(String::from("(reasoning omitted)"))   // ← 保底占位
} else {
    None
};
```

### 层 2：build_chat_messages_for_request

在 `@D:\1os\DeepSeek-TUI-tuohai666\crates\tui\src\client\chat.rs:489-510`：

```rust
let has_text = !content.trim().is_empty();
let has_tool_calls = !tool_calls.is_empty();
let mut has_reasoning = !reasoning_content.trim().is_empty();
if !has_reasoning && requires_reasoning_for(role, has_tool_calls, ...) {
    reasoning_content = String::from("(reasoning omitted)");
    has_reasoning = true;
}
```

也就是说"刚发请求那一刻，如果某个 message 漏了 reasoning_content 但带 tool_calls，就给它塞个占位"。

### 层 3：bulletproof final sanitizer

在 `@D:\1os\DeepSeek-TUI-tuohai666\crates\tui\src\client\chat.rs:181-185`：

```rust
let replay_input_tokens = sanitize_thinking_mode_messages(
    &mut body,
    &request.model,
    request.reasoning_effort.as_deref(),
);
```

这是**最后一道闸**——直接扫整个 wire-payload，强制每个 assistant + tool_calls 的 message 有非空 `reasoning_content`。

为什么三层？因为：
- 层 1 漏掉的场景：会话从磁盘恢复，老 message 本身就缺 reasoning。
- 层 2 漏掉的场景：子代理直接往 message 里塞东西没走 layer 2。
- 层 3：终极保险。

> **学习要点**：当你和外部 API 之间有"硬约束"时，**关键约束要在最靠近 wire 的地方再校验一次**。靠"上游小心点"是不够的——人会犯错，你需要让它崩溃前的最后一刻自动修复。

### 5.1 副作用：`reasoning_replay_tokens`

层 3 的 `sanitize_thinking_mode_messages` 还顺手计算了"为了满足 thinking 约束，**额外** 发了多少 token"，这个数字回传给 UI 显示——让用户知道"这次重发因为占位 reasoning 多花了 X tokens"。透明计费，体验细节满分。

---

## 6. 重试：`send_with_retry`

来自 `@D:\1os\DeepSeek-TUI-tuohai666\crates\tui\src\client.rs:614-677`：

```rust
pub(super) async fn send_with_retry<F>(&self, mut build: F) -> Result<reqwest::Response>
where F: FnMut() -> reqwest::RequestBuilder,
{
    let retry_cfg: LlmRetryConfig = self.retry.clone().into();
    let request_result = with_retry(
        &retry_cfg,
        || {
            let request = build();
            async move {
                self.wait_for_rate_limit().await;
                let response = request.send().await
                    .map_err(|err| LlmError::from_reqwest(&err))?;
                let status = response.status();
                if status.is_success() { return Ok(response); }
                let retryable = status.as_u16() == 429 || status.is_server_error();
                if !retryable { return Ok(response); }
                let retry_after = extract_retry_after(response.headers());
                let body = bounded_error_text(response, ERROR_BODY_MAX_BYTES).await;
                Err(LlmError::from_http_response_with_retry_after(
                    status.as_u16(), &body, retry_after,
                ))
            }
        },
        Some(Box::new(|err, attempt, delay| {
            let (reason_label, human_reason) = retry_reason_label_and_human(err);
            crate::retry_status::start(attempt + 1, delay, human_reason);
        })),
    ).await;
    /* … */
}
```

设计要点：

1. **`build: FnMut()` 而不是 `&Request`** —— 为什么？因为 `reqwest::RequestBuilder` 是消耗型的，重试必须**重新构造一个**。这是 reqwest API 的契约。
2. **可重试 = 429 或 5xx**，4xx（除 429）直接抛错——业务错误重试也是错。
3. **`Retry-After` header 优先**，否则用指数退避。
4. **回调 `start(attempt, delay, reason)`** —— 把"正在重试"的状态推给 `retry_status` 模块，UI 顶部立刻显示 "Retrying in 2.3s — rate limited"。

---

## 7. 健康度：`ConnectionHealth`

来自 `@D:\1os\DeepSeek-TUI-tuohai666\crates\tui\src\client.rs:572-612`。三个方法：

```rust
async fn mark_request_success(&self) { /* 失败计数清零 */ }
async fn mark_request_failure(&self, reason: &str) { /* 累加失败 */ }
async fn maybe_probe_recovery(&self) { /* 失败到阈值后，发探测请求看看是不是恢复了 */ }
```

`maybe_probe_recovery` 防的是这种场景：连续 5 次 503 之后，用户网络恢复了但还没动手发新消息。下次 `send_with_retry` 失败时，会自动跑一次探测——成功就把 health 标 *recovered*，并 log "Connection recovered"，UI 顶部那条红条就消失。

> **学习要点**：**自动健康恢复探测**让用户不必手动重试。这是 production CLI 工具与 demo 的区别。

---

## 8. 非流式路径：`create_message_chat`

来自 `@D:\1os\DeepSeek-TUI-tuohai666\crates\tui\src\client\chat.rs:73-133`。是流式路径的"简化版"：

```rust
pub(super) async fn create_message_chat(&self, request: &MessageRequest) -> Result<MessageResponse> {
    /* 同样构造 body，但 stream: false */
    body["stream"] = json!(false);
    let response = self.send_with_retry(|| self.http_client.post(&url).json(&body)).await?;
    let response_text = response.text().await?;
    let value: Value = serde_json::from_str(&response_text)?;
    parse_chat_message(&value)
}
```

什么时候走非流式？
- 调度模式（`auto_model` routing）只问"用 Pro 还是 Flash"，不需要流；
- 子代理评估 tools 描述更新时也走非流式；
- 健康检查直接 GET `/v1/models`，没流。

---

## 9. Provider 多态

`api_provider: ApiProvider` 字段持有当前 Provider，里面影响以下分支：

```rust
// @D:\1os\DeepSeek-TUI-tuohai666\crates\tui\src\client\chat.rs:apply_reasoning_effort 之类
match self.api_provider {
    ApiProvider::DeepSeek => /* 用 reasoning_effort 字段 */,
    ApiProvider::NvidiaNim => /* NIM 用不同 schema */,
    ApiProvider::Fireworks => /* Fireworks 又一种 */,
    /* … */
}
```

**关键 insight**：这些 Provider 共享 90% 代码（OpenAI 兼容路径），只在以下少数几个点分叉：
- thinking/reasoning 字段名
- tool_choice 接受的形态
- usage 字段位置
- 模型名前缀

只要分叉点小，就**没必要为每个 Provider 写独立 client**。这跟 `ToolSpec` trait 同样的"统一抽象 + 少量定制"哲学。

---

## 10. 你修这一层时该改哪儿

| 你想改的事 | 改哪个文件 |
|---|---|
| 加新 Provider | `config.rs::ApiProvider` + `client/chat.rs` 的 `match self.api_provider` 分支 |
| 改流空闲超时 | `client/chat.rs::stream_idle_timeout` 函数（读 env） |
| 改重试策略 | `~/.deepseek/config.toml` 的 `[retry]` section + `client.rs::RetryPolicy` |
| 加新的 thinking-mode 防御 | `client/chat.rs::sanitize_thinking_mode_messages` 加规则 |
| 加自定义 HTTP header | `~/.deepseek/config.toml` 的 `http_headers` 表 |
| 改 `stream_options` | `client/chat.rs::handle_chat_completion_stream` 顶部 body 构造 |
| 强制 HTTP/1.1 | 设置 `DEEPSEEK_FORCE_HTTP1=1` |
| 加自签 CA | 设置 `SSL_CERT_FILE=/path/to/ca.pem` |

---

## 11. 自检小练习

1. 为什么 `stream_options.include_usage = true` 是 *必须* 设的？关掉会让上层哪个组件直接挂掉？
2. `byte_buf` 有 10 MB 上限。如果服务端故意一次性发 11 MB 的 SSE chunk，客户端会怎样？
3. 三层 thinking-mode 防御，**层 3 (sanitize_thinking_mode_messages)** 为什么不能省？请举一个真实场景说明只靠层 1+2 还会漏。
4. `parse_sse_chunk` 接的是 `&Value`，但 `delta.tool_calls[i].function.arguments` 是 *增量字符串*。客户端是怎么把多个 chunk 的字符串拼起来给上层一个完整的 JSON 的？（提示：客户端 *不* 拼，看 04 节工具状态机）
5. `send_with_retry` 的参数为啥是 `FnMut() -> RequestBuilder` 而不是 `&Request`？请用一句话说清楚 reqwest 的限制。

---

**下一节**：[`07-TUI-事件循环.md`](./07-TUI-事件循环.md) 我们回到 UI 端，看 ratatui 的 event loop 与引擎的 event loop 是如何协作的——以及为什么把它们解耦才能流畅。
