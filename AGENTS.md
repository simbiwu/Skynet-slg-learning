# AGENTS.md - Codex rules for the Skynet SLG learning repository

## P0 purpose

这个仓库用于训练商业 SLG 服务端能力，不是为了快速堆出一个能跑的教学 Demo。

文档、代码注释、自动化测试、调试入口、状态 Ownership、失败恢复、并发语义和可验证性全部属于 P0。课程可以缩小地图、玩家数量、兵种、技能和玩法规模，但不能用错误的架构捷径掩盖真实商业项目需要解决的问题。

## Learner profile

- 学习者长期从事 C++ + Lua MMO / ARPG 服务端，熟悉传统 GameServer、Gate、DB、Center、跨服、AOI、活动、排行榜、网络和线上运维问题。
- 已经具备 Lua 和 C++ 工程经验，不要默认从 Lua 语法、TCP、Hash Table、MySQL 基础、线程基础开始讲。
- 对 Skynet 已有学习基础，知道 Service、`skynet.call`、`skynet.send`、`skynet.dispatch`、Lua State 等概念，但本项目仍要在真实调用链中解释其并发语义。
- 训练目标是能够接手、设计、Review 和排查商业 SLG Server，而不是“会运行示例”。

## Course constraints

- 整个主课程固定为 4 课。四课完成后，学习者应能接手、设计、Review 和排查成熟 Skynet SLG Server 的主要链路，并能用同一工程展示这些能力；课程不以课时数压缩状态归属、持久化、并发、恢复、负载验证或运维内容。
- 每一课是一个完整能力域，不按 API 或小知识点切成十几课。
- 所有课程在同一个仓库持续演进，禁止建立 `lesson01_server`、`lesson02_server` 之类互不相干的重复工程。
- 前一课的代码必须能被后一课继续使用；如果架构需要替换，必须解释为什么替换、迁移成本和商业项目中的常见做法。
- 业务规模可以简化，架构问题不能简化。
- 不要求把所有商业功能写到完整产品级，但必须说明真实项目通常采用什么方案、为什么、有哪些替代方案、当前课程省略了什么。
- 四课的能力域依次为：① Login、双端口接入与首次进入世界；② 可持久恢复的世界行为，包括视野、行军、长事件和跨 Region 移交；③ 多人冲突与联盟，包括确定性战斗、争地、领土和集结；④ 多节点部署、赛季、性能、可观测性与故障演练。第二课开始持久化世界长事件，并同时验证重启恢复。

## 教学顺序与已有进度

第一课曾在 PlayerAgent 和主城业务出现前集中铺开 Wire Contract、Protobuf Schema、Descriptor 构建和多个 Codec。后来虽然把网络章节移到业务后面，却又把 Login、EnterWorld、QueryWorld 全部排在网络前，导致学习者完成内部 Login 后，仍要先学 World、Region 和坐标，才能在浏览器看到第一次 Login Response。两次调整都没有按一条完整的用户行为检查章节顺序。后续课程必须以此为反例；“最终架构需要这些模块”不能作为提前创建全部文件的理由，也不能作为推迟当前行为的实际使用的理由。

每课先说清楚玩家或运维人员要完成的具体动作、可观察结果和失败情况，再写这条动作需要的状态 Owner、内部调用和验证。第一课应先让 Login 创建或复用基础 PlayerAgent、加载玩家快照，并在接入网络后让 H5/Node 实际发送 Login、收到 LoginResponse；完成这一条链路后，才进入 EnterWorld 的主城、坐标和 Region，再扩展 QueryWorld。第二课也从一个具体可操作的世界行为开始，不能先堆 Army、March、AOI 的抽象结构和全部协议字段。

