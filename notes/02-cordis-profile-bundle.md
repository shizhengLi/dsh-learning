# 02｜Cordis、Profile、Bundle 与 YAML 覆盖

> 源码基线为 DeepSeek Harness `dsh-v0.2.3-alpha.2`，提交 `4b399c39a6d1479cd84ca591b4d7c88e24d38a96`。[系列目录](../README.md) · [上一篇](01-headless-agent-run.md) · [上游仓库](https://github.com/deepseek-ai/deepseek-harness)

在 DSH 里，同一个 Agent Loop 可以由 headless 一次性任务、Web 会话或 SDK 驱动。启动器先把一组配置补丁合成为插件树，Cordis 再根据服务依赖激活插件。要理解扩展机制，得沿着这两步分别追问：配置写出了什么？进程此刻运行的又是什么？

这一区分在热更新时尤其重要。YAML 可能已经写入新值，Loader 条目也可能已经收到它，但如果校验失败，插件仍在使用上一次通过校验的配置。另一方面，插件已经挂载，也可能只给某个 Agent 或 Preset 提供工具。单看配置文件，很难回答用户为什么看到了旧行为。[Cordis 入门](../deepseek-harness/docs/cordis-primer.zh.md) · [配置更新回归测试](../deepseek-harness/packages/boot/app-boot/tests/config-reload.spec.ts#L232)

## 1. 从 Profile 到插件实例

| 概念 | 表示什么 | 例子 |
| --- | --- | --- |
| Profile | 一套有名字的应用组合，记录 Bundle 顺序，并有自己的插件安装和配置补丁 | `headless`、`web`、`sdk` |
| Bundle | npm 包发布的一组 Cordis 配置补丁及其运行依赖，用 `dsh.bundle.patch` 指向一个或多个文件 | `dsh-base`、`dsh-headless` |
| Entry | Loader 认识的一条配置记录。树内用 `id` 定位，`name` 指定模块，`config` 提供参数 | `id: agent-loop` 对应 `@deepseek-ai/dsh-agent-loop` |
| Fiber | 一个插件实例的生命周期。追踪依赖、配置、启动、卸载以及该实例登记的资源释放函数 | 两条配置引用同一模块，可以各有自己的 Fiber |
| Service | Provider 提供给其它插件使用的运行时能力，由 Context 按名称查找 | `ctx.llm`、`ctx.sessions`、`ctx.tools` |

内置的 `headless` Profile 按顺序使用 `dsh-base` 和 `dsh-headless`。`web` 使用同一基础层，再叠加 `dsh-web-app`。`sdk-minimal` 则由一个独立 Bundle 提供完整组合。这里的 Profile 解决的是「这个进程怎样装配」。Agent Preset 解决的是「某个 Agent 使用哪套子插件、工具和提示词」，细节留到第 17 篇。[Profile 模板](../deepseek-harness/packages/util/official-bundles/src/bundles.ts#L9) · [Headless Bundle 清单](../deepseek-harness/packages/bundle/headless/package.json#L41) · [Preset Registry](../deepseek-harness/packages/preset/agent-preset-registry/README.md)

`runProfile()` 读取 Profile 和 Bundle，准备一个空的根配置文件，再把有序补丁传给 `boot()`。空文件给根 Include 与 Loader 提供真实的文件路径和解析基准。配置树从空条目列表叠加出来，Bundle 的 patch 可以插入其它层继续修改的条目。[Profile 启动](../deepseek-harness/apps/cli/src/profile-boot.ts#L178) · [组合入口](../deepseek-harness/apps/cli/src/profile-boot.ts#L323) · [composeEntries](../deepseek-harness/packages/boot/app-boot/src/profile.ts#L747)

```mermaid
flowchart LR
    A[CLI 选择 Profile] --> B[读取 Bundle 清单与补丁]
    B --> C[叠加 Profile、Home、命令行补丁]
    C --> D[Include 合成 Entry 树]
    D --> E[Loader 解析模块与依赖]
    E --> F[Fiber 启动插件]
    F --> G[注册 Service、事件、工具与提示词]
```

## 2. YAML 补丁如何命中条目

常规顺序是 **Bundle 声明顺序 → Profile 的 `cordis.patch.yml` → `$DSH_HOME/cordis.patch.yml` → 按传参顺序读取 `--patch`**。同一个字段由后命中的补丁决定。运行时 `readProfilePatches()` 还有一个显式分支，会在符合条件时追加环境开关生成的 telemetry 补丁。排查以环境变量驱动的差异时，要一起看这段逻辑。[补丁顺序](../deepseek-harness/packages/boot/app-boot/src/profile-context.ts#L68)

`applyPatches()` 一边遍历补丁，一边维护 `id → Entry` 索引和字段的来源。`insert` 可追加到树根，也可通过 `id` 插入已有 Group。普通补丁根据 `id` 定位，提供的 `name` 用来校验是否命中预期模块。给了未知 ID 或不匹配的模块名时，会产生诊断并跳过这条补丁。补丁覆盖 Group 的 `config` 时，旧子条目先从索引移除，新子条目随后进入索引，供下一层引用。[补丁实现](../deepseek-harness/vendor/include/src/patch.ts#L96) · [配置导出测试](../deepseek-harness/packages/boot/app-boot/tests/config-dump.spec.ts#L146)

**`config` 在条目上是一个整体字段。** `dsh-base` 为 `agent-loop` 提供 `config: { agents: [] }`。假设团队另有一个 Profile 声明了启动时要创建的 Agent，再叠加以下配置来调工具并行度，前一层的 `agents` 会随 `config` 一起被替换。在基础 Profile 上，Schema 会给缺失的 `agents` 填回空数组，行为看起来正常。自定义 Profile 的启动 Agent 列表就需要在补丁中显式保留。[基础条目](../deepseek-harness/packages/bundle/base/cordis.patch.yml#L544) · [Agent Loop Schema](../deepseek-harness/packages/core/agent-loop/src/index.ts#L294)

```yaml
- id: agent-loop
  config:
    maxParallelToolCalls: 4
```

当你只补 `disabled: true` 时，覆盖的字段只有 `disabled`，`name` 和 `config` 都仍然来自前面的层。反过来，写成 `name: ./another-provider.mjs` 是一次断言，会因名称不匹配而跳过。要换模块，可以停用原条目，再以新条目挂载接替实现，并让消费方继续通过同一个 Service key 获取能力。先确认该 Service 是否允许多实例，以及是否有依赖在切换期间需要停止。[补丁字段语义](../deepseek-harness/vendor/include/src/patch.ts#L125) · [Service 注册规则](../deepseek-harness/vendor/cordis/src/reflect.ts#L303)

### `!!js` 什么时候求值？

Headless 的 `task` 参数提供了一个现成例子。配置写的是 `task: !!js ctx.headlessStartup.task`，同一条目声明 `inject: [headlessStartup]`。Include 解析 YAML 时把 `!!js` 保存成表达式节点。目标 Fiber 准备激活时，Loader 才在目标插件上下文求值 `config`，再把结果交给 Config 校验。`disabled` 的表达式在决定条目是否挂载时求值，其余条目元数据按字面值处理。[Headless 配置](../deepseek-harness/packages/bundle/headless/cordis.patch.yml#L17) · [Config 插值](../deepseek-harness/vendor/loader/src/index.ts#L122) · [Disabled 求值](../deepseek-harness/vendor/loader/src/config/entry.ts#L92)

`--dump-config` 展示合成的 Entry 与来源，还会保留未求值的 `!!js`。它可以解释「哪个补丁覆盖了同名字段」，服务是否注入成功和插件最终是否激活则要到运行时确认。`--dump-default-config` 便于在用户层配置有误时检查 Bundle 自带的组合。[导出入口](../deepseek-harness/apps/cli/src/dump-config.ts#L32) · [导出测试](../deepseek-harness/packages/boot/app-boot/tests/config-dump.spec.ts#L47)

## 3. Context、依赖注入与卸载

插件可以导出 `apply(ctx, config)` 与 `inject`，也可以是 Cordis `Service` 子类。`ctx.plugin()` 按插件实现维护 Runtime 记录，再为这次挂载创建一个 Fiber。Provider 通过 `ctx.provide()` 提供服务，Service 子类也会走这条注册路径。反射服务用当前 Context 的隔离映射查找服务，同一隔离域对同名服务只接受一个 Provider。[插件注册](../deepseek-harness/vendor/cordis/src/registry.ts#L304) · [Service 基类](../deepseek-harness/vendor/cordis/src/service.ts#L38) · [Context 查找与注册](../deepseek-harness/vendor/cordis/src/reflect.ts#L269)

`inject` 是运行时依赖声明。Fiber 缺少必要服务时处于 `PENDING`，服务到位后才进入 `LOADING`，激活后是 `ACTIVE`，启动失败有 `FAILED` 状态。Fiber 用配置 revision 与依赖实现的 `uid` 计算加载身份。如果同名 Provider 的实例被换掉，依赖者也能识别这个变化并走生命周期更新。启动顺序由依赖关系保证，只按 YAML 的行序安排相互依赖的插件会遗漏这个约束。[Fiber 状态](../deepseek-harness/vendor/cordis/src/fiber.ts#L172) · [依赖变化](../deepseek-harness/vendor/cordis/src/fiber.ts#L639)

`ctx.effect()` 把资源与 Fiber 绑在一起。挂载工具、注册模型路由、订阅文件或事件时，要能一并撤销注册、关闭 watcher 或释放连接。`ctx.on()` 底层本身就是一个 effect，卸载时会注销监听器。Cordis 在卸载 Provider 时等待依赖者完成卸载，再清理 Provider 的 effect，使依赖者的清理逻辑仍能访问所需服务。如果资源 A 的关闭必须等资源 B 收尾，应在同一个 effect 的 disposer 中显式 `await`，因为普通的 Fiber 卸载会并行等待独立 disposer。[监听器注册](../deepseek-harness/vendor/cordis/src/events.ts#L267) · [Fiber 卸载](../deepseek-harness/vendor/cordis/src/fiber.ts#L734)

### Cordis 的服务隔离与 DSH 的 Agent Scope

服务隔离回答「`ctx.tools` 查到哪个服务实例」。DSH 的 `core/scope` 另外给 Context 标记一个不透明的 Scope Key，供同一个工具或提示词 Registry 分层使用。Agent 可读全局注册和自己的祖先链上的注册，同名的近层定义可以遮蔽远层定义。事件路由也能用这条祖先关系，使祖先上的监听器接到后代 Agent 的事件。于是两个 Agent 可以共享运行时服务，同时拥有不同的工具集合。文件、网络与进程权限仍交给相应的执行策略校验。[Scope Key 与父子关系](../deepseek-harness/packages/core/scope/src/index.ts#L14) · [工具视图](../deepseek-harness/packages/core/tools/src/index.ts#L190)

## 4. Waterfall 的 `next()` 为什么值得单独讲

| 分发方式 | 执行语义 | 适合做什么 |
| --- | --- | --- |
| `emit` | 依次调用监听器，分发方不等待异步结果 | 通知观察者 |
| `parallel` | 启动并等待并发的监听器 | 多个独立观察者需要在返回前完成 |
| `serial` | 依次等待监听器，遇到 bail 值时返回 | 对顺序有要求的收尾或决策 |
| `bail` | 同步逐个调用，返回第一个 bail 值 | 找到第一个有效的同步决定 |
| `waterfall` | 注册顺序上外层包内层，业务终点是最后那个 `next` | 审批、观测、持久化检查点等环绕控制 |

`waterfall` 直接返回最外层监听器的结果。返回值可以是 Promise，也可以是 AsyncIterable。调用 `next()` 才会进入下游，同一个包装层只能调用一次。省略 `next()` 会截断链条，所以写一个只负责记录耗时的插件也要返回 `next()` 的结果。事件关联的 TypeScript 声明通常用模块声明合并维护，运行时分发再按事件的模式执行。[事件分发](../deepseek-harness/vendor/cordis/src/events.ts#L202) · [Cordis Waterfall 约定](../deepseek-harness/docs/cordis-primer.zh.md#cordis-waterfall-semantics)

上一篇遇到的会话检查点就是一例。`session-checkpoint-policy` 监听 `llm/stream`，给下游流包上一层异步生成器，先 `await ctx.sessions.flush(session)`，然后 `yield* next()`。持久化失败时，生成器在进入下游之前就会失败。工具侧同样有执行前的刷盘关口。这样一项持久化策略就能装配到 Agent 上，不必把存储实现写进模型适配器。[检查点实现](../deepseek-harness/packages/session/session-checkpoint-policy/src/index.ts#L23)

## 5. 热更新到底动了哪些东西

基础 Bundle 的 HMR 默认把源码监听根设为 `[]`，保留显式注册的配置监听。随附的 headless、SDK 和 ACP 补丁会停用这一 HMR 条目，Web 继承基础装配。运行配置更新时，HMR 监听 Profile 清单及涉及的补丁文件，在应用就绪后读取变化，通过 `runExclusive()` 把配置重组和自动重载串行起来。它能避免两个重组操作互相覆盖，已经开始的模型或工具调用仍有自己的生命周期。[基础 HMR 条目](../deepseek-harness/packages/bundle/base/cordis.patch.yml#L27) · [Headless 覆盖](../deepseek-harness/packages/bundle/headless/cordis.patch.yml#L35) · [更新队列](../deepseek-harness/packages/boot/hmr/src/index.ts#L264) · [配置监听](../deepseek-harness/packages/boot/hmr/src/index.ts#L384)

1. **合成和预检。** App Boot 应用新一轮补丁并更新根 Include，Loader 对条目做增加、移除或更新，再等待相关 Fiber 结算。YAML 解析失败、Config Schema 校验失败和新插件代码启动失败发生在不同阶段。仓库用测试覆盖了语法错误保留上次可用树，以及条目收到无效值时已运行 Fiber 保持上一个有效配置。文件替换已经逐出旧模块后，新代码导入或启动失败则可能留下 `failed` 条目。[重新组合](../deepseek-harness/packages/boot/app-boot/src/index.ts#L399) · [YAML 错误测试](../deepseek-harness/packages/boot/app-boot/tests/config-reload.spec.ts#L63) · [运行配置测试](../deepseek-harness/packages/boot/app-boot/tests/config-reload.spec.ts#L232) · [模块替换行为](../deepseek-harness/packages/boot/hmr/README.zh.md)
2. **仅 volatile 值变化。** 已激活 Fiber 的上下文和其它条目选项相同，且差异只有 Schema 声明的 volatile 部分时，Loader 可以校验新值后更新原有引用。`maxParallelToolCalls` 就用这类配置，Agent Loop 在派发工具批次前读取它。已经提交给执行器的批次持有本次传入的上限，后续批次可读到新值。[条目更新](../deepseek-harness/vendor/loader/src/config/entry.ts#L138) · [并行度读取](../deepseek-harness/packages/core/agent-loop/src/agent.ts#L576)
3. **普通重启与模块替换。** 同一条目的普通配置变更通常沿 Fiber 的更新路径执行，插件拥有更新钩子时可决定如何应用。模块文件变化的处理还可能逐出缓存、停止受影响条目并重新导入。具体哪些路径支持恢复旧配置或旧模块，要看失败发生在预检、卸载还是激活阶段。[Fiber 更新](../deepseek-harness/vendor/cordis/src/fiber.ts#L798) · [HMR 实现说明](../deepseek-harness/packages/boot/hmr/README.zh.md)

### 旧请求会切到新 Provider 吗？

DSH 的 `llm.prepareCall()` 在准备阶段获取 adapter 注册记录和解析后的配置，返回绑定了这份实现的 `stream()`。因此同一份已准备调用继续面向它所绑定的 Provider，后续新准备的调用会重新查路由。旧调用能否顺利完成还取决于 Provider 的取消、连接和资源释放实现。Prepared call 绑定本身只回答调用路由的归属。[Prepared LLM call](../deepseek-harness/packages/llm/llm/src/index.ts#L942)

工具还有一层身份校验。`tools.resolve()` 为 step 形成一次准入集合，之后在拿可运行工具时再检查该定义和注册记录是否仍匹配。旧工具卸载并重新注册同名实现后，旧 step 的选择不会把新实现混进来。下一 step 再按当时视图解析工具。这能避免模型按上一代工具 Schema 产生的参数，意外直接跑到新一代实现。[工具准入](../deepseek-harness/packages/core/tools/src/index.ts#L257)

## 6. 用真实补丁函数复核整块替换

在学习仓库根目录用 Node 24 执行下面的命令。它把 YAML 解析后的补丁表示为对象，调用仓库里的 Include `applyPatches()`。短小的解析钩子把一个工作区依赖映射到 vendored 源码，从而直接运行该函数。实验依次复核 `config` 替换、只改 `disabled`、以及 `name` 作为断言三种行为。

```bash
node --experimental-transform-types --input-type=module <<'JS'
import assert from 'node:assert/strict'
import { registerHooks } from 'node:module'

const cosmokit = new URL('./deepseek-harness/vendor/cosmokit/src/index.ts', import.meta.url)
const hooks = registerHooks({
  resolve(specifier, context, next) {
    return next(specifier === '@deepseek-ai/cosmokit' ? cosmokit.href : specifier, context)
  },
})

try {
  const { applyPatches } = await import('./deepseek-harness/vendor/include/src/patch.ts')
  const id = 'agent-loop'
  const name = '@deepseek-ai/dsh-agent-loop'
  const base = { insert: [{ id, name, config: { agents: [] } }] }
  const limit = { id, config: { maxParallelToolCalls: 4 } }
  const disabled = { id, disabled: true }
  const warnings = []
  const wrongName = { id, name: './another-provider.mjs', config: { maxParallelToolCalls: 9 } }
  const { data } = applyPatches([], [base, limit, disabled, wrongName], warning => warnings.push(warning))

  assert.deepEqual(data, [{ id, name, config: { maxParallelToolCalls: 4 }, disabled: true }])
  assert.equal(warnings.length, 1)
  assert.match(warnings[0], /name mismatch/)
  console.log(JSON.stringify(data[0], null, 2))
} finally {
  hooks.deregister()
}
JS
```

如果已按项目开发指南准备好依赖，可以在 `deepseek-harness/` 用 `pnpm dsh --profile headless --dump-default-config` 检查官方组合，用 `pnpm dsh --profile headless --patch ./overlay.yml --dump-config` 检查自定义补丁。带上第 2 节的 YAML，目标条目的 `config` 应只包含并行度一项，`agents` 的默认值会在运行时校验时补齐。本文已运行的是上面的纯补丁实验，完整 Profile 启动与热更新有各自的测试入口。[配置导出测试](../deepseek-harness/packages/boot/app-boot/tests/config-dump.spec.ts) · [配置重载测试](../deepseek-harness/packages/boot/app-boot/tests/config-reload.spec.ts)

## 7. 面试官会怎么追问

### 一条 Entry 写进 YAML 后，插件就可以调用了吗？

逐层查四个信号。先用配置转储确认条目是否存在、命中哪个补丁，再检查 `disabled` 和模块解析，接着看 Fiber 的依赖与状态，最后确认工具对目标 Agent 的 Scope 可见。答到这里已经能区分写入配置、模块载入、服务激活和 Agent 可用四个时间点。

### 维护者只改了并行度，为什么启动时配置的 Agent 消失了？

补丁以 ID 匹配，整块替换了 `config`。如果前一层的 `config.agents` 有值，而新层只给 `maxParallelToolCalls`，运行时可能用 Schema 的空数组默认值启动。应逐个检查 Profile、Home、命令行的配置来源。继续追问时可以加一条现场条件：修改后的并行度只在后续工具批次可见，你会如何用两批并发工具验证读值时点？

### 你写了一个日志插件，为什么接入后工具始终没有执行？

先检查 `tools/execute` 的 Waterfall 回调是否返回 `next()`。想阻止操作的策略才应短路，并返回明确的结果。随后检查插件所有者与注销路径，确认连重载两次后，一次调用只产生一份日志，旧监听器和旧连接都已释放。可用共享计数器记录处理次数，另用延迟的 disposer 验证卸载完成时点。

### 生产系统怎么做 Provider 零停机热替换？

这是扩展设计题。先准备并探测新 Provider，成功后原子切换新请求的路由。每次调用在开始时固定实现身份，旧实例停止接单并等待在途请求排空，超过截止时间再按协议取消。工具或外部写操作另外定义幂等与结果查询策略。需要记录路由版本、拒绝数、在途计数、卸载耗时和新 Provider 激活失败原因。让候选人具体推演「已切流后新 Provider 健康检查变红」与「旧连接一直关闭不了」两个分支，并交代回退后是否仍可能收到旧调用的结果。

下一篇回到 Agent Loop，细看 Inbox、turn/step、取消与失败收尾如何共同约束运行状态。
