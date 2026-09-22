# Project Context

## 1. 为什么建立这个项目

学习者长期做传统 C++ + Lua MMO / ARPG 服务端，熟悉的核心模型偏向：

```text
Gate
GameServer
Player
Map
Monster
Skill
AOI
Chat
DB
Center
Cross Server
```

当前希望补齐 SLG 服务端，重点不是重新学习“游戏服务器是什么”，而是建立下面这些 SLG 特有能力：

```text
Persistent World
Logical Region
World Object Ownership
World AOI / Viewport
Army
March
Pathfinding
Scheduler
Territory
Alliance
Rally
Season
Hot Region
Cross Actor Consistency
Crash Recovery
```

项目同时作为 Skynet 实战工程。课程应把已有 MMO 经验映射到 Skynet，但不能把二者错误等同。

## 2. 学习者已有经验

可以默认已掌握或熟悉：

- C++ 服务端工程；
- Lua 业务；
- 单线程 GameServer；
- Gate / Game / Chat / DB / Center 等传统进程划分；
- 玩家在线状态；
- MySQL；
- 排行榜；
- 活动；
- 跨服；
- 九宫格 AOI；
- 基础寻路概念；
- 网络协议与异常包处理；
- 热更新；
- 线上日志和故障排查；
- Git 基础正在持续补齐。

不要把课程篇幅浪费在解释：

```text
TCP 是什么
HashMap 是什么
什么叫线程
Lua table 基础
MySQL CRUD 基础
```

## 3. 需要重点训练的差异

### 3.1 从角色实时模拟切换到持久世界

传统 MMO ARPG 常围绕：

```text
Player / Monster / Skill / Map / Update
```

SLG 更关心：

```text
World / Region / City / Army / Territory / Time
```

世界对象不会因为玩家离线而消失。

### 3.2 从高频 Tick 切换到事件和时间驱动

不能把 SLG 写成：

```cpp
for every player:
    update_building()
    update_resource()
    update_march()
```

建筑、研究、训练、行军等大量业务应使用：

```text
start_time
finish_time
event
scheduler
offline calculation
```

### 3.3 从单 GameServer 状态切换到 Actor Ownership

必须明确：

```text
谁拥有状态
谁可以写
跨 Owner 怎么传递
失败如何补偿
重复消息如何幂等
```

### 3.4 从角色 AOI 切换到 World Viewport

传奇九宫格 AOI 常由角色位置决定。

SLG 世界地图 AOI 更接近：

```text
Camera / Viewport
-> Region subscription
-> incremental add/remove
-> batch world event push
```

玩家可以查看与“自己角色位置”无关的远方世界区域。

## 4. 教学目标

课程结束后，学习者应能：

- 独立说明一套商业 SLG 服务端的主要 Service / State 边界；
- 设计 Region / Worker 路由；
- Review Skynet coroutine yield 风险；
- 实现和解释 March / Scheduler / Battle / Territory / Alliance 的核心链路；
- 设计 Restart Recovery；
- 判断哪些状态需要 Snapshot、Dirty Save、Immediate Save；
- 分析 Hot Region / Hot Alliance；
- 用 Bot、E2E、Benchmark、Message Queue Metrics、Trace、GDB 诊断问题；
- 面试或真实项目中能讨论不同方案，而不是只会本课程代码。

## 5. 课程不是产品开发

我们不会实现：

- 完整数百英雄；
- 完整技能编辑器；
- 完整策划配置平台；
- 完整 GM 后台；
- 完整支付；
- 完整运营活动；
- 完整社交系统；
- 完整生产部署平台。

但涉及这些系统时，应说明商业项目通常如何接入现有架构，以及当前课程为什么暂不实现。