- 每次只引入当前动作必需的文件、函数和字段。同一个文件可以在后续步骤扩展；写明新增或替换的位置。不得为了让代码看起来一次成型，提前加入后续行为的 Schema、Codec、Service 或状态字段。
- 协议专属文件名或目录必须直接表明它服务哪个协议；共用分发模块只做选择与接口统一。不能仅因未来会增加 Command 就抽出只包装一次编码 API 的文件。新增文件前先画出当前请求经过的调用链，判断同一协议的 Header 和 Body 放在一个模块是否更容易维护。
- 一条业务按实际使用顺序讲解：给出具体请求和结果，确定 Owner，完成最小内部调用，再接入已有或刚好需要的外部入口。面向客户端的业务应尽快从客户端用起来，不能为了“先做完业务层”转去实现下一条业务的 World、AOI、March 等能力域。讲解中可以运行代码观察结果，不把每一小步都做成单独的 Smoke Test 或验收关卡。
- 第一课内部 Login 路径明确后即可引入 WebSocket，只为 Login 增加必要的 Protobuf Schema、Codec、连接处理与 H5/Node 请求。先让一个端口完成 Login Request/Response，再让第二个端口进入同一 Login 业务；之后按 EnterWorld、QueryWorld 的实际需求扩展字段和处理函数。双端口是最终约束，不要求一次讲完所有协议文件。
- 第二课复用第一课已有的网络入口。新增世界行为时，内部语义清楚后就接上该行为的真实请求、响应或推送，让学习者实际使用；只有该行为确实需要时才扩展 Wire 字段、AOI 或调度。纯内部能力用调用过程和必要的运行观察讲清楚，不强制另建测试 Service。
- 一节结束时自然说明当前能做什么、下一节为什么需要新增概念；不为凑齐固定格式重复列文件、测试和验收清单。不能让只有抽象 Contract、没有可操作结果的一节突然跳到另一能力域。
- 发布或大幅重排实操文档前，沿目录模拟学习者从空白工程读下去：每个新概念是否服务于眼前的行为，Login 能用起来之前是否插入了 World 等无关能力域。发现顺序错误时审查整条学习链，不只挪动被指出的章节标题。这个检查由编写者完成，不写成学习者每节都要执行的流程。
- 已完成的阶段、依赖和文件不要求学习者重做。调整课程顺序时，必须写清旧进度如何接到新步骤；已创建的文件可以保留，后续按新步骤补充或替换。
- 架构边界可以先说明到足以避免写错 Owner 的程度；不要因此提前展示整套未来代码。课程资料中的“最终链路”和“当前要写的步骤”必须明确区分。

## Fixed baseline

- Skynet: v1.8.0
- Lua: Skynet bundled modified Lua 5.4.7
- Runtime: Linux / WSL2
- Client: H5（第一课使用原生 HTML + JavaScript）
- Transport: WebSocket Binary Message；本地开发使用 `ws://`，生产使用 `wss://`
- Protocol: 两个 WebSocket Binary 端口；Protobuf Envelope + Protobuf body (`8890`)；8-byte custom application header + 自定义二进制 body (`8891`)
- Custom Header: `version:uint8 + flags:uint8 + command:uint16 BE + sequence:uint32 BE`
- Server Protobuf: lua-protobuf 0.5.3 (`ee4beb3865e2b82ea94b8a4314d78875c550ce20`)
- Browser Protobuf: protobufjs 7.5.4
- Node E2E WebSocket: ws 8.18.3
- Lesson 1 storage: memory-backed Storage abstraction
- Test Client: H5 manual client + Node.js E2E client，二者覆盖两个端口并复用同一份内部业务语义与各协议 Codec
- LuaPanda: optional and opt-in only; normal/test startup must not depend on it

不要静默升级 Skynet、Lua 或协议栈。任何版本变化都必须单独说明原因、兼容风险和迁移影响。

## Core architecture rules

### 1. Ownership before code

修改任何 Service 或状态前，先写清楚：

1. 这份状态的唯一权威 Owner 是谁；
2. 谁可以写；
3. 谁只能缓存或读取；
4. 哪些消息会跨 Owner；
5. Crash 后从哪里恢复；
6. 是否存在重复消息、旧 Timer、旧 fd、旧 generation 或旧 version。

如果一份业务状态同时存在三份“都能写”的副本，必须先修正设计再继续开发。

### 2. Do not confuse logical partition and Service count

Region 是逻辑世界分片，不等于一个 Region 必须对应一个 Skynet Service。

本项目从第一课开始采用：

```text
Logical Region
      ↓
RegionWorker Pool
```

多个 Region 可以由一个 RegionWorker Service 持有。路由可以使用固定规则，例如：

```text
worker_id = region_id % worker_count
```

课程规模可以只有 16 个 Region 和 4 个 Worker，但文档必须说明真实项目会如何根据 CPU、对象密度、热点和消息量调参。

禁止把上千 Region 机械映射成上千 Lua State，再把它描述成商业默认方案。

### 3. Manager services are not permanent hot proxies

`WorldMgr`、`PlayerMgr`、`AllianceMgr` 这类 Manager 负责：

- 生命周期；
- 路由；
- 元数据；
- Service 发现。

