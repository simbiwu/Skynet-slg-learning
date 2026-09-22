# Engineering Decisions

这里记录当前已经确定的长期设计。Codex 不应无理由改动。

## D001 - 使用 Skynet v1.8.0

原因：

- 与已有 Skynet 学习项目保持一致；
- 避免课程过程中 Upstream 变化导致行为漂移；
- 便于源码调试和复现。

如果未来升级，需要单独提交 Decision，说明：

```text
版本变化
Lua 变化
API / Runtime 变化
测试结果
回滚方案
```

## D002 - Linux / WSL2

Server 按 Linux 环境实现。

Windows 只作为开发主机，不采用非官方 Native Windows Skynet Fork。

Codex 第一课先检查现有 WSL2 是否可用，禁止像旧课程那样默认注销或重装 Ubuntu。只有用户明确要求时才修改 WSL 安装。

## D003 - H5 使用 WebSocket + Custom Header + Protobuf

此项的单协议组合已由 D019 修订；保留原记录说明变更来由。

浏览器不能直接使用普通 TCP Socket。课程从第一课开始采用 WebSocket Binary Message，外层由浏览器和 Skynet `http.websocket` 处理 RFC 6455，项目自己的 Application Packet 使用固定 8-byte Header：

```text
version:uint8
flags:uint8
command:uint16 big-endian
sequence:uint32 big-endian
body:protobuf bytes
```

WebSocket 已经保留 Message Boundary，不再增加 2-byte Length Prefix。协议不使用 JSON 代替二进制链路。

Server 使用 lua-protobuf 0.5.3，Browser 使用 protobufjs 7.5.4。已发布的 Field Number 不修改、不复用。

## D004 - H5 Client 和 Node E2E 是长期工具

H5 Client 不是一次性 Login 页面。Node E2E 与浏览器复用相同 Header 和 Proto Schema。后续继续承担：

```text
login
enter_world
open_view
march
attack
bot load
fault case
```

因此连接、Sequence、Request/Response、Push 和 Codec 代码必须可复用。
第一课起两种协议入口都需要真实 WebSocket E2E；见 D019。

## D005 - 第一课 Memory Storage，第二课持久化

第一课目的是建立正确边界和真实链路，不把篇幅消耗在表结构和连接池。

但必须先有：

```text
StorageMgr / Storage Interface
```

PlayerAgent 不直接依赖一个随手写的全局 Lua table。

第二课第一次实现跨重启的 March 与 Scheduler 时，接入 MySQL 等持久化实现；保存顺序、重复事件和重启恢复必须与该行为一起验证。Player / World 业务接口不应因此重写。

## D006 - Region 是逻辑分片，RegionWorker 是执行单元

长期采用：

```text
Region -> Worker
```

而不是：

```text
Region == Service
```

第一课就按 Worker Pool 写。

## D007 - WorldMgr 只做 Router / Metadata

WorldMgr 不能变成整个世界的串行代理。

第一课为了简单，某些请求可以先经过 WorldMgr 做发现和路由；后续高频路径要允许缓存目标 Worker 地址或直接路由。

## D008 - World Grid 使用稀疏动态状态

不为整个世界每一格都创建完整 Lua table。

静态地图和动态世界分离：

```text
Static:
terrain / blocked / road

Dynamic:
city / resource / army / territory
```

第一课可以把默认地形简化成 Plain，但代码结构不能迫使未来给每个 Grid 建表。

## D009 - 首城创建使用幂等 Ensure

使用：

```text
ensure_player_city
```

不使用无条件：

```text
create_city
```

作为 Login / EnterWorld 自动流程。

重复请求必须返回同一个 City。

## D010 - 不在第一课实现最终 AOI

第一课只允许：

```text
query_rect
```

验证 Region Router 和 World Snapshot。

第二课必须升级成 Viewport Subscription + Incremental Push。

文档要明确这是阶段性接口，避免把全量 Query 误认为最终商业方案。

## D011 - March 采用事件驱动

后续 March 不允许每支 Army 高频 Tick 更新位置。

以：

```text
path + start_time + arrive_time + state/version
```

为核心。

## D012 - Battle 必须可确定性重放

