# 20｜何时选 Skill、原生 Tool、MCP 或业务 Connector

> 源码基线为 DeepSeek Harness `dsh-v0.2.3-alpha.2`，提交 `4b399c39a6d1479cd84ca591b4d7c88e24d38a96`。[系列目录](../README.md) · [上一篇](19-mcp-dynamic-management.md) · [上游仓库](https://github.com/deepseek-ai/deepseek-harness)

用户说「读取项目周报，做一份 Word 汇总给我」。这条任务需要找到源文档、用正确的账号读取、按业务要求写报告，再把文件交付给用户。DSH 能把这些职责放到不同层里。同样是企业应用接入，飞书插件把模型引向厂商 CLI，腾讯文档插件则通过 MCP 暴露工具。两条路径最后都可以把读到的材料交给本地文档工具处理。[连接器分组](../deepseek-harness/packages/connector/README.zh.md#packages)

## 1. 四种能力各自解决什么问题

| 能力 | 模型或宿主拿到什么 | 适用场景 |
| --- | --- | --- |
| Skill | 按需加载的工作说明、选择条件与参考资料。 | 周报如何分段、怎样引用来源、Word 文档修改后检查什么。 |
| 原生 Tool | 在 DSH 工具注册表中声明名称、参数、结果与执行函数，可以访问会话和运行时服务。 | 读本地文件、渲染文档或登记交付，尤其需要文件系统与 Session 上下文时。 |
| MCP | 用通用协议发现并调用外部能力，DSH 将远端工具和资源接入工具运行时。 | 多个 Host 复用同一套服务，或服务由另一个团队独立部署和升级。 |
| 业务 Connector | 面向一个产品的宿主插件，组合执行入口、账号状态、配置及模型指引。 | 用户在插件页连接账号，知道授权失败时怎么恢复，并能在对话里使用厂商能力。 |

四种能力可以组合。Skill 负责把工作方法给模型，工具承担具体执行，MCP 可以承载工具的远端实现，Connector 把服务和账号做成用户可管理的入口。比如飞书 Connector 同时注册 `feishu` Skill 和 CLI 账号。腾讯文档 Connector 则注册 MCP 服务器、系统提示词和账号区块。选型前先问缺的是工作方法、执行能力、协议适配，还是账号接入。[Skill 注册表](../deepseek-harness/packages/skill/skill/README.zh.md) · [Tool 注册表](../deepseek-harness/packages/core/tools/src/index.ts#L136) · [MCP 工具桥接](../deepseek-harness/packages/mcp/mcp-client/src/tools.ts#L144) · [连接器分组](../deepseek-harness/packages/connector/README.zh.md#packages)

## 2. 钉钉、飞书、企微为什么都要一个入口 Skill

这三种插件都把官方 CLI 的绝对路径加入运行时上下文，使 Agent 和插件页使用同一套受管理的可执行文件。`cliAccounts.register()` 还给插件页接入账号状态和操作入口。每个账号运行时同时只接受一个账号操作，厂商插件提供自己的状态解析、登录步骤和授权页面规则。账号操作通过宿主 `subprocess` 执行。模型读文档等业务操作，则按 Skill 给出的说明调用普通 Shell 工具，受当前 Agent 的执行权限约束。[CLI 账号注册](../deepseek-harness/packages/connector/cli-account/src/index.ts#L62) · [路径上下文](../deepseek-harness/packages/connector/cli-account/src/context.ts#L22) · [账号操作串行化](../deepseek-harness/packages/connector/cli-account/src/runtime.ts#L96) · [飞书的任务执行路径](../deepseek-harness/packages/connector/feishu/README.zh.md#use-this-package)

| Connector | 执行入口与指南来源 | 账号接入 |
| --- | --- | --- |
| 钉钉 | `dws`。从固定 release 准备二进制及上游指南，校验后发布。入口资料作为单独的文件系统 Provider 挂载，启用 watcher。 | 插件页发起 CLI 浏览器登录。读取 CLI 状态时区分已连接、已退出与无法验证。 |
| 飞书 | `lark-cli`。入口 `feishu/SKILL.md` 随插件打包，Provider 在加载时读一次，子指南留在 CLI 中，由模型调用 `skills read` 取用。 | 插件页分别处理应用关联和用户登录。CLI 默认配置与用户自己的终端共用。 |
| 企业微信 | `wecom-cli`。入口在插件包内，子指南来自固定上游提交，通过 `upstreamSkills` 下载校验。 | 插件页走 CLI 扫码绑定机器人，并用 `auth show` 确认授权。 |

钉钉的路径提示受 release 准备状态约束，飞书的入口 Provider 读取时不会启动 CLI，企微则把指南是否安装也反映在账号视图里。飞书会话快照展示了入口的作用。Agent 先加载 `feishu`，再用运行时给出的 `lark-cli` 读 `lark-doc` 子指南。这个过程消耗了两个工具步骤，目录仍然只需要一个 `feishu` 条目。[钉钉装配](../deepseek-harness/packages/connector/dingtalk/src/index.ts#L69) · [钉钉发布校验](../deepseek-harness/packages/connector/dingtalk/src/release.ts#L153) · [飞书 Provider](../deepseek-harness/packages/connector/feishu/src/skills.ts#L14) · [企微装配](../deepseek-harness/packages/connector/wecom/src/index.ts#L47) · [企微账号流程](../deepseek-harness/packages/connector/wecom/src/account.ts#L42) · [飞书加载入口](../deepseek-harness/snapshots/session/feishu-skills/session.v4.jsonl#L17) · [飞书读子指南](../deepseek-harness/snapshots/session/feishu-skills/session.v4.jsonl#L22)

CLI 账号状态给用户看，业务请求仍由厂商服务检查资源权限。用户能在插件页看到「已连接」，Agent 仍可能因为组织权限、沙箱限制或文档不存在而读不到指定内容。源码里飞书只有经过服务端核验才发布 `connected`，无法核验时会保留 `unverified`。账号操作失败时，共享 CLI 运行时记录操作阶段、命令标识和退出码，避免直接记录可能带凭据的原始异常和输出。[飞书状态解析](../deepseek-harness/packages/connector/feishu/src/account.ts#L101) · [账号错误处理](../deepseek-harness/packages/connector/cli-account/src/runtime.ts#L175)

## 3. 腾讯文档怎样用 MCP 完成同类接入

腾讯文档插件的 `apply()` 把 `tencent-docs` 注册到共用的 `mcpManager`，使用 Streamable HTTP，默认 `on-demand`。固定服务器名稳定了模型侧命名空间，固定的 `mcp-client/tencent-docs` key 允许在配置仍匹配时复用已保存的授权。插件还在系统提示词中提供称呼和连接指引，并把服务注册到 `mcpAccounts`，由账号页面提供连接、退出和重试入口。[腾讯文档注册](../deepseek-harness/packages/connector/tencent-docs/src/index.ts#L19) · [账号操作分派](../deepseek-harness/packages/mcp/mcp-account/src/account.ts#L88)

MCP OAuth 用 `ctx.authorization` 管理按凭据 key 发起的授权尝试。OAuth 适配器注册 flow，浏览器授权完成后向 `AuthorizationSession.commit()` 提交 grant。通用授权服务要求观察到本次确实写入凭据，才报告 `authorized`。飞书、钉钉和企微的账号操作则由厂商 CLI 管理其本地凭据，DSH 用 CLI 观察登录结果。这两类连接器都能在插件页显示状态，账号归属和持久化机制各有自己的实现。[MCP OAuth flow](../deepseek-harness/packages/mcp/mcp-client/src/oauth.ts#L76) · [授权提交](../deepseek-harness/packages/credentials/authorization/src/index.ts#L381) · [CLI 账号运行时](../deepseek-harness/packages/connector/cli-account/src/index.ts#L62)

腾讯文档没有可用 grant 时，连接处于等待授权。此时系统提示词仍可以展示插件称呼及连接引导。授权并连接成功后，`mcp__tencent-docs__load_tools` 才供模型按需选择服务工具。仓库中未授权的快照有腾讯文档指引与共享资源工具提示，预期工具清单里没有该服务器命名空间下的业务工具或加载器。页面能显示入口、服务已连接和具体工具已加载，是三个可分别检查的状态。[腾讯文档授权说明](../deepseek-harness/packages/connector/tencent-docs/README.zh.md#use-this-package) · [未授权的提示词快照](../deepseek-harness/snapshots/session/tencent-docs-plugin/system-prompt.expected.md#L35) · [预期工具声明](../deepseek-harness/snapshots/session/tencent-docs-plugin/tool-schemas.expected.json)

## 4. 从读取文档到交付报告怎样组合

```mermaid
flowchart LR
    A["选择资料来源并确认读取范围"] --> B["通过厂商 CLI 或 MCP 读取原文"]
    B --> C["保留来源和版本，按报告说明组织材料"]
    C --> D["生成工作区文件"]
    D --> E["复读文件或渲染检查"]
    E --> F["通过 present 登记为本轮交付物"]
```

**先读取资料。** 模型通过相应 Connector 的 Skill 或工具目录找到读取入口。客户端应保留读取所用的账号或机器人身份，报告应记录可追查的来源链接、文档 ID 或能取得的版本。读权限按具体文档检查。当远端只返回部分内容、缺页或权限不足，报告要标明缺失的材料范围。

**再生成并核验文件。** `office` Skill 为报告制作提供 Word、PPT、Excel 的工作方法。支持的组合挂载 `tool-office` 后，成功加载 Skill 可以激活相关原生工具。比如让文档保留可编辑的 Word 结构，生成后用 `office_render` 检查页内容或排版。工具会解析 Session 文件系统中的实际输入，检查文件可读性、`officeKit` 服务，以及渲染结果是否能以图片进入当前模型路由。文件写入与最终视觉检查是两次独立的操作。[Office 说明](../deepseek-harness/packages/skill/skill-office/README.zh.md) · [条件激活](../deepseek-harness/packages/document/tool-office/src/index.ts#L147) · [输入与服务检查](../deepseek-harness/packages/document/tool-office/src/index.ts#L230) · [图片路由](../deepseek-harness/packages/document/tool-office/src/index.ts#L310)

**最后登记交付。** `present` 接收已存在的普通文件，用 Session 工作目录解析相对路径。在调用成功后追加 `deliverables/presented`，记录本轮、调用 ID 和规范化路径。Web 可以在轮次结束后展示交付卡片，用户打开的是该路径当前的文件。若要求报告提交后长期复现，需要额外持久化不可变副本和版本身份。若用户还要求把报告发到外部群，发送是另一个业务动作，应该核对收件范围、文件版本和该次发送的授权。[文件验证与事件](../deepseek-harness/packages/deliverables/tool-present/src/index.ts#L78) · [交付卡片装配](../deepseek-harness/packages/bundle/web-app/cordis.patch.yml#L400) · [文件引用语义](../deepseek-harness/packages/deliverables/tool-present/README.zh.md#use-this-package)

## 5. 系统设计题｜把本地读文档工具迁成远程 MCP

假设旧工具叫 `read_company_doc`，Agent、历史记录与执行政策已有调用方。可以在宿主保留这个业务入口，用适配器切换底层执行。在迁移期间记录每次用了哪个后端。若决定让模型直接使用新的 MCP 名称，就要同步更新工具绑定、说明和执行规则。工具身份应该和部署位置解耦，才能在回滚时继续解析同一笔业务操作。

| 需要保持的契约 | 迁移时怎样处理 |
| --- | --- |
| 能力标识与数据契约 | 维持文档 ID 的解释、版本和分页语义，保留输出类型及字段。无法兼容的变更用另一个接口版本。 |
| 调用者与目标资源 | 记录调用的用户或机器人、租户、授权范围、目标文档。迁移后用与远端资源相绑定的授权，服务端对文档 ACL 重做检查。 |
| 执行与错误语义 | 对齐超时、取消、权限拒绝、部分结果和未知结果。写操作带稳定的业务操作 ID，跨后端重试时共用去重约定。 |
| 产物与审计 | 远端路径只在远端有效。用可验证的制品引用标识结果，按权限导入 Session 可访问的文件，再做交付。审计关联工具调用、后端、版本和制品标识。 |

可以先按租户切换只读流量，比较两种后端返回的文档身份、版本和正文边界，再逐步扩大。写操作只由当前选定的后端执行，通过单一业务操作记录恢复超时或中断的执行。回滚切换的是后续路由，已经提交的外部动作依旧按原记录核对。DSH 的 Office 转换在 Host 路径无法映射到 Session 文件系统时会直接报错，这个限制提供了一个具体例子，工具返回的文件路径需要先明确属于哪个文件系统。[Office 临时文件路径检查](../deepseek-harness/packages/document/tool-office/src/index.ts#L403)

## 6. 面试官怎么逐层追问

### 已经有飞书 Skill，为什么还要接入 CLI 或 MCP？

先说明 `SKILL.md` 告诉 Agent 何时选能力、怎么使用、失败后看什么。实际读取必须经 CLI 或服务接口完成。继续追问把所有说明都塞到工具描述会怎样。可以把稳定的参数契约留在 schema，复杂流程留给按需加载的 Skill，运行时路径和账号状态由宿主提供。

### 飞书和腾讯文档都能读企业文档，接入为什么不同？

从源码回答飞书由 `lark-cli` 执行，子指南由 CLI 提供，账号步骤也用 CLI。腾讯文档用宿主 MCP 管理器代管连接和 OAuth，模型按需加载业务工具。继续追问多租户部署，把终端共享默认账号、Host 的固定 grant key 和每次业务请求的身份区分开，说明哪里需要显式租户绑定与独立授权。

### 远端返回成功，能直接告诉用户报告已经交付吗？

先核对操作成功指的是读取、文件生成还是交付。生成阶段要确认文件存在并满足格式及内容要求，交付阶段应登记 Session 能打开的文件。追问用户还要发给同事时，检查目标身份、可见范围和当前批准的版本，再使用对应连接器执行发送。

### 旧工具迁成 MCP，参数和工具名都兼容就够了吗？

接着检查调用者、文档版本、授权范围和错误语义。服务可能响应超时，但写操作其实已提交。问候选人如何回滚，回答应包含稳定的业务操作 ID、服务端状态查询或幂等约定，以及新旧后端产生的制品能否在同一个会话里正确打开和追溯。

下一篇进入 Coding Agent，沿文件读取、搜索、编辑和补丁校验追到一次完整的代码修改。
