# 16｜从 10 个 Skill 管到 1 万个

> 源码基线为 DeepSeek Harness `dsh-v0.2.3-alpha.2`，提交 `4b399c39a6d1479cd84ca591b4d7c88e24d38a96`。[系列目录](../README.md) · [上一篇](15-skill-loading-and-context.md) · [上游仓库](https://github.com/deepseek-ai/deepseek-harness)

上一篇追到 `SKILL.md` 正文进入上下文。这一篇从两个例子继续。一份 `office` 指南覆盖了三种文档类型和不同的运行环境，企业微信的入口技能则引用了固定提交里的多份指南。两者都只占目录里一个位置。规模再增长，宿主要花多少时间发现技能？模型又该看到多少可选技能？[Office Provider](../deepseek-harness/packages/skill/skill-office/README.zh.md) · [企业微信的技能定义](../deepseek-harness/packages/connector/wecom/src/skills.ts#L7)

## 1. DSH 如何缓存目录和处理局部失败

`SkillRegistry.collect()` 用当前工作目录 `cwd`、完整作用域链和注册表 `revision` 做缓存键。合并时先收集全局层，再从远到近处理祖先，最后由最近一层覆盖同名项。同层优先级由 rank 和确定性的顺序规则决定。复用一个空 Session 的作用域对象时，如果它换了 Preset 并接到新的父作用域，链变了就会重新收集。默认最多保留 128 份完整目录，超过上限时淘汰最早插入的缓存项。一份目录里可以有很多技能。[缓存键与淘汰](../deepseek-harness/packages/skill/skill/src/index.ts#L522) · [跨作用域合并](../deepseek-harness/packages/skill/skill/src/index.ts#L554) · [默认容量](../deepseek-harness/packages/skill/skill/src/index.ts#L25)

| 发生什么 | 注册表如何处理 |
| --- | --- |
| 文件系统 Provider 的某个根目录发现失败 | 保留其他根里的候选，将整次观察标为不完整。 |
| 一个 Provider 的 `list()` 抛普通错误 | 记录告警并继续收集其他 Provider，当前结果不入缓存。取消信号则中止收集。 |
| 收集过程中目录发生变更 | 发现 `revision` 已变化后重收集一次，第二次仍遇到变化就返回不完整结果。 |
| 文件变更触发 Provider 失效 | 提升 `revision`、清空目录缓存，并通过 `skills/change` 通知订阅者。 |

文件系统 Provider 还单独限制观察的项目数，默认 128 个。项目观察与目录缓存分别消耗 watcher 和内存。现有快照里，目录带 `incomplete: true`，同一轮仍成功加载了可见的 `editing-cordis-compositions`。如果此时查不到另一个技能，只能确认这次发现没找到它。[文件系统的部分返回](../deepseek-harness/packages/skill/skill-filesystem/src/index.ts#L186) · [观察项目上限](../deepseek-harness/packages/skill/skill-filesystem/src/index.ts#L36) · [注册表的失败分支](../deepseek-harness/packages/skill/skill/src/index.ts#L587) · [目录快照](../deepseek-harness/snapshots/session/skill-root-isolation/session.v4.jsonl#L12) · [加载结果](../deepseek-harness/snapshots/session/skill-root-isolation/session.v4.jsonl#L18)

扩大目录时，要分别算发现开销和模型开销。冷发现会枚举根目录、读取文件并解析元数据，注册表也要逐个等待 Provider 返回。缓存命中可以省掉这些发现工作。到 `agent/pre-step`，`tool-skill` 仍会给模型准备当前作用域内所有可由模型调用且未被排除的摘要。每条描述默认最多 500 个字符，摘要内容或完整性变化时会发布替换目录。这个 digest 标识的是发送给模型的目录内容，编辑技能正文但保持描述不变时，目录 digest 可以不变。[Provider 收集](../deepseek-harness/packages/skill/skill/src/index.ts#L602) · [描述长度](../deepseek-harness/packages/skill/tool-skill/src/index.ts#L29) · [目录注入](../deepseek-harness/packages/skill/tool-skill/src/index.ts#L226) · [目录 digest](../deepseek-harness/packages/skill/tool-skill/src/index.ts#L356)

## 2. Office 怎样用一个入口覆盖三种文档

`skill-office` 注册 `dsh-office` Provider，目录只暴露一个 `office` 候选。发现时读打包的 `office/SKILL.md` 并提取摘要。加载时重新读取正文，再按宿主配置追加 Python 环境说明。模型在目录里只需判断任务是否涉及 Office，加载后再按文档类型选择操作流程。例如，改 PPTX 要考虑文字溢出，写 XLSX 公式要区分公式本身和计算结果，两条工作流都可以从同一入口展开。[Office Provider 的 list 与 get](../deepseek-harness/packages/skill/skill-office/src/index.ts#L110) · [Office 技能正文](../deepseek-harness/packages/skill/skill-office/assets/office/SKILL.md)

宿主挂载 `tool-office` 时，它监听成功的 `skill` 工具结果和用户的 `skill-invocation` 事件，在 Agent 上记录激活状态。下一步可以声明 `office_render`、`office_convert` 和 `office_recalculate`。工具执行仍需运行时的 `officeKit` 服务，并按输入文件检查权限。正常快照第 13 行的请求头没有这三个工具，第 18 行加载 `office`，第 21 行的新请求头声明了它们。另一份快照同样加载了正文，但工具清单缺少渲染能力，模型给出的流程降级为静态检查。目录设计还要考虑当前 Agent 能执行到工作流的哪一步。[条件激活](../deepseek-harness/packages/document/tool-office/src/index.ts#L147) · [执行前检查](../deepseek-harness/packages/document/tool-office/src/index.ts#L230) · [工具声明快照](../deepseek-harness/snapshots/session/office-skills/session.v4.jsonl#L21) · [缺少渲染器的快照](../deepseek-harness/snapshots/session/office-skills-no-renderer/session.v4.jsonl#L21)

## 3. 上游技能怎样固定版本和发布

企业微信和腾讯会议的连接器调用 `ctx.upstreamSkills.register()`，传入自身打包的入口文件和上游资料清单。以企业微信为例，`WECOM_SKILLS` 声明仓库、Git 提交、保留目录、排除项、许可证、文件字节上限和 SHA-256。入口文件用仓库相对路径索引更细的指南，Provider 把已安装的资料目录设为 `resourceBase`。模型选中连接器入口后，再按任务读具体指南。目录、正文、引用资源的分层也对应 Agent Skills 规范中的渐进式披露。[企业微信注册](../deepseek-harness/packages/connector/wecom/src/index.ts#L49) · [固定的资料清单](../deepseek-harness/packages/connector/wecom/src/skills.ts#L10) · [腾讯会议注册](../deepseek-harness/packages/connector/tencent-meeting/src/index.ts#L49) · [Agent Skills 规范](https://agentskills.io/specification)

1. **下载与检查。** 插件加载后发起下载，依次尝试配置的归档源。解包时按清单选文件，检查路径、类型和保留文件的累计大小，再按排序后的路径和内容计算 SHA-256。同一批文件即使被重新打包，也可以匹配固定摘要。[归档选择与验证](../deepseek-harness/packages/connector/upstream-skills/src/archive.ts#L15) · [下载源与摘要校验](../deepseek-harness/packages/connector/upstream-skills/src/archive.ts#L156)
2. **安装与目录更新。** 安装先写同级临时目录，校验和写入完成后用 `rename` 发布。目标目录存在才会列出入口，下载期间或失败且没有可用目录时，Provider 返回空列表。成功发布后使 Skill 目录失效，下一次发现可以看到入口。配置本地 `skillsDir` 时直接使用该目录，注册时检查它是否存在且为目录，跳过归档下载及其摘要校验。[本地目录和候选](../deepseek-harness/packages/connector/upstream-skills/src/index.ts#L121) · [可见性与安装状态](../deepseek-harness/packages/connector/upstream-skills/src/index.ts#L150) · [暂存与发布](../deepseek-harness/packages/connector/upstream-skills/src/index.ts#L221)
3. **旧版本清理。** 同一 Profile 的连接器进程共享自己的资料目录，当前 pin 安装后，注册逻辑会尝试清理其他 pin。升级后如果旧进程还在跑，它原来引用的资料可能已经被删掉。没有 Profile 时，多个 pin 可以共存于连接器的缓存目录。固定上游提交解决了安装时应取哪份资料的问题，旧会话的资料保留则取决于宿主的清理策略。[Profile 目录选择](../deepseek-harness/packages/connector/upstream-skills/src/index.ts#L121) · [旧 pin 清理](../deepseek-harness/packages/connector/upstream-skills/src/index.ts#L191)

## 4. 系统设计题｜企业目录怎样只给模型看 20 条

假设每条摘要连同名称平均占 40 token，1 万条约占 40 万 token，20 条约占 800 token。实际还要计算目录包装、对话历史、工具定义和加载后的正文。DSH 已经把发现、摘要和正文拆开，企业目录可以沿用这个接口，把整表列举改成任务级候选检索。

```mermaid
flowchart LR
    A["用户任务与显式技能名"] --> B["限定租户、Preset 和可访问范围"]
    B --> C["名称、别名与任务描述召回"]
    C --> D["排序并按条数和 token 裁剪"]
    D --> E["模型选择候选"]
    E --> F["按版本加载并复核当前权限"]
```

目录元数据可以放在关系数据库，以 `(tenant_id, skill_id, version)` 标识版本。保存可读名称与别名、维护者、分类、调用方式、依赖能力、发布状态和内容摘要。正文与引用资料使用不可变制品存储，检索索引只存召回和展示所需的元数据。租户与职能权限、Agent 的 Preset、技能是否允许模型调用，都是自动召回前的硬过滤条件。这样隐藏的名称和摘要不会出现在模型提示或目录搜索结果里。

先匹配可见的精确名称，再用关键词检索覆盖任务描述和别名。对长尾自然语言任务，可以补语义检索，再按任务匹配度和当前可用能力排序。比如「把会议纪要发到项目群」可以召回整理纪要与发送消息两类技能，进入发送工具时还须单独检查目标会话与用户授权。同名候选按稳定 ID 和作用域消歧，同一技能的多版本与别名只占一个位置。候选都低于阈值时可以返回空列表。上限取 20，同时用总 token 预算约束实际返回量。

用户显式调用和模型自动选择分别检查各自的调用策略。用户手动指定一个允许显式调用的技能时，可以绕过自动候选的排名，仍须通过同样的访问检查。模型发起加载时，服务端用本轮发出的候选记录定位 `skill_id`、版本和摘要，并重新确认权限、发布状态与撤回标记。这样即使模型在历史消息里看到一个过去可见的名字，当前已被撤权时也无法沿用旧候选拿到正文。

## 5. 发布、回滚和质量回归

**发布时固定制品和索引版本。** 先验收入口格式、引用资料、所声明的工具依赖和跨版本兼容性，再发布不可变制品及对应索引。按团队或租户灰度切换索引版本，监控加载成功率、任务成功率和工具错误，再扩大范围。任务开始时记录目录版本和实际选中的技能版本，回滚时把新任务入口切回旧索引。仍有活跃任务引用的制品需要保留，可用带过期时间的租约控制回收。紧急撤回由在线策略拒绝后续加载与敏感工具调用。历史中已经读过的内容仍可能影响模型，因此执行端也要复核。

**把召回、触发和执行分开量。** 每个任务记录索引版本、权限过滤后候选数、进模型的候选 ID 和版本、实际加载结果、工具错误及最终任务结果。相同任务、模型和工具权限下比较版本，才能判断是检索变化还是运行环境变化。

| 指标 | 怎么算 | 变差时先查哪里 |
| --- | --- | --- |
| `Recall@20` | 对每个标有至少一个适用技能的任务，用前 20 条命中的适用技能数除以该任务全部适用技能数，再对任务取平均。适用集合以该任务有权使用的技能为准。 | 技能描述、别名、过滤条件和索引更新。 |
| 误触发率 | 在无需 Skill 的任务中，发生至少一次技能加载的任务数，占这类任务的比例。 | 召回阈值、描述过宽、模型选择提示。 |
| 技能加载成功率 | 成功加载次数除以请求加载次数，按撤权、制品缺失等原因分别看。 | 发布索引与制品是否同步、版本回收。 |
| 任务成功率 | 满足交付、内容和授权要求的任务数除以评估任务总数。 | 结合候选与调用记录定位，检查命中后有没有选对、选对后有没有执行成功。 |

如果给定的一组任务有多个可替代技能，还可以看 `Hit@20`，即至少命中一个可用技能的任务比例。业务只需选中一个时，这个指标更贴近真实决策。遇到需要多个技能共同完成的任务，应另看全部必需能力是否都覆盖。如果 `Recall@20` 升了，任务成功率却降了，就从已召回样本里查看模型是否选错、权限是否变化，以及相同的版本在依赖环节有没有发生降级。

## 6. 面试官怎么逐层追问

### DSH 缓存了 128 项，意味着一个 Agent 最多用 128 个 Skill 吗？

答的是目录视图的容量。一个视图对应 `cwd + Scope 链 + revision`，可以含许多候选。接着追问某个目录临时不可读会怎样。能解释「可用候选继续用、不完整观察不入缓存、下一次重收集」，也要指出目录命中只解决发现延迟，模型仍需处理收到的摘要。

### 只把 20 条摘要给模型，怎样判断这 20 条够用？

先明确 20 是接口上限还是从 token 预算推出来的。然后区分有标注任务的召回率、无 Skill 任务的误触发率以及实际交付成功率。继续追问 `Recall@20` 高但成功率下降的案例。可以沿同一任务逐项核对排序、模型选择、技能版本和工具结果，判断哪一步开始偏离预期。

### 新版连接器上线后，几个正在跑的旧任务应该怎么办？

先用 DSH 的企业微信入口讲明白提交、制品摘要和本地 pin 的关系，再指出同一 Profile 的清理规则为何可能让旧进程失去资料。生产方案给任务绑定不可变版本，灰度与回滚控制新任务分流，租约控制旧制品回收。面试官再问紧急撤回时，就要区分允许旧任务复读已经加载的说明和允许它继续调用发送工具，两者受不同的检查约束。

### 权限在召回前已经过滤过，为什么加载与工具层还要再查？

召回过滤保护摘要，加载复查保护正文，工具执行授权约束具体动作。三个检查之间，租户角色、技能发布状态和目标资源的访问权限都可能改变。Office 的例子说明，工作流说明可以成功加载，实际渲染依然取决于当前 Agent 的工具、服务和文件权限。

下一篇进入插件与 Preset 的生命周期，继续追一组 Provider、工具和配置怎样一起装载、升级和退出。
