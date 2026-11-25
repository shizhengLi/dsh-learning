# 04｜LLM Provider、鉴权与 SSE 流式解析

> 源码基线为 DeepSeek Harness `dsh-v0.2.3-alpha.2`，提交 `4b399c39a6d1479cd84ca591b4d7c88e24d38a96`。[系列目录](../README.md) · [上一篇](03-agent-loop-state-machine.md)

模型已输出一段文字，又开始拼一个 `read` 调用的参数。接着 HTTP 响应体结束了，客户端没收到 `message_stop`。半段参数能执行吗？下一次模型请求会看到刚刚那段文字吗？这两个问题分别由协议适配器和 Agent Loop 作答。适配器负责判断响应是否完整，Loop 决定如何记下这次尝试，以及是否开启下一次尝试。[协议转换](../deepseek-harness/packages/llm/llm-deepseek/src/translate.ts) · [Agent Step](../deepseek-harness/packages/core/agent-loop/src/agent.ts#L470)

## 1. 同一个 Agent Loop 怎样调用多个 Provider

一次请求至少要区分三种身份。`provider` 是 DSH 注册的路由键，`model` 是交给这个路由的模型 ID，adapter 是负责实现协议的对象。基础配置同时挂载 API key 路由 `deepseek-official`、账号路由 `deepseek-account`，以及可配置其它提供方和兼容网关的 `llm-pi-ai`。两条 DeepSeek 路由共用 Messages 适配器，实现各自的鉴权和模型目录。pi-ai 插件通过另一套 adapter 接入同一个 `ctx.llm`，可以随设置添加路由。[基础配置](../deepseek-harness/packages/bundle/base/cordis.patch.yml#L133) · [API key 路由](../deepseek-harness/packages/llm/llm-deepseek-api-key/) · [账号路由](../deepseek-harness/packages/llm/llm-deepseek-account/) · [pi-ai 适配器](../deepseek-harness/packages/llm/llm-pi-ai/README.zh.md)

`LlmRuntime.prepareCall()` 按 provider 查找注册记录，让 adapter 解析模型能力和连接设置，然后固定有效请求参数。返回的 `PreparedLlmCall.stream()` 只能派发一次，并且校验派发参数与已解析参数一致。DeepSeek 适配器自己的 `prepareCall()` 又把 `connection` 绑定到这个调用，因此已经准备好的端点、凭据引用和请求默认值保持一致。凭据的实际值在发起请求时解析。这样，一边修改模型设置、一边读取旧请求，不会把旧端点和新配置拼成一次调用。[LLM Runtime](../deepseek-harness/packages/llm/llm/src/index.ts#L942) · [DeepSeek 准备调用](../deepseek-harness/packages/llm/llm-deepseek/src/adapter.ts#L50) · [配置切换测试](../deepseek-harness/packages/llm/llm-deepseek/tests/prepared-call.spec.ts#L43)

运行时还会按目标模型的能力选择图片、文件和工具更新怎样进入请求。普通文件引用统一投影成供模型定位的文本，文本模型的图片也走适配投影。已保存的 provider 私有回放字段只交给当前拥有历史路由与目标路由的同一个 adapter 实例。DeepSeek 再按模型 ID 校验自己的 thinking 签名，换模型时可以保留对话文字并省略签名。第 05 篇会继续拆这些上下文如何形成。[请求投影](../deepseek-harness/packages/llm/llm/src/index.ts#L1008) · [模态投影](../deepseek-harness/packages/llm/llm/src/index.ts#L1082) · [DeepSeek 回放元数据](../deepseek-harness/packages/llm/llm-deepseek/src/replay.ts#L33)

```mermaid
flowchart LR
    A[Agent Step] --> B[prepareCall 固定路由与模型配置]
    B --> C[解析本次请求的凭据]
    C --> D[serialize 生成 Messages JSON]
    D --> E[POST Messages]
    E --> F[UTF-8 解码与 SSE 分帧]
    F --> G[translate 转成 StreamChunk]
    G --> H[Agent 结算消息或失败尝试]
```

## 2. URL、鉴权和请求体怎样装配

默认 `baseURL` 是 `https://api.deepseek.com/anthropic`。`messagesApiRoot()` 去掉末尾的斜杠，已以 `/v1` 结尾的路径保留该段，其它路径追加 `/v1`，最终请求 `POST /anthropic/v1/messages`。发送时携带 `anthropic-version: 2023-06-01`、JSON `Content-Type` 与 `Accept: text/event-stream`。使用 Files 引用或历史工具更新时，代码按请求内容添加相应 beta 标头。[默认端点](../deepseek-harness/packages/llm/llm-deepseek/src/config.ts#L125) · [路径归一化](../deepseek-harness/packages/llm/llm-deepseek/src/messages-api.ts#L9) · [HTTP 请求](../deepseek-harness/packages/llm/llm-deepseek/src/adapter.ts#L144) · [DeepSeek Anthropic API 文档](https://api-docs.deepseek.com/guides/anthropic_api/)

| 路由 | 如何取得凭据 | 发送方式和边界 |
| --- | --- | --- |
| `deepseek-official` | 从本次连接配置取得 `apiKeyEnv` 引用，默认 `DEEPSEEK_API_KEY`，交给凭据解析。 | 以 `x-api-key` 发送。授权服务先检查目标是否属于所配置部署的 origin，以及 URL 是否内嵌凭据。 |
| `deepseek-account` | 使用账户服务为目标端点释放的登录凭据。 | 以 `x-dsh-auth-token` 发送。撤销信号可中止在途请求。HTTP 401 的处理会核对被拒绝的是哪份凭据，避免旧请求清除新登录。 |

这套拆分让共享传输层只接收已经解析出的 headers 和回调。`fetch` 使用 `redirect: 'error'`，所以端点通过授权以后也不会自动跟随重定向。凭据、端点、认证失败的处理归路由所有，数据序列化和流式消费归共享适配器所有。[API key 鉴权源码](../deepseek-harness/packages/llm/llm-deepseek-api-key/src/index.ts#L44) · [账号鉴权源码](../deepseek-harness/packages/llm/llm-deepseek-account/src/index.ts#L48) · [授权服务](../deepseek-harness/packages/account/deepseek-auth/src/index.ts#L180)

### 工具结果在 HTTP 请求体里长什么样？

内部历史中，assistant 消息含 `tool-call` 块，工具结果是一条 `role: 'tool'` 的消息。`serialize()` 会把前者写为 `assistant` 的 `tool_use`，把后者写为下一段 `user` 中的 `tool_result`，通过 `tool_use_id` 对齐。并行工具结果排在同段普通用户文字之前。序列化还检查重复调用 ID、无匹配调用的结果，以及历史尾部未回填的调用。上一篇的中断修复正好为这些协议约束提供了完整的历史。[序列化与调用配对](../deepseek-harness/packages/llm/llm-deepseek/src/serialize.ts#L118)

```json
{
  "model": "your-model-id",
  "stream": true,
  "max_tokens": 2048,
  "thinking": { "type": "disabled" },
  "messages": [
    { "role": "user", "content": [{ "type": "text", "text": "读取 greeting.txt" }] },
    {
      "role": "assistant",
      "content": [{ "type": "tool_use", "id": "call_1", "name": "read", "input": { "file_path": "greeting.txt" } }]
    },
    {
      "role": "user",
      "content": [{ "type": "tool_result", "tool_use_id": "call_1", "content": [{ "type": "text", "text": "hello" }] }]
    }
  ]
}
```

这个示例省略了可选系统指令和工具列表。实际请求的工具定义由 `parameters` 映射为 `input_schema`，开启按需加载的定义还会写 `defer_loading`。初始系统文本进入顶层 `system`，支持 `in-history` 的模型可以在历史中保留后续系统更新的位置。`reasoningEffort` 决定是否使用 thinking 和所需的 `output_config.effort`。图片可通过 Files API 取得引用，部分 Files 解析失败会转入受限制的 inline 路径。这些选择只改变请求表示，Session 中仍保存其规范消息和引用。[body 组装](../deepseek-harness/packages/llm/llm-deepseek/src/serialize.ts#L58) · [生成控制](../deepseek-harness/packages/llm/llm-deepseek/src/serialize.ts#L147) · [图片请求](../deepseek-harness/packages/llm/llm-deepseek/src/adapter.ts#L96)

## 3. 从 HTTP 字节到 SSE 帧，再到消息块

网络读取只给出若干字节块，UTF-8 字符、SSE 行和 JSON 文档都有可能跨块。DSH 先让 `TextDecoderStream` 还原文本，再让 `EventSourceParserStream` 按空行等 SSE 规则分帧。`parseSse()` 对帧的 `data` 做 JSON 解析，检查它是有字符串 `type` 的对象，并在 SSE 声明了 `event` 名时检查两者一致。`error` 帧会直接化成 provider 错误。缺少结束空行的尾部帧不会在 EOF 被强行发出来。[SSE 处理](../deepseek-harness/packages/llm/llm-deepseek/src/sse.ts#L13) · [SSE 规范](https://html.spec.whatwg.org/multipage/server-sent-events.html#parsing-an-event-stream) · [逐字节分帧测试](../deepseek-harness/packages/llm/llm-deepseek/tests/stream.spec.ts#L118)

`translate()` 在 SSE 帧之上维护 Messages 的协议状态。先等 `message_start`，再按 wire `index` 找已打开的块。它将块映射为首次出现顺序的内部索引，文字、推理与工具参数各走自己的增量类型。块结束时发出完整 `block-end`，消息结束时检查块已关闭、结束原因存在，然后发出 `usage` 和 `finish`。协议允许额外的 envelope 事件，因此 `ping` 等未知类型可以经过此层而不改变内容。[状态转换](../deepseek-harness/packages/llm/llm-deepseek/src/translate.ts#L111)

| Messages 事件或字段 | DSH 中的结果 |
| --- | --- |
| `text_delta` / `thinking_delta` | 分别产生 `text-delta` / `reasoning-delta`，按块累加。 |
| `signature_delta` | 积累在推理块的版本化 replay metadata，供兼容路由后续复用。 |
| `input_json_delta` | 拼接 `partial_json`，在 `content_block_stop` 形成完整参数字符串。正常完成时检查它确实是 JSON 对象。 |
| `message_delta.usage` | 对同名累计计数取后值。缺席字段保留先前计数。 |
| `message_stop` | 验证结束条件，发布 usage 和结束信息。`refusal` 会先报告已计费 usage，再返回 `CONTENT_REFUSED`。 |

如果收到两条累计的 `output_tokens`，先是 `2`，后来是 `5`，结果应记为 `5`。在这个 Messages adapter 的口径里，`totalTokens` 汇总输入、输出、缓存读和缓存写四项，缓存计数来自各自独立的 wire 字段。跨 Provider 报表应先检查各协议的 usage 口径。另一个边界是 `max_tokens`。原始流可以保留截断的工具数据供排查，assembler 会从这次消息及其回放元数据中移除工具调用块，Agent Loop 对 `max-tokens` 结果会在工具派发前结束当前处理。[usage 映射](../deepseek-harness/packages/llm/llm-deepseek/src/translate.ts#L35) · [协议终止](../deepseek-harness/packages/llm/llm-deepseek/src/translate.ts#L151) · [Assembler](../deepseek-harness/packages/llm/llm/src/assembler.ts#L125) · [工具派发前的 token 分支](../deepseek-harness/packages/core/agent-loop/src/agent.ts#L574)

### 有心跳但没有新文字，算超时吗？

这个适配器的 `streamIdleTimeoutMs` 默认是 300 秒。Idle watchdog 约束的是一次未完成的读取需求。SSE 注释心跳和完整帧会重置计时，`ping` 也会在进入 translator 前提供活动信号。消费者处理上一块而尚未请求下一块时，计时器不运行。监控上可分别记录传输活跃、首个可见 token 和整次请求耗时，这样心跳正常但长期没有文本的情况仍能看出来。[Watchdog 默认值](../deepseek-harness/packages/llm/llm-deepseek/src/defaults.ts#L3) · [Watchdog 实现](../deepseek-harness/packages/util/timeout/src/index.ts#L154) · [适配器取消和超时](../deepseek-harness/packages/llm/llm-deepseek/src/adapter.ts#L70)

## 4. HTTP 已经 200，为什么这次调用还能失败？

HTTP 状态决定响应是否进入流式消费。非 2xx 响应按状态以及可识别的 provider 错误字段归类，错误可携带 `requestId` 和 `Retry-After` 解析出来的毫秒数。已经拿到 200 后，SSE 仍可能发 `error`，或者缺少 `message_stop`、出现错误的块顺序、含无效参数。底层 I/O 抛出的异常再按适配器边界映射为 `TRANSPORT` 等类型。两条路径最终都可以让 `LlmRuntime` 产出错误 `finish`，让 Agent 记下失败尝试。中间件和消费者抛出的异常有独立的失败边界。[错误分类](../deepseek-harness/packages/llm/llm-deepseek/src/transport.ts#L28) · [EOF 检查](../deepseek-harness/packages/llm/llm-deepseek/src/translate.ts#L174) · [适配器失败边界](../deepseek-harness/packages/llm/llm/src/index.ts#L1050)

| 现象 | 典型分类 | 恢复时看什么 |
| --- | --- | --- |
| HTTP 429 或流内 `rate_limit_error` | `RATE_LIMIT` | 确认是限流还是额度耗尽，按策略考虑 `Retry-After`。 |
| HTTP 401/403 | 共享层通常是 `AUTH`，账号路由的 401 可映射到 `ACCOUNT_TOKEN_INVALID` | 核对路由、凭据与目标部署，账号刷新还要保护新登录状态。 |
| 本次请求超过上下文窗口 | `CONTEXT_WINDOW_EXCEEDED` | 检查已记录的上下文与预算，符合条件时走压缩恢复。 |
| 正常读到 EOF，缺 `message_stop` | `STREAM_CLOSED` | 保留结算到的失败尝试，并依据路由配置判断是否重试。 |
| 读流 I/O 异常，或者读取空闲超时 | 可能是 `TRANSPORT` 或 `TIMEOUT` | 结合网络错误与 deadline，确认取消原因和退避预算。 |
| JSON、块顺序或正常完成的工具参数不合法 | `MALFORMED_RESPONSE` | 排查协议与网关兼容性，保留结构化失败证据。 |

普通重试策略会判断错误码、次数和等待上限。默认的 transient 错误名单包括 `EMPTY_RESPONSE`、`RATE_LIMIT`、`SERVER`、`TIMEOUT`、`TRANSPORT`。`STREAM_CLOSED` 是一个可配置重试的错误码，默认名单里没有它。合法的 `Retry-After` 在上限内优先，缺少时用指数退避与抖动。普通模式下超过接受范围的服务端等待时间会交给后继恢复策略处理。等待本身可取消，并以 `llm/retry` 和 `llm/retry-started` 区分已经计划和真正开始的重试。[默认策略](../deepseek-harness/packages/llm/llm/src/retry-policy.ts#L15) · [允许配置的错误码](../deepseek-harness/packages/llm/llm/src/error.ts#L62) · [重试执行](../deepseek-harness/packages/llm/llm-retry/src/index.ts#L147)

一次可重试的适配器失败会先结算为 `assistant/attempt`。重试在同一个 step 内重新准备模型调用，已经接纳的用户消息保留，失败尝试的文字留给查看或回放该尝试的消费者。成功时另写本次尝试的 `assistant/message`。并存的多 Provider 使路由可选，`llm-retry` 在既定 step 中重试也仍受该 step 的模型选择约束。设计跨 Provider 故障切换时，还需要明确切换点、候选路由和历史能力降级规则。[失败尝试与重试](../deepseek-harness/packages/core/agent-loop/src/agent.ts#L531) · [路由选择约束](../deepseek-harness/packages/core/agent-loop/src/agent.ts#L601)

## 5. 面试官会怎么追问

### 为什么不能在每次 `read()` 后直接 `JSON.parse()`？

先讲清三个边界：字节读取、SSE 帧、Messages 内容块。一个 UTF-8 汉字可以被分到两次读取，一帧可以分多次读到，一组工具参数又能跨多个已经合法的 JSON 帧。继续追问可以要求候选人构造逐字节拆分、两帧同块到达、缺最后分隔空行三个测试，指出每一层应该成功还是报错。

### 路由配置改了，正在生成的请求用哪个 key？

说清 `prepareCall()` 固定的是配置、凭据引用和路由实现，本次实际发送前再从该引用读取密钥。验证时可准备 A/B 两个模拟端点，在准备调用后立刻把配置切到 B，断言旧调用仍去 A 并从 A 的凭据引用取值。追问账号凭据在途被撤销与 401 到达顺序，考察谁有权让会话凭据失效。

### HTTP 200 后已吐字，流断了要不要重试？

先按证据区分明确完成、缺少终止帧、I/O 错误和用户取消。再决定重试预算、客户端怎样展示失败尝试，以及哪些内容进入历史。若一段工具参数仍在拼接，等待协议结束和结构校验。进一步追问时可以让候选人说明如何记录 request ID、每次 attempt 的 usage 和重试原因，以及在限流高峰怎么避免重试放大流量。

### 怎样设计一套可接入三家模型的网关？

可以用统一的请求、块、工具调用、usage 与失败类型承接通用逻辑，每个路由另外声明模态、token 限制、工具更新及回放能力。在开始调用前确定路由和凭据归属，记录实际 provider/model 与请求标识。故障切换时用候选路由重新验证工具能力和历史表示。面试官可给一个压力条件：第一家支持 thinking 签名与图片，第二家只支持文本，切换后仍需继续工具会话。要求候选人画出能力判断、上下文降级、失败尝试保留和计费归属的时序。

下一篇进入 System Prompt、仓库指令和运行时上下文，追踪它们在何时成为模型真正可见的输入。
