# DeepSeek Harness 源码精读与 Agent 开发面试

面向 DeepSeek、阿里云、腾讯、字节跳动等公司的 Agent 开发和 LLM 应用岗位。以 DeepSeek Harness（DSH）为主线，沿一次模型请求追到工具执行、会话持久化、客户端同步，再把这些实现放进 Coding Agent、办公 Agent 和多租户平台的设计题中。公司名称对应备考方向，题目由本系列按源码与工程场景编写。

源码基线是本地 `deepseek-harness/` 的 `4b399c39a6d1479cd84ca591b4d7c88e24d38a96`，对应 `dsh-v0.2.3-alpha.2`。入口见 [项目 README](deepseek-harness/README.zh.md)、[架构文档](deepseek-harness/docs/architecture.zh.md)和 [默认基础配置](deepseek-harness/packages/bundle/base/cordis.patch.yml)。系列共规划 **38 篇**，按编号推进。篇号同时作为后续文章的引用编号。

## 技术栈与实现总览

| 领域 | 技术栈与实现原理 | 源码入口 |
| --- | --- | --- |
| 工程与启动 | TypeScript、Node.js ESM、pnpm Workspace，`tsc` 处理编译和类型检查，`tsdown` 与 Vite 面向不同产物。CLI 选择 Profile，再按顺序叠加 Bundle 和配置补丁，得到实际插件树。 | [根清单](deepseek-harness/package.json) · [Profile 启动器](deepseek-harness/apps/cli/src/profile-boot.ts) |
| 插件与运行时 | Cordis 管理依赖、作用域和可释放的 Effect。Agent、Session、工具执行器与模型适配器按服务组合，通过有类型的事件提供扩展点。Schemastery 与 Zod 用于配置和运行时数据校验。 | [Cordis](deepseek-harness/vendor/cordis/) · [事件与派发语义](deepseek-harness/docs/cordis-primer.zh.md) · [Agent Loop](deepseek-harness/packages/core/agent-loop/src/) |
| 模型调用与上下文 | DeepSeek Messages 与 pi-ai 进入统一 LLM 接口。Provider 转换请求、解析流、归一化消息与工具调用。请求历史从 Session 事件投影，System Prompt、工具 Schema、压缩和附件按各自机制参与组装。 | [LLM 接口](deepseek-harness/packages/llm/llm/src/) · [DeepSeek Provider](deepseek-harness/packages/llm/llm-deepseek/src/) · [System Prompt](deepseek-harness/packages/core/system-prompt/src/) |
| 会话与数据库 | 默认使用追加式 JSONL 会话日志，默认启用 Zstandard 压缩。SQLite 会话插件显式启用，使用 `node:sqlite`、WAL、事务与写入租约。另有独立 SQLite FTS5 派生索引，默认配置关闭正文全文检索。非会话业务状态的基础配置走 JSON，另有 SQLite 文档型 KV 后端。 | [默认装配](deepseek-harness/packages/bundle/base/cordis.patch.yml) · [JSONL](deepseek-harness/packages/session/session-persistence-jsonl/) · [SQLite 会话](deepseek-harness/packages/session/session-persistence-sqlite/) · [全文索引](deepseek-harness/packages/session-query/session-query-sqlite/) · [SQLite KV](deepseek-harness/packages/storage/storage-sqlite/) |
| Skill 与扩展 | 文件系统 Provider 发现技能并解析 YAML frontmatter，Registry 合并多个来源与作用域，`tool-skill` 发布可见目录并按需加载正文。Preset 组合能力，Plugin Manager 管理插件安装和启用。 | [Skill Registry](deepseek-harness/packages/skill/skill/src/) · [本地发现](deepseek-harness/packages/skill/skill-filesystem/src/) · [技能工具](deepseek-harness/packages/skill/tool-skill/src/) · [Plugin Manager](deepseek-harness/packages/boot/plugin-manager/src/) |
| MCP | MCP SDK Client/Core 提供协议能力，DSH 连接本地 stdio 或远程 Streamable HTTP 服务，将工具注册到自己的工具系统。Manager 管理连接实例与状态，Resources 提供资源发现和读取。 | [连接与传输](deepseek-harness/packages/mcp/mcp-client/src/) · [管理器](deepseek-harness/packages/mcp/mcp-manager/src/) · [资源桥](deepseek-harness/packages/mcp/mcp-resources/src/) |
| 工具与执行环境 | 原生执行器处理模型工具调用，PTC 执行器通过 `run_code` 给程序提供受控工具接口。文件系统、Shell、PTY、LSP、子进程和沙箱由不同插件提供，权限策略在工具流水线上决定允许、拒绝或请求审批。 | [工具流水线](deepseek-harness/docs/tool-execution-pipeline.zh.md) · [PTC](deepseek-harness/packages/ptc-runtime/ptc-runtime-node/) · [本地沙箱](deepseek-harness/packages/sandbox/sandbox-local/) |
| 前端、SDK 与原生层 | Web 使用 React、Vite 与 Node HTTP/HTTPS Host，Typert Gateway 承载类型化调用与流。桌面使用 Electron，CLI、TypeScript/Python SDK 与 ACP 提供独立入口。`native/system` 提供平台相关原生能力。 | [Web](deepseek-harness/apps/web/) · [Desktop](deepseek-harness/apps/desktop/) · [Gateway](deepseek-harness/packages/api/gateway/) · [SDK](deepseek-harness/packages/sdk/) · [Python](deepseek-harness/python/sdk/) · [原生层](deepseek-harness/native/system/) |
| 验证与观测 | Vitest、会话 Snapshot、真实 API E2E、浏览器性能基准与 OpenTelemetry 插件。回放用于验证已记录轨迹上的行为，真实模型和环境的变化由 E2E 与任务评估覆盖。 | [测试策略](deepseek-harness/docs/testing.zh.md) · [Snapshots](deepseek-harness/snapshots/) · [Benchmarks](deepseek-harness/benchmarks/) · [OTel](deepseek-harness/packages/telemetry/otel/) |

