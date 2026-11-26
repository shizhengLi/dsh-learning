# 29｜可观测性、Token 成本与性能诊断

> 源码基线为 DeepSeek Harness `dsh-v0.2.3-alpha.2`，提交 `4b399c39a6d1479cd84ca591b4d7c88e24d38a96`。[系列目录](../README.md) · [上一篇](28-web-desktop-sdk-acp-and-reconnect.md) · [上游仓库](https://github.com/deepseek-ai/deepseek-harness)

一次修改代码的请求过去 10 秒完成，今天的 P95 变成 20 秒。可能是 Provider 首段响应变慢，也可能是 Agent 多试了一次、某个工具排队，或者长历史让浏览器卡住。DSH 的 Session 日志能按轮次和步骤串起过程。排查要先看日志已经记录了什么，再决定在哪个边界补测量。

## 1. 从 Session 对齐到模型尝试与工具调用

| 层次 | 可以用什么关联 | 源码中能观察到什么 |
| --- | --- | --- |
| Session 和 turn | `sessionId`、`seq`、`turn` | `turn/start` 到 `turn/end` 的时间与结果。用户提交到 turn 开始的等待需要另找入口时间。 |
| Step | `turn + step` | 步骤开始、模型结果、工具调用及步骤结束。工具结果可能引出下一 step。 |
| 模型 attempt | 实时的 `attemptId`。持久分析结合轮次、步骤及重试边界。 | 成功或中断的内容、尝试的流记录与用量。一个 step 可能发出多次尝试。 |
| Tool | `callId`，结果也记录调用来源序号。 | `tool/call` 到 `tool/result` 的跨度和失败类型。细分排队、审批、工具体执行时还需要更细的打点。 |

模型步骤组装请求后创建 `AssistantStreamAttempt`。接收分片时 Host 写入接收时间，再把分片交给累积器和实时流。正常回复以 `assistant/message` 提交。失败的开始尝试可以用 `assistant/attempt` 结算，部分中断内容也可进入消息。`request/context` 记录 Provider、模型和容量等配置变化，相同配置可以跨请求沿用，分析时应读最近的配置状态。[步骤与请求](../deepseek-harness/packages/core/agent-loop/src/agent.ts#L357) · [尝试生命周期](../deepseek-harness/packages/core/agent-loop/src/agent.ts#L469) · [分片接收时间](../deepseek-harness/packages/core/agent-loop/src/assistant-stream.ts#L60) · [请求上下文](../deepseek-harness/packages/core/agent-loop/src/agent.ts#L700)

重试插件先追加含原因和 `delayMs` 的 `llm/retry`，等待结束后再写 `llm/retry-started`。两个事件用 `retryId` 关联，可区分等待与重新发请求。现有快照里，step 1 先有 `assistant/attempt`，随后记录退避和重试，最后才出现成功消息。从单个成功回复的时间推这轮模型开销，就会漏掉之前的尝试和退避。工具日志则通过调用 ID 和来源序号对齐，同一批并行工具可能在时间上重叠。[重试与退避](../deepseek-harness/packages/llm/llm-retry/src/index.ts#L157) · [重试快照](../deepseek-harness/snapshots/session/empty-response-retry-current/session.v4.jsonl#L15) · [工具事件关联](../deepseek-harness/packages/core/tools/src/batch.ts#L243)

## 2. Session Telemetry 怎样把记录交给 OpenTelemetry

Session 日志保存会话事实。遥测协调器复制事件信封和载荷，经 `session-telemetry/record` waterfall 处理，再移交给后端。通用组件支持实时或按需捕获。这份基础装配的 Session OTLP 使用 `FEEDBACK_ONLY`，只有符合条件的本会话反馈事件才触发按需捕获截至该反馈的历史。普通请求性能分析可以从本地日志入手。部署需要持续监控时，可另接只采集运行指标的链路，并按其数据范围做授权与脱敏。[捕获模式](../deepseek-harness/packages/session/session-telemetry/README.zh.md#use-this-package) · [默认装配](../deepseek-harness/packages/bundle/base/cordis.patch.yml#L206) · [反馈捕获条件](../deepseek-harness/packages/session/session-telemetry-otel/src/index.ts#L42) · [按需接线](../deepseek-harness/packages/session/session-telemetry-otel/src/index.ts#L219)

上报器把会话事件序列化为 OTel 日志记录，带 Session ID、事件内容和事件时间。它对单个请求的未压缩字节数、批次条数与队列容量设限，再经 OTLP 发送。基础配置的请求上限为 4,000,000 字节，队列上限为 2,048 条。超大单条或队列满时可拒绝本条。协调器的游标记录已经交接到后端的序号，无法证明 Collector 已接收。崩溃可丢失队列中的数据，重复捕获也可能重复导出日志前缀。接收方可按 Session ID、格式版本和事件序号去重。[复制、脱敏与交接](../deepseek-harness/packages/session/session-telemetry/src/coordinator.ts#L179) · [日志字段映射](../deepseek-harness/packages/telemetry/otel/src/session-log.ts#L206) · [批处理与字节限制](../deepseek-harness/packages/telemetry/otel/src/session-log.ts#L89) · [后端的交付边界](../deepseek-harness/packages/session/session-telemetry-otel/README.zh.md#known-limitations-and-deferred-work) · [OpenTelemetry 日志模型](https://opentelemetry.io/docs/specs/otel/logs/data-model/)

这里导出的是 Session 事件副本。分布式 Trace 需要另在请求接纳、Provider 调用、工具执行、持久化和客户端消费等边界建立关联。部署扩展时可以把 Session 与调用标识放进可追查的详情或 Trace，指标标签保留有限的 Provider、模型、路由、工具类别和结果类型。脱敏规则要覆盖准备导出的内容字段。反馈样本的触发条件也决定它适合诊断个案，无法代表全部线上流量的 P95。[会话遥测脱敏约定](../deepseek-harness/packages/session/session-telemetry/README.zh.md#use-this-package)

## 3. Token 数字怎样从上下文压力走到成本

`tokenMeter.measure()` 用 Session 当前可见内容和请求头估算压力，供压缩与占用显示读取。文本采用近似规则，可见表面的变化相对用量锚点计算增量。请求头匹配且 Provider 用量能作为可靠锚点时，用已报告用量修正估算，否则估算当前请求。图片可用适配器声明的视觉 Token 价格，卸载后的出现位置按占位文本处理。这类价格的单位首先是 Token。它回答当前请求大概有多大。[计量服务](../deepseek-harness/packages/llm/token-meter/README.zh.md#use-this-package) · [锚点选择](../deepseek-harness/packages/llm/token-meter/src/index.ts#L139)

`tokenUsage` 投影累加 Provider 报告的未缓存输入、缓存读写和输出。对同一尝试，用最新或终态样本替换旧样本。`llm/retry-started` 关闭替换槽，下一次尝试的用量独立累计。用于显示整轮精确用量的 `deriveTurnTokenUsage()` 更严格，轮次不完整、一次尝试缺必要用量或总量矛盾时，不返回精确合计。推理 Token 在可验证的计量中属于输出子集，成本计算要避免重复加。适配器对不同 Provider 的输入与缓存字段做归一化，计算账单时仍需核对各路由的计价约定。[累计投影](../deepseek-harness/packages/llm/token-meter/src/usage-projection.ts#L104) · [逐次尝试校验](../deepseek-harness/packages/llm/token-meter/src/turn-usage.ts#L81) · [重试边界](../deepseek-harness/packages/llm/token-meter/src/turn-usage.ts#L193) · [DeepSeek 字段映射](../deepseek-harness/packages/llm/llm-deepseek/src/translate.ts#L37)

例如仓库的 replay 快照在同一轮记录两个请求。第一步未缓存输入 2,877、输出 90。第二步未缓存输入 168、缓存读取 2,816、输出 25，第二步已有记录的输入共 2,984。两次请求报告的总数分别是 2,967 与 3,009，合计 5,976。实际费用要用各尝试对应的模型、时间和费率，将未缓存输入、缓存读取、缓存写入、输出这几个互斥桶分别计价后相加。回放数据可以说明口径，不能给出线上账单。[第一步用量](../deepseek-harness/snapshots/session/deepseek-messages-replay/session.v4.jsonl#L15) · [第二步用量](../deepseek-harness/snapshots/session/deepseek-messages-replay/session.v4.jsonl#L22) · [样本去重与重试用例](../deepseek-harness/packages/llm/token-meter/tests/token-usage-projection.spec.ts#L166)

## 4. P95 翻倍时怎样分辨四类瓶颈

先定义用户看到的首字延迟和完整回复时间，再把同一批慢请求关联到 Host 时间线。看首个协议分片、首个可展示文本和浏览器首次可见三个不同的点。首个分片可能只有元数据或推理，Host 收到文本也要经过传输、折叠与浏览器渲染。耗时在同一进程内使用单调时钟计算，跨设备比较要用关联 ID 与各端的局部耗时。OpenTelemetry 的模型推理首分片指标提供了一种可参考的测量边界。[分片时间的源码](../deepseek-harness/packages/core/agent-loop/src/assistant-stream.ts#L60) · [OpenTelemetry GenAI 指标模型](https://github.com/open-telemetry/semantic-conventions-genai/blob/main/model/gen-ai/metrics.yaml)

| 可疑环节 | 慢样本上先核对什么 | 怎样缩小范围 |
| --- | --- | --- |
| Provider | 请求开始到首分片、首字及完成的时间。另看重试数、退避、输出长度与缓存构成。 | 固定模型、输入规模与路由分组。若只有首分片慢，继续区分连接、请求传输和 Provider 排队证据。 |
| 工具与调度 | 工具获准、实际开始和结算时间，以及活跃数、等待队列与超时原因。 | 先分开审批和容量等待。并发工具找阻塞后续步骤的分支。 |
| Host 本地 I/O | Session 打开、历史读取、投影恢复、持久化和事件循环延迟。 | 区分冷启动与热读，比较相同历史规模和磁盘环境。 |
| Web 客户端 | 到帧、处理完成、首次可见、主线程任务时间和历史窗口大小。 | 对照 Host 已发而浏览器未显示的时间，检查是否在折叠大量历史或执行昂贵渲染。 |

用同一个请求的时间线找关键路径。多个工具并行时，总工具耗时可大于用户端耗时。各阶段的 P95 也不能相加来推总 P95，因为落在尾部的请求可能不同。可以先取端到端最慢的一组请求，再比较它们的阶段分布，单独记录缺失、取消和失败样本。看「是否 Provider 变慢」时，还应把重试多、输入更长和模型路由改变区分开。

## 5. 仓库性能基准各自能回答什么

| 基准 | 计时边界 | 适合定位 |
| --- | --- | --- |
| `long-session-browser` | Chromium 加载合成的 240 轮会话，测历史打开、分页、Trajectory、流式回复和输入。 | Host 与 Web UI 的端到端回放路径，以及浏览器主线程负载。 |
| `active-stream-reconnect` | Node worker 将含 100,000 个 delta 的基线交给生产客户端的替换逻辑，测替换时间和保留堆。 | 隔离重连时的客户端数据折叠，覆盖下一帧仍能接收的正确性。 |
| `session-corpus` | 独立合成语料测 3,000 个 Session 的列表，及各 1,000 个 Session 的搜索和 fork。 | 定位 Host 列表、索引冷启动与长历史分叉成本。 |

这些基准固定输入和测量终点，可以判断一条链路是否相对基线退化。长会话基准用回放适配器，并通过浏览器动画帧确认状态可见。重连基准在 Node 测折叠，无法说明浏览器渲染花了多久。仓库的中位数预算也与生产 P95 有不同的用途。容量控制还需要看线上并发、失败及流量构成，在相近工作负载下比较发布前后的分布。[长会话浏览器基准](../deepseek-harness/benchmarks/long-session-browser/README.zh.md) · [活跃流重连基准](../deepseek-harness/benchmarks/active-stream-reconnect/README.zh.md) · [会话语料基准](../deepseek-harness/benchmarks/session-corpus/README.zh.md)

## 6. 面试官怎么逐层追问

### 有了 turn/start 到 turn/end，怎样得到模型首字延迟？

先定位同一轮中的 step 和 attempt，找到 Host 收到的首个文本 delta。请求发送的准确时间还要有 Provider 边界打点。再追问客户端看到文字为何更晚，回答应该包含网络传输、客户端折叠和渲染，并明确哪个计时点叫作首字。

### 为什么上下文压缩后，历史 Token 用量不下降？

上下文变小时，后续请求的估算压力可能下降，旧请求已经产生的用量仍在日志里。接着问同一 step 出现三次重试，说明同一尝试的用量样本怎样替换、不同尝试怎样累加，以及哪些失败可能没有可用用量。

### Collector 收不到某条日志，能否断定 Agent 没有执行？

先看本地权威日志，再检查反馈捕获范围、队列拒绝、传输和接收确认。追问怎样建立审计级交付，可以给出持久 outbox、接收方去重、确认游标与独立保留策略。还要说明可观测性指标和完整审计各需要哪些数据。

### P95 翻倍但 Provider 指标稳定，下一步往哪查？

核对样本范围与指标定义，再看是否出现更多步骤、重试或工具等待。检查 Host 的 Session I/O 与浏览器处理帧的时间，用仓库基准缩小到客户端折叠、渲染或 Host 侧。结论应能指向具体阶段和一组可复查的慢请求。

下一篇进入 Agent 评测，继续看怎样用回放、任务成功率、成本和人工介入做发布判断。
