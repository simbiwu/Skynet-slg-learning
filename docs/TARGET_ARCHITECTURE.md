# Target Architecture

这份文档描述课程最终方向。第一课只实现其中最小一部分。

## 1. 总体图

```text
                         +------------------+
                         | H5 / Node Clients|
                         |  Manual / Bots   |
                         +---------+--------+
                                   |
                 WebSocket 双端口：Protobuf / 自定义二进制
                                   |
                         +---------v---------+
                         | WebSocketGateway  |
                         +---------+---------+
                                   |
                         +---------v---------+
                         | ConnectionWorkers |
                         +---------+---------+
                                   |
                         +---------v--------+
                         |    PlayerMgr     |
                         +---------+--------+
                                   |
                     +-------------+-------------+
                     |                           |
             +-------v-------+           +-------v-------+
             |  PlayerAgent  |           |   Storage     |
             +-------+-------+           +---------------+
                     |
         +-----------+--------------------+
         |                                |
 +-------v-------+                  +-----v------+
 |   WorldMgr    |                  | Alliance   |
 | routing/meta  |                  | Service(s) |
 +-------+-------+                  +------------+
         |
         | direct route after discovery
         v
 +------------------------+
 |   RegionWorker Pool    |
 |------------------------|
 | Region 0,4,8...        |
 | Region 1,5,9...        |
 | ...                    |
 +----+---------+---------+
      |         |
      |         +----------------+
      |                          |
 +----v-----+              +-----v------+
 | World AOI|              | Army/March |
 +----------+              +-----+------+
                                  |
                            +-----v------+
                            | Scheduler  |
                            +-----+------+
                                  |
                            +-----v------+
                            | BattlePool |
                            +------------+
```

课程后期加入：

```text
MySQL
Cross Server
Season Service / Season World
Metrics / Trace / Profiling
```

## 2. 为什么第一课就使用 RegionWorker Pool

教学上最简单的是：

```text
1 Region = 1 Service
```

但商业项目如果有数百、数千逻辑 Region，这会带来大量 Lua State、Service 生命周期和消息路由成本。

因此本项目从开始就区分：

```text
Logical Partition
Execution Partition
```

例如：

```text
WORLD = 256 x 256
REGION = 64 x 64
Region Count = 16
Worker Count = 4
```

路由：

```text
logical_worker_id = region_id % worker_count
lua_array_index = logical_worker_id + 1
```

一个 Worker 内：

```lua
regions = {
    [0] = ...,
    [4] = ...,
    [8] = ...,
    [12] = ...,
}
```

课程规模很小，但这个边界可以自然扩展。

## 3. 第一课状态边界

### PlayerAgent

持有：

```text
player_id
player persistent snapshot
resource / hero 等第一课最小玩家数据
city_id reference
connection binding metadata（视具体实现）
```

不持有 World Grid 权威状态。

PlayerAgent 不持有 Socket fd，也不解析 Protobuf。它保存当前逻辑连接的 ConnectionWorker Address 和 Connection ID，用于拒绝旧连接消息。

### WebSocketGateway / ConnectionWorker

WebSocketGateway 持有两个 Listen fd、Worker Address 和 Round-robin 下标，不参与每包转发。`:8890` 接收 Protobuf Envelope + Body，`:8891` 接收 8-byte Header + 自定义二进制 Body；监听端口决定 Codec。

ConnectionWorker 持有：

```text
fd -> Connection
connection generation/state
PlayerAgent Address
按端口选择的 Wire Codec
```

一次业务 `skynet.call` 恢复后必须确认连接仍然是相同 generation。网络和 Codec 状态止于接入层，业务 Service 只接收普通 Lua Table。

### WorldMgr

持有：

```text
world config
region layout
region -> worker routing
worker lifecycle / discovery
```

不保存：

```text
每个 City
每支 Army
每块 Territory
```

### RegionWorker

持有其负责 Region 内：

```text
world object
cell occupation
spatial index
city world marker
```

第一课 World Object 只需要 City。

### Storage

第一课 Memory Worker 是持久化替身。

边界要保留，以便第二课实现 March 等长时间世界行为时接入 MySQL 和重启恢复。

## 4. `ensure_player_city` 为什么必须幂等

第一次进入世界需要给玩家创建主城。

错误做法：

```text
PlayerAgent:
  检查没有 city_id
  -> create city
  -> world 成功
  -> PlayerAgent 写 city_id
```

如果 World 已成功，但 PlayerAgent 在保存 city_id 前 Crash，重试可能再创建一个 City。

第一课使用：

```text
ensure_player_city(player_id, ...)
```

World 侧按玩家唯一键检查：