不要让所有高频业务永远经过 Manager。

例如世界高频消息后续应尽量路由到目标 RegionWorker，而不是：

```text
Player -> WorldMgr -> RegionWorker
```

永远多一跳。

### 4. Skynet coroutine correctness

不要把 Skynet Service 简化理解为“永远不可重入的单线程 GameServer”。

一个 Service 中：

```lua
local old = state.value
local r = skynet.call(other, "lua", ...)
state.value = old - r
```

在 `skynet.call` 处会 yield。当前 coroutine 挂起期间，其他消息可以进入同一个 Service 并修改 `state`。恢复以后 `old` 可能已经过期。

任何重要修改都必须审查：

1. 哪些调用会 yield；
2. yield 前读取了哪些可变状态；
3. 恢复后是否重新验证；
4. `call` 是否真的需要；
5. 能否改成 `send` / event；
6. 是否需要 generation/version/idempotency key；
7. 是否制造深同步 RPC 链。

### 5. KISS, but not fake simplicity

- 不在 Skynet 上再造第二套 Actor Framework。
- 模块组合优先于复杂 BaseService 继承体系。
- 不要每个业务模块都做成 Service。
- 不因为“商业级”就提前加入 Redis、Kafka、Etcd、Kubernetes、分布式锁。
- 只有当前课程出现真实需求时才引入新基础设施。
- 优化必须来自 Benchmark / Profiling，不允许仅凭“Lua 慢”作结论。

## SLG state ownership baseline

课程演进过程中至少保持下面的边界：

```text
PlayerAgent
  玩家个人持久状态、个人队列、英雄/资源等玩家私有业务状态

WorldMgr
  世界元数据、Region -> Worker 路由、生命周期
  不持有高频 World Object 权威状态

RegionWorker
  所属逻辑 Region 内的空间权威状态：
  City world marker / Resource / Army spatial state / Territory spatial state 等

Scheduler
  长时间事件的调度权威记录和触发
  不能依赖“每对象每秒 Update”

Battle
  确定性战斗计算；输入、seed、版本确定时结果可重放

Alliance
  联盟成员关系、权限、联盟业务状态
  与 World/Territory 的归属边界必须显式定义

Storage
  持久化接口和恢复来源
```

具体字段可以随课程调整，但每次调整必须更新架构文档。

## SLG mandatory review questions

任何涉及 World / Army / March / Timer / Territory / Alliance 的改动前，至少回答：

1. 对象当前属于哪个 Region / Worker？
2. 跨 Region 时谁先失去 Ownership，谁后取得 Ownership？
3. 中途失败是否可能导致对象消失或重复？
4. Command 是否可以安全重试？
5. 是否有 idempotency key / version / generation？
6. 旧 Timer 在重启或状态改变后还能否误触发？
7. Server Restart 后该对象从哪里恢复？
8. 两个玩家同时争抢一个 Grid，最终串行点在哪里？
9. 热点战争时是否会把某个 Worker 的 Message Queue 打爆？
10. AOI 是全量查询还是增量订阅？是否有 Batch / Merge / Backpressure？

## Documentation rules

课程示例和实际工程遵循 `docs/CODE_STYLE.md`。业务身份、生命周期与状态变量使用能看出含义的名称，不以 `c`、`p`、`v` 等单字母代替 Connection、Packet、Response；不同用途的变量分行初始化，条件分支和函数体不要压成一行。关键函数首次出现时，读者应在函数附近知道调用方、参数来源与形状、返回值及可选字段、Owner 状态修改、yield 和失败去向；跨模块或跨 Service 的 Table 必须给出字段契约。`pcall` 的成功标记与“结果或错误”分别命名。修改示例时连同后续替换版本、文字和测试一起核对。

任何有意义的行为或架构变化，都必须同步更新 `docs/` 中对应文档。

重要路径必须按实际执行顺序讲解：

```text
入口
-> Service
-> 消息
-> yield
-> resume
-> state owner
-> response / event
```

必须注明：

- 仓库完整路径；
- 关键函数；
- 当前 Process / Thread / Service / Lua State / coroutine；
- 参数从哪里来；
- 哪里修改状态；
- 哪里会 yield；
- 失败时返回到哪里；
- 如何验证。

不要只列 API。

## 写作风格要求（强制门禁）

面向学习者的课程、专题文档、代码说明和工程讲解，必须使用自然、克制、像有实际经验的人写出来的中文，不得使用典型的 AI 写作腔。