在该源码版本的依赖清单、锁文件、运行配置及实现中，未找到 Redis 集成证据。DSH 的 `jobs-local` 在进程内运行，定时任务另有存储及投递逻辑。第 31 篇从这些约束出发，讨论云端何时引入 Redis、Redis 失联时如何恢复，以及关系数据库应该持有哪些权威状态。[本地 Jobs](deepseek-harness/packages/jobs/jobs-local/) · [定时任务](deepseek-harness/packages/schedule/schedule/) · [Redis Streams](https://redis.io/docs/latest/develop/data-types/streams/)

## 系列目录

### 一、从启动到模型和工具执行（01–07）

| 篇 | 文章 | 内容与面试切入点 |
| --- | --- | --- |
| 01 | [**一条指令怎么跑完**](notes/01-headless-agent-run.md) | 从 `dsh --profile headless` 跟到 Agent 创建、消息入队、模型流、工具回填和持久化。画出 user、turn、step、attempt、tool call 的时序。追问：一次用户输入触发三次模型请求时，各层分别有几个实例？在哪个状态可以退出？[CLI](deepseek-harness/apps/cli/src/bin.ts) · [Agent 生命周期](deepseek-harness/docs/agent-lifecycle.zh.md) |
| 02 | [**Cordis、Profile、Bundle 与 YAML 覆盖**](notes/02-cordis-profile-bundle.md) | 从 `apply`、注入、Effect/Disposer、Scope 和 Waterfall 到 Profile 合成、按 ID patch、配置 HMR。追问：Provider 热替换时，怎样定义旧调用、下一轮调用和资源释放的先后关系？[Cordis Primer](deepseek-harness/docs/cordis-primer.zh.md) · [Profile Boot](deepseek-harness/apps/cli/src/profile-boot.ts) |
| 03 | [**Agent Loop 与运行状态机**](notes/03-agent-loop-state-machine.md) | 跟读创建、恢复、inbox、turn/step 边界、继续条件、取消与错误收尾。区分瞬时流事件、已提交消息与仅供审计的失败尝试。追问：模型已吐字、工具结果只完成一半时崩溃，重启如何判定后续动作？[Agent Loop](deepseek-harness/packages/core/agent-loop/src/) · [Agent 服务](deepseek-harness/packages/core/agent/src/) |
| 04 | [**LLM Provider、鉴权与 SSE 流式解析**](notes/04-llm-provider-auth-sse.md) | 从 DeepSeek Messages 请求翻译到 SSE 增量、推理内容、工具参数拼装、usage、prepared call、重试与多 Provider 路由。追问：用户已看到部分文本，此时 HTTP 流断开，重试对界面和历史分别意味着什么？[DeepSeek 适配器](deepseek-harness/packages/llm/llm-deepseek/src/) · [统一流接口](deepseek-harness/packages/llm/llm/src/) · [重试](deepseek-harness/packages/llm/llm-retry/) |
| 05 | [**System Prompt、指令与上下文来源**](notes/05-system-prompt-and-context.md) | 追踪 Prompt Section、变量、工具 Schema、仓库指令、工作目录、文件与历史会话引用，并理解改动如何进入日志和请求。追问：修改 `AGENTS.md`、切换工作目录或停用工具时，怎样判断本轮看到的上下文？[Prompt](deepseek-harness/packages/core/system-prompt/src/) · [Context](deepseek-harness/packages/context/) |
| 06 | [**Function Calling 与原生工具流水线**](notes/06-function-calling-tool-pipeline.md) | 从注册、参数和结果校验、Agent Scope、`pre-execute`/`execute`/`post-execute` 到并行安全、错误回填与 UI 呈现。追问：一批调用里有并行读、串行写和审批等待，你会给哪些步骤设置并发限制和取消点？[工具流水线](deepseek-harness/packages/core/tools/src/pipeline.ts) · [原生执行器](deepseek-harness/packages/core/tool-executor-native/) |
| 07 | [**PTC：让模型写程序调用工具**](notes/07-programmatic-tool-calling.md) | 用 `run_code` 与原生 Function Calling 做对照，跟进 SDK 生成、子进程运行、回调桥、调用预算和大结果处理。追问：50 个细粒度读操作时，什么时候选模型分批调用，什么时候选 PTC？怎样设置资源上限和部分失败语义？[PTC 执行器](deepseek-harness/packages/core/tool-executor-ptc/) · [Node Runtime](deepseek-harness/packages/ptc-runtime/ptc-runtime-node/) |

### 二、会话、数据库、检索与记忆（08–14）

| 篇 | 文章 | 内容与面试切入点 |
| --- | --- | --- |
| 08 | [**Session Event Log、投影与回放**](notes/08-session-event-log-projection-replay.md) | 用 `SessionEvent`、序列号、`deriveMessages()` 和 Projection 解释事件溯源。拆解模型视图、UI 视图、审计历史和分叉前缀。追问：如何验证恢复前后模型输入一致？事件回放和重新执行工具是什么关系？[Session](deepseek-harness/packages/core/session/) · [Projection](deepseek-harness/packages/session/session-projection/) |
| 09 | [**默认 JSONL：写入、压缩、恢复与迁移**](notes/09-jsonl-session-persistence.md) | 读写句柄、懒创建、Zstd 帧、写者租约、尾部恢复、格式版本链与独占发布。追问：进程写到半帧、磁盘写满和迁移未发布时，各自应从哪一份数据恢复？[JSONL 实现](deepseek-harness/packages/session/session-persistence-jsonl/src/) · [持久化契约](deepseek-harness/packages/session/session-persistence/src/) · [版本状态](deepseek-harness/docs/session-format-status.md) |
| 10 | [**SQLite 会话库：WAL、事务与单写者**](notes/10-sqlite-session-persistence.md) | 读 `node:sqlite` 的表与访问路径，讲 Trace、Segment、Frame、父前缀共享、group commit、JSONL 首次导入和 schema 版本。追问：两个 Session 并发提交与同一个 Session 双写有何区别？提交结果未知时怎样安全重开？[SQLite 数据库实现](deepseek-harness/packages/session/session-persistence-sqlite/src/sqlite.ts) · [SQLite WAL](https://sqlite.org/wal.html) |
| 11 | [**业务存储：JSON、SQLite KV 与投影缓存**](notes/11-domain-storage-and-projection-cache.md) | 从 Storage Registry、命名空间、Domain Schema 到落盘、事务、Workspace 和 Projection Checkpoint。追问：业务记录写事务、投影缓存延迟刷盘和会话事件提交分别承诺什么？什么时候可以丢弃并重建缓存？[Storage](deepseek-harness/packages/storage/) · [Workspace](deepseek-harness/packages/workspace/workspace/) · [投影缓存](deepseek-harness/packages/session/session-projection-cache/) |
| 12 | [**会话全文检索：FTS5、实时与历史索引**](notes/12-session-full-text-search-fts5.md) | 搜索开关、惰性启动、内容提取、临时与持久索引、排序分页和重建；延伸到 BM25、中文检索、向量与混合召回。追问：索引比日志慢时如何定义读一致性？怎么证明搜索没吞掉最新消息？[Query](deepseek-harness/packages/session-query/session-query/src/) · [FTS5 实现](deepseek-harness/packages/session-query/session-query-sqlite/src/) |
| 13 | [**长上下文：Token 预算、裁剪、摘要与缓存**](notes/13-context-compaction-and-cache.md) | 跟读 token meter、普通工具结果裁剪、基础摘要、旧图卸载和保留历史。结合前缀缓存讲成本、上下文保真与分支变化。追问：模型摘要遗漏关键约束，怎样在后续步骤发现并补回？[压缩插件](deepseek-harness/packages/compaction/) · [Token Meter](deepseek-harness/packages/llm/token-meter/) |
| 14 | [**附件与跨会话记忆**](notes/14-attachments-and-cross-session-memory.md) | 走读二进制内容寻址、日志引用、Provider 图片上传、历史会话引用和大输出落盘。扩展到长期记忆的写入条件、召回策略、访问控制、保留与删除。追问：借用历史会话内容时，如何让来源、权限和删除传播都可追踪？[附件](deepseek-harness/packages/attachment/attachment-local/) · [跨会话引用](deepseek-harness/packages/context/session-reference/) · [输出落盘](deepseek-harness/packages/spill/) |

### 三、Skill、插件与 MCP（15–20）

| 篇 | 文章 | 内容与面试切入点 |
| --- | --- | --- |
| 15 | [**Skill 从 `SKILL.md` 到模型上下文**](notes/15-skill-loading-and-context.md) | 本地目录与单文件形式、frontmatter、项目/自定义/用户路径、重名顺序、模型与用户可见性、技能目录事件、`skill` 工具与 `/name` 调用。追问：目录里有 A，正文却加载失败，问题可能出在发现、作用域还是加载阶段？[文件系统 Provider](deepseek-harness/packages/skill/skill-filesystem/src/index.ts) · [Skill Registry](deepseek-harness/packages/skill/skill/src/index.ts) · [技能加载](deepseek-harness/packages/skill/tool-skill/src/index.ts) |
| 16 | [**从 10 个 Skill 管到 1 万个**](notes/16-skill-catalog-at-scale.md) | 从 DSH 的作用域、目录缓存和不完整发现出发，拆解 Office 的按需加载与连接器的固定版本分发，再设计团队目录的权限过滤、候选召回、版本发布和质量回归。只给模型 20 条摘要，怎样用 Recall@20、误触发率和任务成功率定位问题？[Office Skill](deepseek-harness/packages/skill/skill-office/) · [Upstream Skills](deepseek-harness/packages/connector/upstream-skills/) |
| 17 | [**插件市场、Preset 与生命周期治理**](notes/17-plugin-market-and-preset-lifecycle.md) | 从市场索引、pnpm 安装和 Profile HMR 走到 Preset 挂载、代际回收与会话恢复。追问：升级后新会话用新版、旧会话继续工作，怎样限制两套版本的资源占用？[Plugin Market](deepseek-harness/packages/boot/plugin-market/src/) · [Plugin Manager](deepseek-harness/packages/boot/plugin-manager/src/) · [Preset Registry](deepseek-harness/packages/preset/agent-preset-registry/src/) |
| 18 | [**MCP 接入实战：传输、发现与调用**](notes/18-mcp-transport-discovery-and-invocation.md) | 从只读服务配置走到模型工具声明，拆解 stdio 与 Streamable HTTP、现代能力发现与旧版 `initialize`、工具注册和结果适配。用资源接口对照工具与 Prompt Templates。追问：服务已连接，为什么模型看不到工具？[MCP Transport](deepseek-harness/packages/mcp/mcp-client/src/transport.ts) · [工具适配](deepseek-harness/packages/mcp/mcp-client/src/tools.ts) · [MCP 版本协商规范](https://modelcontextprotocol.io/specification/2026-07-28/basic/versioning) |
| 19 | [**MCP 动态管理：按需加载、重连、OAuth 与权限**](notes/19-mcp-dynamic-management.md) | 从配置对账、探测和状态流追到 Agent 按需加载，再拆解目录发布、连接代际、OAuth 授权与执行权限。追问：新目录已发布，旧连接上尚未完成的请求怎样判定归属和结果？[MCP Manager](deepseek-harness/packages/mcp/mcp-manager/src/) · [连接实现](deepseek-harness/packages/mcp/mcp-client/src/connection.ts) · [按需工具](deepseek-harness/packages/mcp/mcp-client/src/tool-loading.ts) · [MCP Resources](deepseek-harness/packages/mcp/mcp-resources/) |
| 20 | [**何时选 Skill、原生 Tool、MCP 或业务 Connector**](notes/20-skill-tool-mcp-and-connector.md) | 沿企业文档读取和报告交付对照四类能力，走读钉钉、飞书、企微和腾讯文档的接入路径。追问：同一能力从本地迁到远程时，怎样保持参数、身份、审计与交付物契约？[Connector](deepseek-harness/packages/connector/) · [Authorization](deepseek-harness/packages/credentials/authorization/) · [文件交付](deepseek-harness/packages/deliverables/tool-present/) |

### 四、执行环境、权限与任务调度（21–27）

| 篇 | 文章 | 内容与面试切入点 |
| --- | --- | --- |
| 21 | **Coding Agent 的代码感知与编辑** | 文件读写、ripgrep 搜索、结构化替换与 Patch、诊断与 LSP，结合 Git diff 和测试形成验证闭环。追问：模型生成 patch 后读到 stale 文件，如何避免覆盖并发编辑？[FS](deepseek-harness/packages/fs/) · [LSP](deepseek-harness/packages/lsp/) · [ripgrep](deepseek-harness/packages/util/ripgrep/) |
| 22 | **Shell、子进程、PTY 与 SSH** | 区分单次命令、持久终端和后台任务，讲 stdout/stderr、PTY、进程生命周期、取消、环境变量和远程执行。追问：关掉 Agent 时如何验证进程已清理？如何让输出消费慢时仍能看到关键日志？[Shell](deepseek-harness/packages/shell/) · [Subprocess](deepseek-harness/packages/subprocess/) · [Terminal](deepseek-harness/packages/terminal/) · [SSH](deepseek-harness/packages/ssh/) |
| 23 | **沙箱、审批与凭据** | 从文件/进程沙箱、命令授权、权限预设、签入凭据到审批记录，分析文件内容和工具结果作为不可信输入时的防护。追问：用户同意一个动作后，参数、身份、工作目录或策略发生变化，原授权还能否复用？[Sandbox](deepseek-harness/packages/sandbox/) · [审批](deepseek-harness/packages/interaction/user-approval/) · [Credentials](deepseek-harness/packages/credentials/) |
| 24 | **Plan、Todo、Goal 与长任务控制** | 对比计划模式、任务列表、目标驱动多轮运行、人工提问、连续重复提醒与超时策略。追问：Session 从断点恢复后，如何确定续跑条件、已消耗轮数和用户接管状态？[Plan](deepseek-harness/packages/plan/) · [Todo](deepseek-harness/packages/todo/) · [Goal](deepseek-harness/packages/goal/) · [Guard](deepseek-harness/packages/guard/) |
| 25 | **Subagent、Team 与 Workflow** | 分析 spawn、fork 与外部 Provider，父子上下文、活跃树、取消传播、共享任务板的 CAS 与 workflow 并行。追问：三名子 Agent 共用 checkout 时，怎样定义任务所有权、文件写入责任和结果合并？[Subagent](deepseek-harness/packages/subagent/) · [Team](deepseek-harness/packages/team/) · [Workflow](deepseek-harness/packages/workflow/) |
| 26 | **本地 Job、Cron 与投递语义** | 跟读进程内 Jobs 的限流、输出保留、取消与退出，以及 Schedule 的任务状态、到期、投递回执和恢复。追问：会话 inbox 已刷盘、任务尚未确认时崩溃，如何处理重复投递、漏跑和时区变更？[Jobs Local](deepseek-harness/packages/jobs/jobs-local/src/) · [Schedule Runtime](deepseek-harness/packages/schedule/schedule/src/runtime.ts) |
| 27 | **Browser Use、Computer Use 与办公交付** | 对照 DOM/语义描述、结构化应用接口、截图与输入事件，跟踪支持状态，讲 Office 产物的渲染、校验和交付。追问：可靠性、延迟、可审计性和权限各会把你推向哪种执行方式？[Browser Use](deepseek-harness/packages/browser-use/) · [Computer Use](deepseek-harness/packages/computer-use/) · [实验能力](deepseek-harness/packages/experimental/README.md) · [交付物](deepseek-harness/packages/deliverables/) |

### 五、产品接口、可观测性与评测（28–30）

| 篇 | 文章 | 内容与面试切入点 |
| --- | --- | --- |
| 28 | **Web、Desktop、SDK、ACP 与断线重连** | Web Host/Client 插件树、Typert RPC、多路流、Session Follow、Electron 壳与 IPC、TS/Python SDK 的 JSON-RPC 和 ACP。追问：客户端离线十秒后如何恢复已提交历史与当前生成，如何避免重复显示同一个片段？[Web](deepseek-harness/apps/web/) · [Desktop](deepseek-harness/apps/desktop/) · [Session Controller](deepseek-harness/packages/api/session-controller/) · [SDK](deepseek-harness/packages/sdk/) · [ACP](deepseek-harness/packages/acp/) |
| 29 | **可观测性、Token 成本与性能诊断** | 把 Session、turn、step、attempt、tool 串起来，区分首字延迟、排队、重试、工具耗时和端到端时延。用 token usage、长会话和重连基准制定容量验证。追问：P95 翻倍时如何区分 Provider 变慢、工具饱和、本地 IO 与 UI 渲染？[Session Telemetry](deepseek-harness/packages/session/session-telemetry-otel/) · [OTel](deepseek-harness/packages/telemetry/otel/) · [Benchmarks](deepseek-harness/benchmarks/) |
| 30 | **Agent 评测：回放、任务成功率与发布回归** | 比较 Unit、契约、Snapshot、真实 API E2E，设计代码任务通过率、工具误调用、成本和人工介入等指标。练习配对任务、重复试验、环境归因与失败样本回归。追问：成功率升了 3 个点、成本涨了 40%，怎样判断收益是否真实且值得上线？[测试策略](deepseek-harness/docs/testing.zh.md) · [会话 Snapshot](deepseek-harness/snapshots/session/) |

### 六、从 DSH 出发做系统设计（31–38）

| 篇 | 文章 | 设计题、产出与压力条件 |
| --- | --- | --- |
| 31 | **用 Redis 和 PostgreSQL 设计云端 Agent 作业系统** | 给出 Run/Attempt/Step 状态机、PostgreSQL 持久记录与 Outbox，比较 Redis Pub/Sub、Streams 和有序集合的适用场景。讨论限流、缓存、ACK、重领、续租、fencing token、幂等和死信处理。压力条件为外部副作用已经发生，Worker 在确认前退出。验证如何安全恢复，以及 Redis 故障时怎样处理积压。[Redis Streams](https://redis.io/docs/latest/develop/data-types/streams/) |
| 32 | **从零设计一个最小 DSH** | 设计 Agent Loop、Model Adapter、Tool Registry、Session Store、Policy 与 Transport 的接口及事件顺序。产出一个最小会话时序和演进路线，从单进程 CLI 扩到多入口。压力条件：模型中途取消和未知工具结果都要能恢复与解释。 |
| 33 | **设计一个云端 Coding Agent** | 从接收仓库、理解 issue、编辑、执行测试到提交供评审的变更。设计隔离 workspace、Git worktree、搜索与 LSP、测试证据、Human-in-the-loop 和配额。压力条件：大仓库、多任务并发、依赖安装失败、用户同时修改同一文件。 |
| 34 | **设计一个企业办公助理，WorkBuddy 类场景** | 用「读群消息 → 找文档 → 生成报告 → 经确认发出」贯穿账户连接、企业 ACL、草稿、审批、交付物、消息通知和定时任务。压力条件：成员离职、权限撤回、生成后接收人变更，仍要验证当前授权与动作参数。 |
| 35 | **设计多租户 Skill／MCP 平台** | 以 1 万技能、100 个 MCP 服务为题设，画出发布面与执行面，设计版本快照、召回、租户隔离、按需连接、授权、灰度、审计和质量回归。压力条件：插件吊销或升级时，已有工作如何收尾、后续请求何时切换？ |
| 36 | **设计会话与企业知识检索** | 定义 Session 事件、派生索引、对象存储、全文与向量检索的职责，设计增量消费、ACL 过滤、删除传播、重建与召回评估。压力条件：历史索引落后，权限和删除更新仍要按预期生效。 |
| 37 | **设计高并发在线 Agent 服务** | 拆分 API Gateway、调度、执行 Worker、模型网关与事件推送，设计租户配额、背压、取消、断线游标与资源规划。用 1,000 条活跃流作为面试题设，估算连接、模型 RPM/TPM、持久化写入与外部工具并发。 |
| 38 | **综合模拟面试与故障推演** | 准备三套模拟题，分别围绕 Harness 内核、Coding Agent、企业 LLM 应用。每套包含源码开场、现场推演、架构图、成本取舍与故障验证。产出可口述的 90 秒项目介绍、大约 10 分钟的原理解读和 45 分钟的系统设计答题结构。 |

## 面试官怎么逐层追问

后续每篇会附源码起点、由浅入深的追问、答题要点和一个可验证场景。回答可按「具体调用 → 数据或状态 → 失败边界 → 取舍与估算 → 验证方法」展开。

| 主线 | 从 DSH 开始 | 深入一层 | 现场设计与判分点 |
| --- | --- | --- | --- |
| Agent Loop | DSH 在哪里决定发下一次模型请求？ | `tool/result`、取消、失败 attempt 和 `turn/end` 各在什么时候落日志？ | 模型流与本地写入之间崩溃时，能否指出权威记录、重放边界，以及是否会再次执行有副作用的工具？ |
| SQLite / Redis | 默认会话持久化是什么？三个 SQLite 用途各是什么？ | WAL 下读写怎么并发？写者租约和 DB 事务分别保护什么？索引落后如何重建？ | 切成多 Worker 后，把完成状态、热点缓存、调度分别放哪？能否处理 ACK 丢失、重复消费、租约过期和未知提交结果？ |
| Skill | `SKILL.md` 怎么成为模型可调用的 Skill？ | 多 Provider 同名、用户与模型可见性、目录缓存和正文更新怎么处理？ | 1 万个技能里只给模型 20 个候选，如何做权限过滤、版本锁定、召回实验和质量回滚？ |
| MCP | DSH 如何把外部 Server 的工具接进 `ctx.tools`？ | `tools/list` 变更、重连、OAuth grant 失效和取消分别由何处处理？ | 远程写操作超时后，候选人能否区分连接恢复、调用重试、幂等和对账？能否约束租户与会话间的凭据使用？ |
| 代码执行 | 文件工具、Shell、PTY、沙箱和审批分别负责什么？ | 并行读写、进程终止、测试失败、危险操作待确认时，如何呈现结果？ | 两个 Agent 对同一仓库提修复，能否交代工作目录、并发编辑、验证与回滚，并说明能观察到的证据？ |
| 办公助理 | DSH 如何组合 Connector、Skill、模型、工具和交付物？ | 文档授权、对话历史和模型输入分别在哪里校验与留痕？ | 从读取文档到发送报告，能否把阅读权限、目标身份、发送确认、异步失败恢复和审计设计完整？ |

## 阅读路线与后续写作规格

- **通用 Agent / LLM 应用**：01 → 03 → 04 → 06 → 08 → 13 → 18 → 28 → 30 → 32。
- **Coding Agent**：01 → 06 → 07 → 21 → 22 → 23 → 24 → 25 → 33。
- **企业办公 Agent**：15 → 16 → 18 → 19 → 20 → 23 → 26 → 27 → 34。
- **云平台和基础设施**：08 → 09 → 10 → 11 → 12 → 26 → 31 → 35 → 37。

后续以 `notes/<篇号>-<主题>.md` 编号落稿，完成一篇便在本目录补上链接。源码篇聚焦实现原理、关键调用链、失败案例和面试追问，按需要在正文引用源码、配置或测试证据。涉及可选插件时，明确启用条件与生效的 Profile。系统设计篇给出题设、规模估算、API、数据模型、时序、故障恢复、容量取舍和验证指标。
