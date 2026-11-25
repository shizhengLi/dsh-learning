# 17｜插件市场、Preset 与生命周期治理

> 源码基线为 DeepSeek Harness `dsh-v0.2.3-alpha.2`，提交 `4b399c39a6d1479cd84ca591b4d7c88e24d38a96`。[系列目录](../README.md) · [上一篇](16-skill-catalog-at-scale.md) · [上游仓库](https://github.com/deepseek-ai/deepseek-harness)

用户在插件市场点击安装，新会话里还是找不到工具，旧会话里却能继续调用。查这个问题前，先确认用的是同一个 Profile、两个 Agent 绑定的 Preset，以及工具注册发生在哪个作用域。在 DSH 里，市场版本、磁盘依赖、配置选择、运行时激活和 Agent 的实际组合各有自己的状态。

## 1. 从插件市场到 Agent 有四层状态

| 对象 | 源码里的含义 | 排查时看什么 |
| --- | --- | --- |
| Market 条目 | 已配置索引中的包与版本元数据。 | 索引是否列出了这个版本，安装参数指向哪里。 |
| Bundle | 声明 `dsh.bundle` 并携带配置补丁的包。 | 依赖是否可解析，是否选进 Profile 的 `dsh.profile.bundles`。 |
| Plugin 行 | 配置合并后由 Cordis Loader 装载的一条插件声明。 | 条目是否启用，导入是否成功，所需服务是否已就绪。 |
| Preset | 一套带 ID 的子插件清单，装配到独立的作用域。 | Agent 绑定了哪一代，工具和技能是否从该作用域可见。 |

一个 Bundle 可以声明 Preset，Preset 的子插件可以注册工具、提示词片段或 Skill Provider。`PluginManager.listPlugins()` 把 Preset 条目纳入可查看的清单，但这些行的开关是只读的。想调整组合，可以通过 Bundle 补丁添加 Preset 声明，或覆盖已有声明的子插件列表。`AgentPresetRegistry` 的读取接口可导出声明内容供展示。[管理清单](../deepseek-harness/packages/boot/plugin-manager/src/index.ts#L480) · [Preset 声明与覆盖](../deepseek-harness/packages/preset/agent-preset-registry/README.zh.md#use-this-package) · [Web Preset 装配](../deepseek-harness/packages/bundle/web-app/presets/cordis.patch.yml#L126)

## 2. 市场索引怎样搜索并定位安装版本

`dsh-plugin-market` 读取 Profile 已配置的 HTTP(S) 或 Git 索引。`search()` 只接受配置中已有的有效地址。每个 URL 的并发加载共享同一个 Promise，成功后按 TTL 缓存。解析 JSON 和校验结构在 Worker Thread 内完成。搜索首次收到带词查询时才构建 MiniSearch 索引，按 500 条分批加入以让出事件循环。名称、标题、关键字参与加权，另支持前缀匹配、有限的模糊匹配，以及按分类和标签筛选。[地址校验与缓存](../deepseek-harness/packages/boot/plugin-market/src/index.ts#L365) · [并发加载](../deepseek-harness/packages/boot/plugin-market/src/index.ts#L405) · [Worker 解析](../deepseek-harness/packages/boot/plugin-market/src/parse-thread.ts#L36) · [文本索引](../deepseek-harness/packages/boot/plugin-market/src/search.ts#L89)

`search()` 先筛选排序并切页，再根据当前 Profile 的 `listBundles()` 和运行时版本给当页结果标注安装状态及兼容性。因此，缓存的市场文档可以复用，安装状态会随本地操作刷新。默认版本优先选择兼容的正式版。只有目录里完全没有正式版时，兼容的预发布版才可成为默认版本。[查询与分页](../deepseek-harness/packages/boot/plugin-market/src/index.ts#L231) · [标注当前安装情况](../deepseek-harness/packages/boot/plugin-market/src/index.ts#L377) · [版本选择](../deepseek-harness/packages/boot/plugin-market/src/index.ts#L508)

点选具体版本后，npm 条目生成 `name@version` 以及可选 registry，Git 条目生成包含固定提交和可选包目录的 spec。市场使用的是索引提供的版本信息，安装还会读取实际包的 manifest 和补丁。安装校验优先检查包的 `engines.dsh`，未声明时才检查 DSH 相关 peer 依赖。市场的兼容标记可以帮助选版，实际依赖和插件装载结果还要以后续检查为准。[安装参数](../deepseek-harness/packages/util/plugin-market-index/src/index.ts#L330) · [安装后校验](../deepseek-harness/packages/boot/plugin-manager/src/index.ts#L1081) · [运行时版本约束](../deepseek-harness/packages/boot/app-boot/src/plugin-compatibility.ts#L77)

## 3. 安装、启停、卸载怎样落盘和生效

| 用户操作 | 哪份状态变化 | 运行时还要做什么 |
| --- | --- | --- |
| 安装 Bundle | pnpm 更新 Profile 依赖和锁文件。通过校验后，默认把 Bundle 加入选择列表，也可只安装。 | 重新组合配置并检查应启动的条目，或等下次启动。 |
| 启停 Bundle | 更新 `package.json` 中有序的 `dsh.profile.bundles`。重新启用会追加到末尾。 | 加入或移除整层补丁，停用时保留依赖。 |
| 启停单个 Plugin | 在 Profile 的 `cordis.patch.yml` 中按条目 ID 及模块名匹配规则更新 `disabled`。 | 重算有效配置。更高优先级的补丁仍可覆盖这次选择。 |
| 卸载 Bundle | 先取消选择，确认相关 Loader 条目已退场，再移除依赖。 | 如禁用 HMR 且 Bundle 仍在运行，先停止 Profile。 |

单行启停用 YAML AST 更新最后一个匹配项，尽量保留注释。若低层配置已实现目标状态，可以删掉只包含开关的冗余覆盖项。Bundle 的选择顺序有实际含义，同一条目的后续补丁可能改变前面一层的结果。`enabled` 表示配置意图，`application` 才描述这次变更是否生效，例如 `applied`、`restart-required`、`overridden` 或 `failed`。[单行开关](../deepseek-harness/packages/boot/plugin-manager/src/patch.ts#L25) · [Bundle 开关](../deepseek-harness/packages/boot/plugin-manager/src/index.ts#L1278) · [插件启停与覆盖](../deepseek-harness/packages/boot/plugin-manager/src/index.ts#L936) · [卸载顺序](../deepseek-harness/packages/boot/plugin-manager/src/index.ts#L1156) · [操作结果类型](../deepseek-harness/packages/boot/plugin-manager/src/types.ts#L174)

并发管理有两处协调。包操作用 Profile 的 manifest 文件锁串行化，运行时配置变更则进入 HMR 的 `runExclusive()`，pnpm 在 HMR 队列外执行。启用时，运行时先发布新的包解析映射，再启动需要这些映射的插件。停用时先停止相关条目，再删除对应映射。HMR 还会检查当前已加载模块的文件变化，尝试替换可以重新装载的模块。没有 HMR 时，可以保存设置并报告 `restart-required`，当前进程继续用已经装载的组合。[Profile 锁](../deepseek-harness/packages/boot/plugin-manager/src/index.ts#L1385) · [配置队列](../deepseek-harness/packages/boot/plugin-manager/src/index.ts#L1320) · [HMR 重组顺序](../deepseek-harness/packages/boot/hmr/src/index.ts#L924) · [解析映射的生命周期用例](../deepseek-harness/packages/boot/plugin-manager/tests/package-reload.spec.ts#L217)

失败要看发生在哪一阶段。pnpm 调用失败或安装后校验拒绝时，安装路径会恢复事先读取的 `package.json` 与 `pnpm-lock.yaml`。完成包校验后，如果启用时插件导入或激活失败，依赖和选择可能已经保存，可以修正配置后重试。第三方 `updateBundle()` 在 HMR 路径上会检查目标、卸载旧版、尝试安装新版，失败时再尝试重装旧版，并单独返回 `restore` 的结果。因此，客户端要同时展示失败阶段、磁盘是否变化、运行时状态和恢复结果。[安装与激活边界](../deepseek-harness/packages/boot/plugin-manager/src/index.ts#L1081) · [失败后保留设置的用例](../deepseek-harness/packages/boot/plugin-manager/tests/manager.spec.ts#L986) · [更新补偿](../deepseek-harness/packages/boot/plugin-manager/src/index.ts#L791)

这些能力管理的是宿主进程。Agent 通过 `plugin_manager` 执行管理操作时，工具检查 `danger-full-access` 或当次审批，安装时的依赖构建脚本有单独的批准记录。装进 Host 的插件代码在 Host 进程运行。Preset Scope 约束普通工具和技能解析的可见范围，所以第三方插件的代码信任仍然要在宿主侧决定。[工具授权](../deepseek-harness/packages/boot/plugin-manager/src/tools.ts#L89) · [构建脚本批准](../deepseek-harness/packages/boot/plugin-manager/src/build-approval.ts#L85) · [Preset 隔离边界](../deepseek-harness/packages/preset/agent-preset-registry/README.zh.md#known-limitations-and-deferred-work)

## 4. Preset 挂载怎样影响新旧会话

`AgentPresetRegistry.register()` 收到声明后立即创建由注册表持有的 Scope，并用声明位置的解析基址挂载内存 Loader 树。挂载会审计插件导入、激活与服务是否泄漏到全局。条目若仍在等 Host 服务，会保留在树中，绑定或读取时可等 Host Loader 结算后重新审计。失败的定义保留诊断，其他正常的定义仍能使用。Preset 专有服务的生产者和消费者应位于共同的 Cordis 隔离 realm。Agent 的 Scope 父链接决定工具和技能可见性，显式的 `serviceFor()` 则可在 Agent 保留的组合内查服务。[注册与激活](../deepseek-harness/packages/preset/agent-preset-registry/src/index.ts#L96) · [重新审计](../deepseek-harness/packages/preset/agent-preset-registry/src/index.ts#L150) · [挂载与泄漏检查](../deepseek-harness/packages/preset/agent-preset-registry/src/mount.ts#L204) · [服务查找](../deepseek-harness/packages/preset/agent-preset-registry/src/index.ts#L339)

同一个 Preset ID 可以先后对应多个运行时代际。新 Agent 绑定当前代际。定义被替换或移除时，旧代际标记为 `retired`，等待 `users` 降到零后销毁 Scope 和 Loader 树。已有 Agent 保留自己的绑定，子 Agent 通过 `composeFrom()` 继承父 Agent 的精确代际，临时历史读取也可以获取短期引用。获取引用时若遇到定义在审计期间被替换，`retain()` 会重读当前定义，避免绑定已退出新会话入口的组合。[引用获取与绑定](../deepseek-harness/packages/preset/agent-preset-registry/src/index.ts#L247) · [回收条件](../deepseek-harness/packages/preset/agent-preset-registry/src/index.ts#L163) · [子 Agent 继承](../deepseek-harness/packages/preset/agent-preset-registry/src/index.ts#L312) · [历史读取引用](../deepseek-harness/packages/preset/agent-preset-registry/src/index.ts#L388)

源码的代际用例先给 Agent 绑定旧版 `standard`，再用同 ID 注册新定义。旧 Agent 退出后，继承了旧版的子 Agent 仍让旧代际留存。子 Agent 退出后才回收，新 Agent 的工具清单来自新组合。这里保留的是进程里的 Preset 组合，Profile 级 HMR 仍会重组 Host 的共享插件与服务，所以旧会话依赖的全局能力也可能随之改变。[代际用例](../deepseek-harness/packages/preset/agent-preset-registry/tests/registry.spec.ts#L38) · [挂载的全局依赖](../deepseek-harness/packages/preset/agent-preset-registry/README.zh.md#understand-the-implementation) · [Profile HMR](../deepseek-harness/packages/boot/hmr/src/index.ts#L937)

空白会话能在首轮开始前切换 Preset，`select()` 按 Agent ID 串行处理选择，检查回合边界后重绑作用域，提交 `agent-preset/selected`。会话投影从头部读取初始 Preset ID，再用选择事件更新。Web 冷恢复时按保存的 ID 挂载**当前定义**，原先的内存代际已经随进程退出。如果这个 ID 已被删除，会因定义找不到而拒绝恢复。更改默认 Preset 影响后续创建时的默认选择，已有记录仍由自己的 ID 决定。[空白会话切换](../deepseek-harness/packages/preset/agent-preset-registry/src/index.ts#L356) · [会话投影](../deepseek-harness/packages/preset/agent-preset-registry/src/session.ts#L37) · [Web 冷恢复](../deepseek-harness/packages/api/session-controller/src/agent.ts#L299) · [挂载当前定义](../deepseek-harness/packages/api/session-controller/src/agent.ts#L262)

## 5. 系统设计题｜升级时怎样保留旧会话并限制成本

如果产品要求跨进程恢复到原先的工具和提示词组合，可以让任务在创建时记录不可变的 `composition_id`。组合记录包括 Preset 配置摘要、插件版本及制品摘要、工具接口版本和适用的 Host 运行时版本。资源制品与该记录关联存储，恢复时按目标版本找 Worker。发布前验证组合与 Host 接口兼容性并启动一组新版 Worker，健康检查通过后原子切换新会话的路由。

```mermaid
flowchart LR
    A["新版组合校验并启动"] --> B["切换新会话的入口"]
    B --> C["新会话绑定新版 Worker"]
    B --> D["旧会话继续用旧版 Worker"]
    D --> E["停止接收新会话并等待存量任务结束"]
    E --> F["回收 Worker，按保留规则清理制品"]
```

同一组合的会话可以共享一组 Worker，通过请求作用域隔离租户状态。对原生依赖、运行时 ABI 或全局服务不兼容的版本，给旧版和新版分配不同进程。进程隔离会付出额外的内存、连接池和启动成本，因此可以约束每个插件族至多同时运行两个版本，并给旧版设活跃会话配额和排空时限。第三个版本进入发布队列时，先判断旧版是否已经排空，或是否有通过状态迁移的路径。会话闲置时可释放 Worker 引用，后续按保留的制品冷启动，具体期限由产品的恢复承诺决定。

长任务持有带心跳的租约。租约到期后的回收还要检查执行器记录的任务状态与执行归属，避免网络分区时把仍有任务的 Worker 当成空闲。制品保留期限可以长于 Worker 驻留期限。需要回滚时先把新会话路由切回已知可用版本，迁移中的会话按已持久化的阶段继续。带副作用的工具调用应记录操作 ID 和执行结果，切换或恢复后先核对结果再重试。紧急撤回仍由当前授权策略拒绝后续敏感调用，旧组合只保留已批准的运行范围。

## 6. 面试官怎么逐层追问

### 市场显示已安装，为什么模型仍然看不到这个工具？

先确认市场的包状态，再查 Bundle 是否选中、补丁是否生效、插件行是否装载，最后看 Agent 绑定的 Preset 及工具可见性。继续追问 `application: restart-required` 与 `application: failed` 的区别，答出前者可能已经保存配置并等待重启，后者需要查看失败阶段和诊断。

### 停用一个 Bundle 时，为什么要关心包解析映射的删除顺序？

先停止相关插件，插件清理期间仍可解析所需模块和资源。源码用例检查了先停插件再发布丢弃映射的顺序。再追问两个并发安装为什么既需要 Profile 锁又需要 HMR 队列。前者保护依赖和磁盘状态，后者串行化 Loader 的运行时重组，两处协调各自保护不同资源。[映射与停止顺序用例](../deepseek-harness/packages/boot/plugin-manager/tests/package-reload.spec.ts#L233)

### 旧 Agent 退出后，旧 Preset 一定可以销毁吗？

先说出 `retired && users === 0`。子 Agent 和历史读取仍可能持有引用。继续追问进程重启后为什么可能换成新版，回答日志只持久化 Preset ID，冷恢复要绑定当前定义。若要求恢复到同一版本，需要把组合身份和制品可用性纳入恢复契约。

### v1 有长任务，v2 正在接流量，v3 又要发布，怎么处理？

先明确允许同时驻留的版本数和旧任务的完成承诺。可以让 v3 暂停接入，给 v1 停止接收新会话，并按任务状态核对租约。接着追问仍在进行的外部调用如何迁移，回答要持久化操作 ID 与结果，先核对再重试。能明确区分释放一个 Worker、回收磁盘制品和停止旧版本接流量，就能把资源约束落到具体操作上。

下一篇进入 MCP，沿传输、能力协商和工具调用，追到模型实际拿到的工具声明。
