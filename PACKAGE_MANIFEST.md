# Package Manifest

```text
README.md
AGENTS.md
PACKAGE_MANIFEST.md

docs/
  PROJECT_CONTEXT.md
  COURSE_ROADMAP.md
  TARGET_ARCHITECTURE.md
  ENGINEERING_DECISIONS.md
  TEST_STRATEGY.md
  Skynet_SLG第一课_从H5登录到进入世界_实操.md

codex/
  LESSON_01_SPEC.md
  CODEX_START_HERE.md
```

## 使用方式

1. 保留本目录作为课程资料目录；实际代码工程按实操文档建在 WSL 的 `~/workspace/skynet-slg-server`。
2. 用 Codex 打开本课程资料目录。
3. 第一条指令建议直接说：

```text
先完整阅读根目录 AGENTS.md、docs/ 和 codex/ 中的资料。
按照 codex/CODEX_START_HERE.md 接手这个项目。
先检查当前环境和目录现场，然后按现有第一课实操教程逐段辅导我亲手完成。
不要一次性把整套项目全部写完。
```

4. 后续按第一课实操教程逐步完成代码、测试、调试和 Git Checkpoint；不要重新生成另一份第一课文档。

## 课程数量

主课程固定 4 课，同一工程持续演进：

```text
Lesson 1  Login / 双端口接入 / EnterWorld / QueryWorld
Lesson 2  可恢复的世界行为：AOI / Army / March / 持久化 / Scheduler
Lesson 3  多人冲突：Battle / Territory / Alliance / Rally
Lesson 4  多节点 / Season / 性能 / 可观测性 / 故障演练
```
