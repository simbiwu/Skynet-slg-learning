# Lesson 01 Spec
# 从空仓库到可进入世界的 SLG Server

## 1. 第一课目标

第一课结束时，仓库必须已经形成一个可以长期继续开发的 Server + Test Client 基础，而不是只运行几个 Service 的内部 Demo。

真实链路：

```text
Chrome H5 / Node E2E
  -> :8890 WebSocket Binary Message / Protobuf Envelope + Protobuf Body
  -> :8891 WebSocket Binary Message / 8-byte Header + 自定义二进制 Body
  -> WebSocketGateway
  -> ConnectionWorker / 对应 Wire Codec / 相同内部 Request
  -> Auth
  -> PlayerMgr
  -> PlayerAgent
  -> Memory Storage
  -> enter_world
  -> WorldMgr
  -> RegionWorker
  -> ensure_player_city
  -> query_rect
  -> Response
  -> 按原连接协议编码 Response
  -> H5 / Node Client
```

正确 Token：

```text
login ok
enter_world ok
world snapshot contains own city
```

错误 Token：

```text
AUTH_FAILED
```

重复进入世界：

```text
返回同一 City
```

不能产生两个主城。

## 2. 第一课不是简单 Demo

必须保留下面的长期结构：

```text
service/
  protocol/
  gateway/
  auth/
  player/
  storage/
  world/

lualib/
  protocol/
  world/
  common/

client/

tests/

scripts/

config/

docs/
```

具体文件名由 Codex 根据实操文档设计，但不能为了第一课方便把所有逻辑写进：

```text
main.lua
```

或：

```text
player_agent.lua
```

## 3. 第一课 World 参数

课程默认可以使用：

```text
World: 256 x 256
Logical Region: 64 x 64
Region Count: 4 x 4 = 16
RegionWorker Count: 4
```

路由示例：

```text
logical_worker_id = region_id % 4
lua_array_index = logical_worker_id + 1
```

这个参数只是课程值，不是行业标准。

## 4. World 数据模型

第一课只需要 City World Marker。

最小：

```text
WorldObject
  id
  type = CITY
  owner_player_id
  x
  y
  version
```

需要至少两个索引：

```text
object_id -> object
position_key -> object_id
```

不为所有 256 x 256 Grid 创建完整动态 table。

## 5. Player 数据模型

第一课最小玩家数据：

```text
player_id
name
city_id
resource (可选很小)
```

`city_id` 是引用。

City 的空间位置权威状态在 RegionWorker。

PlayerAgent 不直接维护：

```text
city.x
city.y
```

作为可写权威副本。

## 6. RegionWorker

每个 RegionWorker Service 持有多个 Logical Region。

示例：

```text
Worker 0:
  Region 0
  Region 4
  Region 8
  Region 12
```

内部可：

```lua
regions[region_id] = {
    objects_by_id = ...,
    object_at_pos = ...,
}
```

第一课必须让学习者在 Debug Console 中看到：

```text
16 Logical Region
4 RegionWorker Service
```

从现场理解 Region != Service。

## 7. WorldMgr

负责：

```text
world config
region layout
region -> worker route
worker lifecycle
```

第一课允许：

```text
PlayerAgent -> WorldMgr -> Worker
```

用于讲清链路。

但文档必须明确：

后续高频 World 消息不能永久让 WorldMgr 成为所有请求的同步代理。

## 8. 首城创建

不要使用无条件：

```text
create_city
```

进入世界自动流程必须使用：

```text
ensure_player_city(player_id, preferred_position or spawn_rule)
```

要求：

1. 同一玩家重复调用返回同一 City；
2. City 已存在时不重复占格；
3. 位置冲突时有明确 Spawn Rule；
4. 最终格子占用检查发生在 RegionWorker；
5. 不能只在 PlayerAgent 先查后写。

第一课可以使用非常简单的可重复出生点算法，例如基于 player_id 计算候选点，并逐步探测空格。

第一课必须先稳定计算：

```text
home_region_id = player_id % region_count
```

同一玩家的 `ensure_player_city` 总是路由到同一个 Home Region。候选点只在该 Region 内探测，RegionWorker 使用 `city_by_player_id` 保证重复调用返回同一对象。跨 Region 出生留到后续课程。

但必须写清：

商业项目通常还会考虑：

```text
出生州
新手保护
联盟出生
地形
资源分布
服务器拥挤
迁城
```

第一课不实现这些玩法。

## 9. Storage

第一课可以沿用旧项目思路：

```text
StorageMgr
-> MemoryWorker Pool
```

但数据所有权要写清：

Memory Storage 是“持久化替身”，不是 Player 在线状态 Owner。

PlayerAgent 在线时拥有 Player 的运行态。

Storage 通过 Skynet Message 返回 Snapshot。跨 Service 参数与返回值经过序列化，各 Service 的 Lua State 不共享同一个 Lua Table 引用；本课不为此再做一次递归深拷贝。

## 10. 协议

两个监听端口各接收一个完整 WebSocket Binary Message。Protobuf 端口 `8890` 的 Message 是 `.slg.Envelope`，包含 `version/flags/command/sequence/body`，其中 Body 是相应的 Protobuf 业务消息。自定义二进制端口 `8891` 的 Message 使用：

```text
version:uint8
flags:uint8
command:uint16 big-endian
sequence:uint32 big-endian
body:custom binary bytes
```