战斗输入、seed、battle version 相同时，结果应一致。

## D013 - DB 不是跨 Actor 锁

不通过“先写数据库”来解决在线并发 Ownership。

在线一致性由业务 Owner、消息顺序、版本、幂等和补偿设计解决。

## D014 - 商业方案与课程实现必须分开说明

每个重要模块文档必须有三部分：

```text
本课实际实现
商业项目常见方案
本课暂时省略和以后替换点
```

禁止把课程规模直接描述成“行业标准”。

## D015 - ConnectionWorker 持有 Transport

`WebSocketGateway` 只监听并把 fd 分配到固定数量的 ConnectionWorker。ConnectionWorker 持有 fd、Connection Table、协议状态和 PlayerAgent Address；PlayerAgent 持有玩家状态和逻辑 Connection Owner/ID，不直接读取 Socket，也不依赖 Protobuf Runtime。

登录后的业务 Packet 仍先进入 ConnectionWorker：

```text
WebSocket Frame
-> Header / Protobuf Decode
-> PlayerAgent internal Lua message
-> Response Table
-> Protobuf / Header Encode
-> WebSocket Frame
```

这样 Codec 错误、畸形包和 fd 生命周期留在接入边界。ConnectionWorker 是固定 Pool，不采用一连接一 Lua State。
第一课起 Gateway 持有两个监听端口，并把端口决定的协议模式交给 ConnectionWorker；业务 Service 不区分协议模式，见 D019。

## D016 - H5 整数和协议兼容

第一课所有业务 ID 使用 `uint32`，JavaScript `Number` 可以精确表达。未来如果使用 `uint64`，H5 必须采用 String、protobufjs Long 或 BigInt 的明确方案，不能直接把任意 64-bit ID 转成 `Number`。

本地使用 `ws://127.0.0.1`。生产部署使用 `wss://`，通常由反向代理或负载均衡终止 TLS，并在 Upgrade 前检查 Path、Origin、认证和限流。

## D017 - 首城稳定路由到 Home Region

第一课使用稳定函数把玩家映射到 Home Region：

```text
home_region_id = stable_hash(player_id) % region_count
worker_array_index = (home_region_id % worker_count) + 1
```

首城候选位置只在 Home Region 内探测。RegionWorker 为每个 Region 保存 `city_by_player_id`，同一玩家的重复 `ensure_player_city` 总会回到同一个 Owner。跨 Region 出生、选州、联盟出生和拥挤迁移留到后续课程。

## D018 - 长时间任务不依赖玩家在线

Skynet C Runtime 已有分层 Time Wheel，少量活跃任务可以直接注册长 `skynet.timeout`。课程禁止每个对象每秒轮询。

玩家私有且不影响其他人的建筑、科技和训练任务保存 `finish_at/version/state`，可以在登录或访问模块时 Lazy Settlement。世界行军由 Region/World Owner 持有，玩家下线不会删除行军。

Runtime Timer 只负责唤醒，持久化记录才是 Crash Recovery 来源。任务统一携带：

```text
task_id
version
finish_at
state
```

单 Shard Timer 数量达到需要集中管理的规模后，再根据测量结果引入 TimerShard + Min Heap、分桶或应用层 Timing Wheel。

## D019 - 双端口协议接入，同一业务链路

用户明确要求 Server 同时监听两个端口，并用两套 Wire Protocol 进入同一套 Login / Player / World 业务。第一课两个端口都使用 WebSocket Binary Message 和 `/game` Path：`8890` 为 Protobuf Envelope + Protobuf 业务 Body，`8891` 为 8-byte Header + 自定义二进制 Body。测试端口分别为 `18890`、`18891`。端口决定协议，不按 Payload 猜测，也不允许同一连接切换。两个 Codec 转成相同内部 Lua Table，ConnectionWorker 再执行相同的认证、绑定和业务分发。

原 D003 的 Header + Protobuf Body 是有效的协议组合，但未满足双协议教学目标，因此停止作为第一课唯一协议。保留原 Command ID 1/2/3、Sequence 0 保留、64 KiB 上限、Skynet v1.8.0、lua-protobuf 0.5.3、protobufjs 7.5.4、ws 8.18.3。此变更需要同时修改 Gateway、ConnectionWorker、H5、Node E2E、Protocol Unit Test 和第一课实操文档。已有旧格式客户端不能直接连接新端口；迁移时应给旧客户端保留独立监听或显式升级版本，不能静默将旧 Payload 当新协议解析。

