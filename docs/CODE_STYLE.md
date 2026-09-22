# Lua 服务端代码约定

这份约定用于课程示例和后续同一工程。读一个函数时，应从函数附近知道谁调用它、参数和返回值是什么、它修改谁的状态、哪里可能 yield；不能要求读者先追完整个文件才能猜出这些信息。

## 命名和数据形状

业务身份、生命周期和状态变量用表达含义的名字，例如 `connection`、`decoded_packet`、`player_id`、`response`、`worker_index`。`c`、`p`、`v`、`r` 不用来保存这类值；`i` 只用于紧邻的简单循环。多个不同用途的变量分行初始化，`if` 的分支和函数体单独换行；不要用一行代码同时表达判断、状态变化和退出。`string.unpack` 或 Service RPC 的多个返回值可以一次接收，但每个接收变量必须按返回顺序命名。通用框架回调要求的 `fd`、`session`、`source` 等惯用名可以保留，但首次出现时说明它们来自哪里。

跨模块或跨 Service 传递的 Table 在首次使用处列出字段、字段来源和可选项。例如 `wire.decode_request` 的结果包含 `version`、`flags`、`command`、`sequence`、`body`、`request`；Login 的 `request` 包含 `player_id`、`token`。返回 `{ code, message, player? }` 时，说明 `player` 只在成功时存在，`nil` 表示什么。后续新增字段时同步更新契约。Lua 没有静态字段提示不应成为省略契约的理由；需要时可增加 LuaLS 类型注解，但注解不能替代运行语义说明。

`pcall` 的两个返回值要区分“执行是否成功”和“函数返回值或错误”。用 `decode_ok, packet_or_error`、`call_ok, response_or_error` 之类的名字，并在调用处把成功结果收束为 `packet` 或 `response`。不要让业务代码长期传递含义不明的 `value`。

新增业务命令时，以 `protocol/commands.json` 和 `protocol/game.proto` 为协议定义，运行构建脚本生成 Lua/JS 命令资料。通用 Codec 按命令资料选择 Body 类型和字段；Gateway、ConnectionWorker、`wire.lua` 不加业务分支。只有增加通用字段类型、改变连接语义或传输协议时，才修改对应底层模块。业务 Handler 的分发仍在状态 Owner 内完成。

## 函数契约

关键函数首次出现时，在相邻注释或文字中说明：调用方；每个非显然参数的来源与形状；成功、失败和 `nil` 的返回语义；修改的 Owner 状态；可能 yield 的调用，以及恢复后需要重新检查什么。回调尤其要说明是谁调用，例如 `handle.message` 由 `http.websocket` 在读完一条消息后调用，`CMD.accept` 由 Gateway 通过 Skynet 消息触发。`skynet.call` 的返回值按目标 Service 的契约解释，不能只看本函数的接收变量名。

示例允许先以一段文字定义共用 Table，再让相关函数引用该契约；不必在每一行重复注释。注释应回答“为什么这样处理”和失败后的去向，名称本身能说明的赋值不需要复述。改动函数签名、Table 字段或错误码时，同步修改调用方、文档和必要测试。