```text
已经有城 -> 返回已有 City
没有 -> 创建
```

第一课先计算稳定 `home_region_id`，将同一玩家始终路由到同一 Region Owner；Region 内的 `city_by_player_id` 先查后建。这样 Client 重试、连接重试或 coroutine 异常后再次调用，都不会生成重复城市。

课程后期还会进一步处理：

```text
version
generation
persistent recovery
```

## 5. 世界对象

第一课最小 WorldObject：

```text
id
type
owner_player_id
x
y
version
```

后面逐步加入：

```text
Resource
Army
Fort
AllianceBuilding
Territory
```

不要建立一个包含所有玩法字段的巨大通用 table。

空间层只保存空间层需要的状态。

## 6. AOI 演进

第一课：

```text
query_rect
```

只用于验证路由和世界状态。

第二课升级为：

```text
open_view
-> subscribe regions
-> initial snapshot
-> world event
-> batch diff push
-> move_view add/remove subscriptions
```

第一课文档必须明确 `query_rect` 不是最终商业 AOI 方案。

## 7. March 演进

第二课的 March 不使用：

```text
每 50ms 更新所有 Army 坐标
```

核心记录：

```text
path
start_time
arrive_time
speed/version
state
```

需要显示当前位置时，根据时间和路径推导。

只有发生：

```text
arrive
recall
speed change
intercept
battle
```

时改变状态。

## 8. Scheduler

大量长时间事件不应该每对象轮询。

课程会实现一个可恢复 Scheduler，并讨论：

```text
Skynet timeout
Min-Heap
Timing Wheel
分桶
```

的适用条件。

最终目标不是宣称某一种算法永远最好，而是能根据 Timer 数量、精度、修改频率、恢复需求做选择。

玩家下线时，PlayerAgent 可以退出；City、Army 和 March 不能随之删除。玩家私有且没有外部即时影响的任务可以持久化 `finish_at` 并在下次访问时补算。会影响其他玩家的 March、Rally 和 Territory 事件由长期存在的 World/Region Owner 与 Scheduler 共同处理。

Skynet Runtime Timer 不提供业务持久化。进程重启后要从 Storage 加载 `task_id/version/finish_at/state` 并重建提醒；旧 Timer 到达时必须用 Version 拒绝。

## 9. Battle

Battle 应支持：

```text
same input
+ same seed
+ same battle version
= same result
```

便于：

```text
battle report
replay
GM reproduce
client presentation
regression
```

## 10. Persistence

不同状态不能用一个统一保存策略粗暴处理。

第二课接入持久化时就区分：

```text
Player:
  snapshot / dirty save

World:
  region dirty state / snapshot

March:
  active state + scheduler recovery

Alliance:
  critical mutation + normal dirty state

Battle:
  result/report, not every internal frame
```

第二课先落实 Player、World 和 March 的保存、重启恢复与重复触发语义；第三课按 Alliance、Territory 和 Battle 的实际写入补充策略。一次跨 Owner 操作不能因为用了 MySQL 就被当成原子事务，必须说明权威串行点、提交顺序、失败后的重试或补偿。

## 11. Commercial scaling questions

课程实现每个模块时，必须同步讨论：

- 1 万在线和 10 万注册有什么差别；
- 100 万 Grid 是否真的都需要 table；
- 一个 Region 5000 Army 时怎么办；
- 一个 Alliance 200 人同时操作时怎么办；
- 一个 AOI 事件向 1000 观察者广播怎么办；
- 一个 Worker 的 MQ 长度持续增长说明什么；
- 什么时候需要拆 Worker / 进程 / 节点；
- 什么时候 C/C++ Native 优化才有收益。

## 12. 多节点与赛季

第四课将已有 Gateway、Player 和 World Service 部署到多个 Skynet 进程或节点。WorldMgr 保留 Region 路由元数据，RegionWorker 保留空间权威；节点之间通过明确的地址和版本路由请求。节点重启或迁移时，要先确认旧 Owner 不再有写入权，推进可持久化的分配 epoch，再从持久化状态恢复新 Owner；写入和延迟消息都需校验 epoch，避免两个节点同时宣称权威。课程必须演练旧节点尚未退出、新节点已接管的情况。`skynet.cluster` 提供跨节点调用机制，业务仍需处理超时、断连、重复请求与移交过程中的不确定结果。

赛季结算要区分玩家长期资料、赛季世界状态、战报与归档数据的生命周期。结算、重置或迁移以可恢复的阶段状态推进；进程在任一阶段退出后能继续或安全重试。具体保留和清空哪些字段随本课程的实际业务确定，不能靠一次全库清表完成赛季切换。