1. 先讲具体事实、判断和结论，不要用空泛的开场白铺垫。
2. 不要为了显得完整而强行分成大量小标题，不要每两三句话就列一组项目符号。
3. 避免机械使用“首先、其次、再次、最后”“总的来说”“综上所述”“值得注意的是”“需要强调的是”等连接词。
4. 禁止频繁使用“不是……而是……”“不仅……更……”“这意味着……”“本质上……”“真正的……”“关键在于……”等模板句式。
5. 不堆砌正确但无用的废话，不反复换一种说法重复同一个结论。
6. 不为了制造节奏而大量使用短句、排比句、反问句和口号式表达。
7. 不擅自拔高主题，不做情绪升华。
8. 长短句自然交替。能用一段话讲清楚的内容，不拆成五六个条目。
9. 技术内容解释因果关系和实际运行过程，不只罗列概念、术语和优缺点。
10. 缺少可靠信息时直接说明不确定，不用听起来合理的内容填补空白。
11. 写完后删除空洞开场、重复结论、无信息过渡、模板化金句和无必要小标题。
12. 文字应像熟悉该领域的主程在认真向同行说明问题，不像营销软文或标准化培训材料。
13. 写作以实际执行过程为主线：
    - 从真实入口、请求或故障现象开始，沿代码发生顺序讲解；
    - 说明请求经过哪些 Service、在哪里挂起、在哪里恢复、数据由谁持有；
    - 明确当时所在的 Process、Thread、Service、Lua State 和 coroutine；
    - 顺着 Message Path 解释参数、状态修改、yield 和恢复后的失效条件；
    - 跨 C Runtime、Lua Runtime 和业务 Service 时按调用顺序切换层次；
    - 术语和 API 在执行过程中第一次出现时就地解释；
    - 追踪 Response、Error 和 Exit 最终回到哪里；
    - 伪代码必须说明省略环节，不能改变真实语义。
14. 每个概念都说明解决的问题、内部工作方式、工程用法和常见错误。不能把 Actor、coroutine、消息驱动、高并发、解耦当成解释。
15. 可以与传统 C++ + Lua MMO Server 比较，但只在有助理解时比较。
16. 所有代码标明从仓库根目录开始的完整路径，并解释关键代码为什么这样写。
17. API 必须放进具体消息路径，说明调用方、接收方、参数、返回方式、yield 和失败表现。
18. 性能、并发和内存结论必须给出负载、Service 划分、yield、队列、数据规模或测量环境等成立条件。
19. 所有关键代码必须写 WHY 注释，不能只写 WHAT。

示例代码检查覆盖整份实操文档，不能只修正用户指出的一段。每个关键函数首次出现时说明参数由哪一层传入、返回给谁、修改哪份 Owner 状态，以及是否会 yield；递归、闭包和回调要说明参数在后续调用中如何变化。关键局部变量要说明它保存的身份、索引或生命周期信息。若代码仅为跨 Service Table 隔离而额外复制数据，先检查 Skynet 消息序列化是否已经提供隔离，不得用看似合理的注释为多余实现背书。修订后逐段核对示例与文字、测试和既有工程进度。

## H5 / WebSocket / Protobuf rules

- Gateway 和 ConnectionWorker 是接入层：Gateway 负责监听、连接分配与协议模式；ConnectionWorker 负责 WebSocket 生命周期、认证绑定、通用编解码调用和向业务 Owner 转发。新增普通业务 Command 不得修改这两个 Service 的命令分支。新增字段或 Body 布局先修改协议定义并重新构建；只有新增通用字段类型时才扩展 Codec。新增业务处理只修改目标业务 Owner 的分发与实现。确实改变连接生命周期、认证、安全策略或传输协议时，才审查接入层代码。
- 第一课从 EnterWorld 开始把协议源、构建工具、生成物和编解码器集中到顶层 `protocol/`；文件名要说明 Server/H5、Protobuf/自定义二进制以及职责。`protocol/commands.json` 是命令号、Protobuf 消息名和自定义二进制字段布局的发布清单；`game.proto` 仍是 Protobuf Field Number 的权威来源。构建时校验两份定义的字段一致性并生成 Lua/JS 命令资料。新增普通命令不再手写 Server/H5/Node 三套同形字段编解码分支，也不改通用协议选择模块；只扩展协议定义、目标业务 Owner、客户端操作和必要测试。只有新增字段类型时才扩展通用 Codec，并说明兼容性。
- 一个 WebSocket Binary Message 对应一个完整 Application Packet，不再增加 TCP Length Prefix。
- Gateway/ConnectionWorker 按监听端口处理 WebSocket、Protobuf Envelope/Body 或 Custom Header/Body；两条链路转为同一内部请求，PlayerAgent 不解析外部 Wire Format。
- ConnectionWorker 持有 fd、连接状态和 generation；PlayerAgent 只持有逻辑 Connection Owner/ID。
- 两端口只接受 Binary Frame，限制 Application Packet 大小，拒绝未知 Version、非法 Flags、未知 Command、畸形 Protobuf 和畸形自定义二进制 Body。
- 已发布的 Protobuf Field Number 不修改、不复用。删除字段使用 `reserved`。
- 第一课业务 ID 使用 `uint32`。未来需要 `uint64` 时，H5 使用 String/Long/BigInt 方案，禁止无说明地转为 JavaScript `Number`。
- 本地开发可以使用 `ws://`。生产必须使用 `wss://`，并在 Upgrade 前处理 Origin、Path、认证和限流。
- LuaPanda、Proto 动态加载和开发静态服务器不能进入普通测试或生产启动路径。