第一课两端口只接受 Binary Frame，Application Packet 上限为 64 KiB。监听端口决定 Codec，不按 Payload 猜测。Codec 在 ConnectionWorker 中结束，两条链路都转为同一内部 Lua Table 和 Command ID；PlayerAgent 不依赖 Protobuf 或自定义 Wire Format。

建议至少有：

```text
login
enter_world
query_world
```

`enter_world` Response 可以包含：

```text
player
city
world basic metadata
```

`query_world`：

```text
min_x
min_y
max_x
max_y
```

返回该矩形内 World Object。

第一课 `query_world` 是阶段性验证接口。

第二课升级为 AOI Viewport Subscription。

## 11. H5 Client 和 Node E2E Client

不能写成：

```text
只会发一次 login 然后退出
```

H5 Client 应从第一课形成可复用结构：

```text
connect
request/session
login
enter_world
query_world
close
```

Node E2E Client 分别连接两个真实 Skynet WebSocket 端口，复用相同业务语义和 Sequence 规则，各自按 Protobuf Schema 或自定义二进制布局编码。H5 可选择端口进行手工操作和 Chrome DevTools 验收，Node Client 负责双协议自动回归。

后面能继续加入：

```text
open_view
march
recall
attack
bot
```

## 12. Coroutine / yield 强制分析点

实操文档必须沿至少一条请求说明：

```text
H5 enter_world
-> ConnectionWorker
-> PlayerAgent
-> WorldMgr
-> RegionWorker
-> return
```

标出每一个：

```text
skynet.call
```

在哪里 yield。

特别检查：

### ConnectionWorker

连接状态由 `fd + connection_id + table identity` 共同标识。`call` 恢复后必须重新验证连接。

恢复后必须重新验证 connection / generation。

### PlayerAgent

如果：

```lua
local city_id = player.city_id
skynet.call(world, ...)
player.city_id = ...
```

要解释其他消息能否在 yield 期间进入并修改 player。

同一玩家客户端业务第一课应保持串行策略，或者使用明确 queue / state machine。

### WorldMgr

不要在持有临时可变路由状态时跨长 call。

### RegionWorker

最终空间写竞争应在同一个 Worker/Region Owner 内串行化。

## 13. `call` 与 `send`

第一课每一个跨 Service 同步调用都要解释是否确实需要 Response。

例如：

```text
login -> need response
ensure city -> need response
query -> need response
```

日志、非关键通知不要为了方便全部 `call`。

## 14. 第一课调试要求

实操文档应包含：

### Skynet Debug Console

至少能检查：

- Service 列表；
- PlayerAgent；
- 4 个 RegionWorker；
- Message Queue / Service 状态（按官方能力）；
- WebSocketGateway 和固定数量 ConnectionWorker。

### GDB

至少追踪：

```text
main
skynet_start
skynet_context_new
snlua_create
```

目的不是重复旧课程全部 Runtime 教学，而是确认：

```text
一个 Process
多个 Worker Thread
多个 Service Lua State
RegionWorker 只有 4 个 Lua State
```

### Lua Debugger

LuaPanda 可选。

正常启动和测试不能依赖它。

### Chrome DevTools

必须能在 Network / WebSocket Messages 中找到两个端口的 Login、EnterWorld、QueryWorld Binary Message；自定义二进制端口把 Header 字节对应回 Version、Command 和 Sequence，Protobuf 端口从 Envelope 字段解释这些值。

## 15. 第一课自动化验收

统一：

```bash
./scripts/linux/test.sh
```

结束必须有明确：

```text
ALL_TESTS_OK
```

至少覆盖：

```text
application header / protobuf test
world route test
correct login
bad token
enter world
repeat enter world same city
two players world query
cell conflict
out of world
```

真实 WebSocket E2E 不允许全部替换成直接 `skynet.call` 内部测试。

## 16. 第一课 Git 教学

延续旧项目方式，但不要过度占课程篇幅。

建议检查点：

```text
chore: initialize slg project
feat: add protocol and login path
feat: add player and storage
feat: add world routing and region workers
test: complete lesson one e2e
docs: complete lesson one practical guide
```

是否创建远程仓库由用户现场决定。

不要擅自假定 GitHub Repo URL。

## 17. 第一课完成后必须能解释

1. 为什么 Region 不等于 Service；
2. 为什么 WorldMgr 不保存所有 City；
3. 为什么 City 的位置不由 PlayerAgent 权威持有；
4. 两个玩家同时占同一格在哪里串行化；
5. `ensure_player_city` 为什么比 `create_city` 更适合登录流程；
6. 哪些 `skynet.call` 会 yield；
7. 为什么 Service 不是“永远不可重入的单线程对象”；
8. Memory Storage 和在线状态 Owner 的区别；
9. `query_rect` 为什么不是最终 AOI；
10. 如果 RegionWorker 出现 MQ 堆积，下一步应该观察什么，而不是先加锁。

## 18. 第一课明确不做

不要提前实现：

```text
Army
March
A*
AOI subscription
Battle
Territory
Alliance
Rally
MySQL
Redis
Cross Server
Season
```

可以在文档中说明后续怎么接，但不要为了显得商业级提前堆代码。

第一课的商业价值来自：

```text
边界正确
链路真实
测试真实
调试真实
后续可演进
```

不是模块数量。
