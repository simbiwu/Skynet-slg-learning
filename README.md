# Skynet SLG Server Learning

这个目录保存 SLG 课程资料和实操手册。实际手写的 Server、H5 Client、测试和 Git History 放在 WSL Linux 文件系统中的独立工程：

```text
课程资料目录（Windows）：
G:\simbi\dev\skynet-slg-learning

实际实操工程（WSL Ubuntu）：
~/workspace/skynet-slg-server
```

两者分开是为了延续现有 MMO 课程的学习方式，也避免在 `/mnt/g` 下编译 Skynet。课程文档给出完整文件和代码，学习者在 WSL 工程中逐步创建、运行、断点和提交。

## 接手顺序

新 Codex 会话先完整阅读：

1. `AGENTS.md`
2. `docs/PROJECT_CONTEXT.md`
3. `docs/COURSE_ROADMAP.md`
4. `docs/TARGET_ARCHITECTURE.md`
5. `docs/ENGINEERING_DECISIONS.md`
6. `docs/TEST_STRATEGY.md`
7. `codex/LESSON_01_SPEC.md`
8. `codex/CODEX_START_HERE.md`
9. `docs/Skynet_SLG第一课_从H5登录到进入世界_实操.md`

随后先检查 WSL、工具链和目录现场。不要注销或重装已有 Ubuntu，不要自动生成整套最终工程。默认工作方式是：学习者按实操文档写一个阶段，运行并理解以后再进入下一阶段。

## 固定技术基线

- Skynet：v1.8.0
- Lua：Skynet v1.8.0 bundled modified Lua 5.4.7
- Server Runtime：Linux；Windows 开发机使用 WSL2
- Client：原生 HTML + JavaScript H5
- Transport：WebSocket Binary；本地 `ws://`，生产 `wss://`
- Application Protocol：双端口；`:8890` 为 Protobuf Envelope + Protobuf Body，`:8891` 为 8-byte Custom Header + 自定义二进制 Body
- Header：`version:uint8 + flags:uint8 + command:uint16 BE + sequence:uint32 BE`
- Server Protobuf：lua-protobuf 0.5.3，固定 Commit `ee4beb3865e2b82ea94b8a4314d78875c550ce20`
- Browser Protobuf：protobufjs 7.5.4
- Node E2E：ws 8.18.3
- 第一课存储：Memory Storage，但保留可替换的 Storage 边界
- 开发调试：Chrome DevTools、LuaPanda、Skynet Debug Console、GDB
- 世界分片：Logical Region + 固定数量 RegionWorker

## 课程目标

主课程固定四课，所有代码在同一个 `~/workspace/skynet-slg-server` 工程中持续演进。课程覆盖登录与双协议入口、可恢复的世界行为、多人冲突与联盟、多节点和赛季运行，并以状态归属、并发、恢复、压测和故障排查为求职准备的能力目标。具体顺序见 [四课路线](docs/COURSE_ROADMAP.md)。最终链路包括：

```text
H5 / Node E2E
  -> WebSocketGateway / ConnectionWorker
  -> Login / PlayerAgent
  -> WorldMgr / RegionWorker
  -> Viewport / Army / March
  -> Scheduler / Battle
  -> Territory / Alliance / Rally
  -> Persistence / Recovery
  -> Cross Server / Season
```

完成课程后，需要能解释并验证：

- 每份状态的唯一 Owner、写入者和恢复来源；
- `skynet.call` 的 yield、恢复以及 Stale State 风险；
- Region 与 RegionWorker 的关系；
- 玩家下线后 City、Army、March 为什么仍然存在；
- 长时间任务如何使用 `finish_at + version`，以及进程重启后如何恢复；
- Hot Region、Hot Alliance、AOI Fanout 和 Message Queue 积压如何诊断；
- 两个 H5 端口如何经各自 Codec 汇合到同一个业务 Handler，并按原协议返回 Response。

## 参考项目

旧 MMO 学习项目：

https://github.com/simbiwu/skynet-mmo-learning

新项目继承它的环境、调试、Git 和教学方法，不复制 Sproto、TCP Gate、Scene、Monster 和 ARPG Combat 业务代码。
