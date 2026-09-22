# Codex Start Here

你现在位于课程资料目录 `G:\simbi\dev\skynet-slg-learning`。实际手写工程固定放在 WSL 的 `~/workspace/skynet-slg-server`。

你的任务不是直接生成最终项目，而是接手教学和工程辅导。

## Step 1 - 先读资料

按顺序完整阅读：

```text
AGENTS.md
docs/PROJECT_CONTEXT.md
docs/COURSE_ROADMAP.md
docs/TARGET_ARCHITECTURE.md
docs/ENGINEERING_DECISIONS.md
docs/TEST_STRATEGY.md
codex/LESSON_01_SPEC.md
```

旧参考仓库：

```text
https://github.com/simbiwu/skynet-mmo-learning
```

重点参考：

```text
AGENTS.md
docs/Skynet第一课_启动链源码导读_实操.md
```

学习其“从实际执行过程出发、一步一步搭建、每阶段验证”的教学方式。

不要复制旧 MMO 业务代码。

## Step 2 - 检查当前现场

在修改文件前先检查：

```text
pwd
git status
git log
WSL / uname
gcc / make / git / gdb
VS Code Remote 环境（如可观察）
当前目录是否已经存在工程文件
是否已有 third_party
```

禁止默认：

```text
注销 Ubuntu
重装 WSL
删除用户已有文件
覆盖远程 Git
```

课程资料与实际工程分开。即使当前目录只有资料包，也在 WSL Linux Home 中创建 `~/workspace/skynet-slg-server`，不要在 `/mnt/g` 下编译 Skynet。

如果已有用户代码，先审计再决定如何合并。

## Step 3 - 使用并维护第一课实操文档

第一课实操文档已经生成：

```text
docs/Skynet_SLG第一课_从H5登录到进入世界_实操.md
```

不要在新会话中重新生成或另起一份。先结合 WSL 现场校验它，再按文档逐段辅导学习者亲手创建工程。现场路径、依赖版本或上游 API 确有变化时，直接修订这份文档，并同步 `ENGINEERING_DECISIONS.md` 中受影响的决定。

要求：

- 从当前现场开始，不假设固定盘符；
- 每条命令说明在哪个终端执行；
- 每个代码文件标完整仓库路径；
- 关键行为能从真实入口观察到结果；不要为每个小步骤另建 Smoke Test；
- 每个 `skynet.call` 说明调用方、接收方、yield 和恢复点；
- 解释 Process / Thread / Service / Lua State / coroutine；
- 解释 State Ownership；
- 给出真实 WebSocket E2E 和 Chrome 手工操作路径；
- 给出 Debug Console / GDB；
- 给出常见故障定位；
- 给出练习和最终检查项，不增加逐节验收关卡；
- 不用 AI 培训腔；
- 不用几十个空洞小标题；
- 不为了完整重复解释 TCP/Lua/C++ 基础。

修改文档后，先自行审计：

```text
是否真的可以从当前目录一步步执行？
有没有路径错误？
有没有未解释的第三方依赖？
有没有把 Region 写成 1 Service 1 Region？
有没有让 WorldMgr 变成全部世界请求的永久代理？
有没有把 Memory Table 当正式持久化？
有没有真实 WebSocket E2E？
```

## Step 4 - 按文档辅导实施

默认工作方式：

```text
文档一个阶段
-> 实施
-> 在当前行为可用时运行并观察
-> 解释现场
-> 下一阶段
```

如果用户让你直接实施某个阶段，可以直接实施。

但不要在第一条响应里把几十个文件全部生成完并只说“完成”。

课程的目标是学习运行过程和架构判断。
Login 业务之后立即接 H5 网络 Login；玩家真正能登录后才引入 EnterWorld 的坐标、Region 和 World。Git 提交由学习者按需要选择，不作为每阶段前置条件。

## Step 5 - 文档和代码同步

每次架构或行为发生变化：

```text
代码
测试
docs
```

一起改。

如果实际现场迫使你改变 `LESSON_01_SPEC.md` 中的重要设计，先在：

```text
docs/ENGINEERING_DECISIONS.md
```

追加新的 Decision 和理由。

不要静默漂移。

## Step 6 - 第一课验收

最终至少验证：

```text
[ ] Skynet v1.8.0 可脚本恢复和编译
[ ] Chrome H5 真实 WebSocket Login / EnterWorld / QueryWorld
[ ] Node E2E 连接两个真实 WebSocket 端口
[ ] Protobuf Envelope/Body 和 8-byte Header/自定义二进制 Body Unit Test
[ ] 错 Token AUTH_FAILED
[ ] PlayerAgent 动态创建
[ ] Memory Storage 经 Skynet Message 返回玩家 Snapshot，Agent 与 Worker 不共享 Lua Table 引用
[ ] EnterWorld 成功
[ ] 16 Logical Region
[ ] 4 RegionWorker Service
[ ] 同一玩家重复 EnterWorld 返回同一 City
[ ] 两玩家不能占同一 Grid
[ ] query_world 可以跨 Region 聚合结果
[ ] test.sh 输出 ALL_TESTS_OK
[ ] Debug Console 可观察 PlayerAgent / RegionWorker
[ ] Chrome DevTools 可观察 Binary Frame
[ ] LuaPanda 可命中 ConnectionWorker / PlayerAgent / RegionWorker
[ ] GDB 可说明 Runtime / Service Lua State
[ ] 学习者能说明 call 的 yield 和 State Owner
```

只有这些达到后，第一课才结束。

不要提前进入 Army / March。