Protobuf Descriptor 在构建阶段从 `protocol/game.proto` 生成 `protocol/game.pb`；ConnectionWorker 只加载生成产物，不在普通 Service 启动时调用 `protoc` 编译 Schema。

## D020 - WSL npm 必须使用 Linux Node

现场发现 WSL 未安装原生 Node/npm 时，`npm` 仍可能解析到 Windows 的 `/mnt/c/Program Files/nodejs/npm`。第 2、7 节在执行 `npm install` 前检查 `command -v node/npm` 和 `node -p 'process.platform'`；平台必须为 `linux`，npm 路径不能来自 `/mnt/`。否则先在 Ubuntu 安装原生 `nodejs npm`，再生成或恢复 Lock File，避免把 Windows 的安装产物混入 Linux 工程。

## D021 - 协议模块统一归顶层 protocol

Login 首次跑通后，在 EnterWorld 阶段把协议源、构建工具、生成物和两端编解码集中到顶层 `protocol/`。运行时代码按职责命名为 `server_protobuf_codec.lua`、`server_custom_binary_codec.lua`、`server_protocol_dispatch.lua` 和 `h5_dual_protocol_codec.mjs`；生成资料为 `generated_server_commands.lua` 与 `generated_h5_commands.mjs`。`8890` 的 Envelope 与 Protobuf Body 同处 Server Protobuf 模块；`8891` 的 8 字节 Header 与自定义 Body 同处 Server 自定义二进制模块。共用模块只按连接的端口模式选择 Codec，不重复解析字段。两条链路继续产生同形内部请求，业务 Service 不依赖外部字节格式。

这次调整不更改两端口的 Wire Contract。学习者先按原文件完成 Login，在 EnterWorld 阶段按教程迁移；验证两端口后删除旧入口，不要求重做已完成的 Login。后续新增命令的扩展规则见 D023。

## D022 - 普通业务命令不修改 Gateway 和 ConnectionWorker

Gateway 只处理监听、端口模式和连接分配。ConnectionWorker 只处理 WebSocket 生命周期、Login 认证绑定、通用编解码调用和已登录请求的统一转发。EnterWorld 阶段把初始只支持 Login 的 ConnectionWorker 扩成通用转发，是第一课接入层的一次成型步骤；此后新增普通业务命令不应再修改 Gateway 或 ConnectionWorker。新增命令要在协议定义、目标业务 Owner 的处理入口和客户端操作中落地，并覆盖两端口行为；字段编解码由协议工具链承担。只有连接生命周期、认证、安全或传输层语义变化才允许修改接入层。

## D023 - 命令资料生成与自定义二进制通用 Body Codec

Login 阶段先手写一个 Body 看清字节布局。增加 EnterWorld 时建立 `protocol/commands.json`，发布 Command ID、Protobuf 消息名和 `8891` Body 的有序字段布局；`protocol/build.sh` 生成 Lua 与 JavaScript 可加载的命令资料。`game.proto` 继续定义 `8890` 的 Protobuf Field Number，构建工具解析 Proto 并核对两份源文件的字段名称、类型与顺序。运行期 `server_protobuf_codec.lua` 用命令资料选 `pb.encode/decode` 的类型，`server_custom_binary_codec.lua` 用同一资料递归编解码已支持的字段类型；H5 与 Node 复用 `h5_dual_protocol_codec.mjs`。新增普通命令时编辑清单、Proto 和业务 Owner，重新构建并补客户端操作与必要测试，不修改 Gateway、ConnectionWorker 或通用 Codec。要增加新的字段类型或改变连接语义时才修改通用 Codec 或接入层。

自定义二进制的字段顺序、整数宽度、Presence 和数组长度都是公开的 Wire Contract；已发布版本不能静默改变。生成工具检查重复 ID、字段类型及两份协议定义的一致性；协议与真实 WebSocket 测试检查两端语义一致。构建产物不由普通 Service 启动时动态生成。