## Practical teaching mode

Codex 默认不是“替用户一次写完整个项目”，而是工程教练。

第一课接手时：

1. 读取本资料包；
2. 检查当前目录、WSL、编译器、Git、Skynet/第三方现场；
3. 完整阅读并按现场校验 `docs/Skynet_SLG第一课_从H5登录到进入世界_实操.md`；只有现场或设计变化时才修订，不重复生成；
4. 实操文档必须能让学习者从空白 WSL 工程逐步创建代码；
5. 再按文档分阶段实施、运行、测试、调试；
6. 在关键行为可运行时说明如何观察结果；不要给每个小步骤增加单独的 Smoke Test 或正式验收；
7. 不要在没有解释的情况下自动生成几十个文件把课程直接“做完”。

第二课沿用第一课同一工程、网络入口和测试入口。先选定第一条具体的世界行为，说明它如何改变 Player/Region/World 状态；内部调用讲清楚后，接上该行为需要的请求、响应或推送，让学习者实际操作。AOI、消息推送和新 Wire 字段按当前行为的需求逐步加入。每次扩展都给出从上一课可运行版本到当前版本的修改步骤，不让学习者重新创建一套工程。

第三课从可操作的多人争地行为开始，引入 Battle、Territory、Alliance 与 Rally 所需的状态和协议；每个跨 Owner 写入都说明最终串行点、幂等和失败恢复。第四课把同一工程部署为多 Skynet 节点并完成赛季、压测与故障演练；不能把 Cluster API 调通当作分布式正确性验证。

如果用户明确要求 Codex 直接实施某一阶段，可以实施，但仍必须同步文档和测试。

## Test rules

行为变化后运行统一测试入口：

```bash
./scripts/linux/test.sh
```

测试应覆盖有实际风险的协议、状态和并发行为；优先扩展已有测试。不要为每个教学小节、每个 Service 或每次概念引入创建独立 Smoke Test。教学中的临时手工运行可帮助观察调用链，但不作为强制验收关卡。

后续 AOI / March / World 性能变化增加专用 Benchmark。

发现 Bug 必须补 Regression Test。

课程后期至少要有：

- protocol/unit test
- E2E real WebSocket test
- restart/recovery test
- duplicate/retry/idempotency test
- concurrent world conflict test
- bot load test
- AOI / hot region benchmark
- failure injection case

## Do not

- 不静默升级 Skynet/Lua；
- 不换非官方 Native Windows Skynet Fork；
- 不静默替换双端口 WebSocket 协议栈；
- 不把 WebSocket Frame Header 与项目 Application Header 混为一层；
- 不让 PlayerAgent 直接依赖 protobufjs、lua-protobuf 或 Socket fd；
- 不删除教学注释只为了代码短；
- 不用宽泛 `pcall` / fallback 把错误吞掉；
- 不把所有逻辑塞进 PlayerAgent；
- 不把所有 World 消息永久塞进 WorldMgr；
- 不让每个 Grid / Region 都变成一个 Service；
- 不用每秒轮询所有建筑、March、资源生产对象；
- 不把 DB 当成跨 Service 锁；
- 不把“加 Redis/Kafka”当成架构成熟的证明；
- 不在没有测试数字的情况下宣称某实现“性能更好”。
