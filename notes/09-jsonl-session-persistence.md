# 09｜默认 JSONL：写入、压缩、恢复与迁移

> 源码基线为 DeepSeek Harness `dsh-v0.2.3-alpha.2`，提交 `4b399c39a6d1479cd84ca591b4d7c88e24d38a96`。[系列目录](../README.md) · [上一篇](08-session-event-log-projection-replay.md) · [上游仓库](https://github.com/deepseek-ai/deepseek-harness)

用户在屏幕上看到了 `DONE`，进程随即被杀。重开时能看到多少对话，取决于几个不同的时点：assistant 事件是否已经进入内存日志，存储后端是否已写出字节，以及刷盘是否成功。默认 JSONL 后端把这些时点落实为写句柄、批处理、压缩帧和可恢复的日志前缀。[持久化句柄契约](../deepseek-harness/packages/session/session-persistence/src/handle.ts) · [JSONL 后端](../deepseek-harness/packages/session/session-persistence-jsonl/README.zh.md)

## 1. DSH 把会话写到哪里

基础 Bundle 挂载 `dsh-session-persistence-jsonl`，根目录配置为 Harness home 下的 `sessions`。后端自身要求显式提供 `root`。它按原始项目和 Session ID 分层，项目目录用于导航，Session ID 经编码后占用一个路径段。每个 Session 目录可以同时保存多个格式代际。当前基线的 `SESSION_FORMAT_VERSION` 是 `4`，默认写 `session.v4.jsonl.zstd`，配置 `compression: 'none'` 的存储根则写 `session.v4.jsonl`。历史 v0 采用省略版本号的文件名。[基础配置](../deepseek-harness/packages/bundle/base/cordis.patch.yml#L142) · [目录与文件命名](../deepseek-harness/packages/session/session-persistence-jsonl/src/format.ts#L248) · [当前版本常量](../deepseek-harness/packages/core/session/src/types.ts#L90)

```text
$DSH_HOME/sessions/
  <项目目录>/
    <编码后的 Session ID>/
      session.lock           # POSIX 写者锁
      session.v3.jsonl.zstd    # 已迁移会话可能保留的历史文件
      session.v4.jsonl.zstd    # 由当前 writer 追加的代际
```

每个代际的第一条记录是 header，带 Session 身份和格式版本，后续记录是事件。当前格式一条事件占一行。磁盘表示还有无损的字段编码，例如连续的 `sourceEventSeqs` 可用区间缩写，解码后仍还原为准确序号。后端在一个存储根内选择一种物理编码，检测到另一种编码的代际会报告不匹配。因此修改压缩配置涉及存储根的选择。版本代际与压缩表示分别解决事件解释和字节编码问题。[物理事件编码](../deepseek-harness/packages/session/session-persistence-jsonl/src/format.ts#L307) · [代际与编码检查](../deepseek-harness/packages/session/session-persistence-jsonl/src/index.ts#L1651)

## 2. 从内存追加到文件可恢复

| 时点 | 已经成立的事实 |
| --- | --- |
| 内存 `Session.append()` 返回 | 事件已被 Session 接纳，观察者可处理。实时写入监听会把事件放入后端缓冲。 |
| JSONL 的直接 `SessionHandle.append()` 返回 | 这次追加已按 JSONL 后端实现写入并同步，存储游标前进。 |
| 实时路径 `ctx.sessions.flush(session)` 返回 | 参与刷盘的持久化监听已排空该 Session 的实时缓冲并通过句柄屏障。 |
| 写句柄 `close()` 结束 | 关闭会尝试排空待写数据并释放所有权，排空或释放失败会返回错误。 |

持久化接口给不同后端留了批处理空间。通用 `SessionHandle.append()` 只约定接受与可见性，调用方需要跨崩溃保证时用 `flush()`。这个版本的 JSONL 实现更强，直接写句柄的 `append()` 在物理同步后才完成。另一方面，实时的 `session/event` 写入先缓冲再排空，所以看到 UI 事件与直接等待持久化句柄的追加是两种证据。[通用契约](../deepseek-harness/packages/session/session-persistence/src/handle.ts#L85) · [JSONL 句柄](../deepseek-harness/packages/session/session-persistence-jsonl/src/storage.ts#L186) · [实时事件监听与刷盘](../deepseek-harness/packages/session/session-persistence-jsonl/src/storage.ts#L538)

实时缓冲的定时批处理窗口是 200 毫秒。这个数字控制主动等待多久才触发一次写入，由于排队和磁盘 I/O，无法据此推断事件一定在 200 毫秒内落盘。每个写句柄通过 Promise 链串行处理变更，多个排空请求共享正在进行的 drain。写入失败时，实时批次按原顺序放回缓冲区，等待下一次排空。头部事件的 `seq` 必须匹配存储游标，存储成功后才推进游标。[批处理常量](../deepseek-harness/packages/session/session-persistence-jsonl/src/storage.ts#L36) · [缓冲排空与写入](../deepseek-harness/packages/session/session-persistence-jsonl/src/storage.ts#L278)

```mermaid
flowchart LR
    E[Session 接纳事件] --> Q[后端实时缓冲]
    Q --> D[定时排空或显式刷盘]
    D --> S[同一写句柄串行处理]
    S --> T[校验所有权与连续 seq]
    T --> W[写入文件并同步]
    W --> P[推进已写游标]
```

### 为什么有文件 `fsync`，还要同步目录

普通 `create(header)` 先在进程内登记待实体化的会话。首次非空追加才写 header 与事件。显式刷新一个空会话时，也可发布仅含 header 的文件。POSIX 首次发布先写同目录临时文件并同步文件，再用 `link()` 建立最终文件名。目标名已存在时，硬链接操作会失败，保住已有产物。最后同步目录以持久化名称到文件的映射，再尽力清理临时文件名。Windows 路径使用自身的无覆盖、写穿透发布操作。[懒实体化](../deepseek-harness/packages/session/session-persistence-jsonl/src/index.ts#L440) · [POSIX 发布](../deepseek-harness/packages/session/session-persistence-jsonl/src/index.ts#L1400) · [Windows 发布](../deepseek-harness/packages/session/session-persistence-jsonl/src/index.ts#L1444)

已有文件的追加会记下写前长度，写入一批，再同步文件。捕获到写入或同步失败，会尝试截回旧长度并再次同步，保住原来的连续前缀。若回滚也失败，后端会同时报告两个错误。遇到这种失败，调用方应重新确认磁盘前缀和写入权，再判断是否能补写。仅凭一次失败返回，无法断定磁盘从未改变。[追加、同步与回滚](../deepseek-harness/packages/session/session-persistence-jsonl/src/index.ts#L1524)

## 3. 谁能写同一个会话

后端先在进程内按 Session ID 记录写者，再取得跨进程的内核锁。已存在的会话在 `open(id, 'write')` 时取得锁。尚未实体化的会话在首次发布前取得锁，并在写句柄关闭时释放。POSIX 使用对 `session.lock` 的非阻塞 `flock`，并核对锁住的 inode 仍对应预期路径。Windows 使用从路径派生的命名内核信号量。两个写者争用同一会话时，会得到明确的已被占用错误。[写打开](../deepseek-harness/packages/session/session-persistence-jsonl/src/index.ts#L514) · [跨进程锁](../deepseek-harness/packages/session/session-persistence-jsonl/src/lease.ts#L1)

进程退出时内核回收锁。一个仍存活但卡住的写者则继续占有它，避免另一个写者接手后旧写者恢复并追加冲突的序号。读取者不抢占写锁，同一后端若正在关闭本进程的写句柄，读打开会等排空结束。面试里把单机方案推广到多机器时，需要重新设计写入权以及过期写者的提交校验。[内核锁语义](../deepseek-harness/packages/session/session-persistence-jsonl/src/lease.ts#L3) · [读取打开时的关闭交接](../deepseek-harness/packages/session/session-persistence-jsonl/src/index.ts#L479) · [关闭与释放](../deepseek-harness/packages/session/session-persistence-jsonl/src/storage.ts#L218)

## 4. 为什么按批写独立的 Zstandard 帧

默认路径将 header 单独压成一个带校验和的帧，实时追加的每个持久批次再压成独立帧并拼到末尾。已有帧可以保留，读取 `stat` 或 `list` 时，只需读取对应的 header 行或 header 帧。完整正文读取则按帧扫描、解压并校验内容。迁移写新代际时仍保留独立 header 帧，历史事件可以流式压到一个正文帧。帧也因此是物理编码单位，一帧可以包含多条事件，一个 turn 可以跨多个帧。[编码与帧边界](../deepseek-harness/packages/session/session-persistence-jsonl/src/index.ts#L1489) · [Zstandard 实现](../deepseek-harness/packages/session/session-persistence-jsonl/src/zstd.ts#L107) · [迁移的流式编码](../deepseek-harness/packages/session/session-persistence-jsonl/src/generation.ts#L713) · [Zstandard 格式规范](https://www.rfc-editor.org/rfc/rfc8878.html)

### 三种尾部故障对应三种处理

| 故障位置 | 读取与后续写入 |
| --- | --- |
| 原始 JSONL 的最后一行未写完整 | 读取暴露有效的连续前缀。写句柄下次追加前截断不完整的尾部字节。 |
| 最后一个 Zstandard 帧结构不完整 | 按可用字节尝试解压，保留其中可验证的完整 JSONL 事件。下一次追加先截掉不完整帧，再将恢复出的事件持久写回。 |
| 完整帧校验失败或包含撕裂记录 | 视为损坏并拒绝打开。自动恢复的范围限于符合条件的末尾残缺数据。 |

代码先用 `scanZstdFrames()` 判定完整帧和未完成尾帧，再校验完整帧。最后一帧若确认不完整，才读取可用的解压前缀。恢复出的事件仍须通过格式验证，此时能确认的是事件前缀有效，整个残缺帧的校验和无法用于确认完整性。如果读到的逻辑前缀仍留着开放 turn，Agent Loop 会生成 `interrupted` 收尾，并用写句柄追加。该次追加会先完成待处理的物理修复，再写入收尾事件。[原始尾部读取](../deepseek-harness/packages/session/session-persistence-jsonl/src/index.ts#L900) · [压缩尾部读取](../deepseek-harness/packages/session/session-persistence-jsonl/src/index.ts#L1076) · [下一次追加前的回写](../deepseek-harness/packages/session/session-persistence-jsonl/src/storage.ts#L322) · [Agent 恢复](../deepseek-harness/packages/core/agent-loop/src/index.ts#L794)

## 5. 从历史格式读出当前 Session

后端按数字选出最高的规范文件代际，再核对 filename 与 header 中的版本。看到高于当前 writer 的版本时拒绝解读，并报告产物路径。即使同目录保留了更早文件，也按已选代际判断。旧文件可能只保留迁移时的历史，新事件会继续追加到当前代际，自动回退可能把已完成的操作误当作从未发生。[代际选择](../deepseek-harness/packages/session/session-persistence-jsonl/src/index.ts#L1651) · [校验与版本拒绝](../deepseek-harness/packages/session/session-persistence-jsonl/src/index.ts#L1289)

格式目录声明了 `v0 → v1 → v2 → v3 → v4` 的相邻转换。`stat()` 与 `list()` 读取 header，报告选中的格式状态。`open(id, 'read')` 可以解码历史正文、执行相邻转换并校验当前逻辑事件，只读查看不会发布后继。`open(id, 'write')` 先取得写入权，再准备并发布当前代际。中间格式按转换顺序解释，正常产物只有最终生成的后继文件。[静态格式目录](../deepseek-harness/packages/session/session-format-catalog/src/generated.ts#L15) · [读写打开](../deepseek-harness/packages/session/session-persistence-jsonl/src/index.ts#L479) · [格式状态记录](../deepseek-harness/docs/session-format-status.md)

写迁移先将当前格式编码到同目录临时文件并同步，随后用 Worker 校验。发布前比对已写字节与校验得到的摘要，重新检查源文件以及相关来源是否变化，再执行无覆盖发布。存在同名目标时会核对是否为满足预期的产物。旧代际的字节保持原样，历史读取准备可按修订键复用。这样既保留迁移证据，也避免用过期源数据悄悄覆盖已发布历史。[临时编码](../deepseek-harness/packages/session/session-persistence-jsonl/src/generation.ts#L713) · [校验与发布](../deepseek-harness/packages/session/session-persistence-jsonl/src/generation.ts#L883) · [历史准备与修订](../deepseek-harness/packages/session/session-persistence-jsonl/README.zh.md#use-this-package)

## 6. 面试官会怎么追问

### 调用方收到 `append()` 成功，可以立刻告诉用户数据不会丢吗？

先指出是内存 Session 接纳、通用持久化接口追加，还是 JSONL 写句柄已同步。对实时的 Agent 路径，还要确认 `session/flush` 已经排空缓冲，或有相应的已确认写入边界。继续追问可以要求候选人分别画出模型请求派发、顶层工具派发和 headless 输出前的持久性检查点。

### 断电后，一个压缩帧的校验和没写完，能保留它里面的事件吗？

先识别帧的完整性，然后说明可恢复尾帧只保留其中完整、可验证且与旧序号连续的事件。完整帧损坏属于另一类错误。下一次写入先处理残缺字节和可恢复记录，再追加新批次。追问可以把故障点放在恢复写回与新批次之间，要求候选人说明重试如何避免重复序号。

### 两个进程都想恢复同一个 Session，怎么保证顺序？

先让一个进程取得内核写锁，并把整次写句柄生命周期纳入锁的所有权。单个句柄内部再用串行队列和连续 `seq` 约束写入。云端设计可进一步讨论租约过期后的 fencing，尤其是旧 Worker 在长暂停后重新执行时，存储如何拒绝该次追加。

### 迁移保留了旧 v3 文件，为什么不能自动用它救场？

旧文件只能证明迁移时的那份历史。当前代际可能保存了之后的操作。可把失败拆成临时文件未发布、已发布后崩溃、版本过新或当前代际确实损坏。让候选人逐项说明权威文件在哪里，以及何时应该阻止恢复并由运维核对。

下一篇进入 SQLite 会话库，继续看同一套持久化接口怎样映射到 WAL、事务与共享分叉前缀。
