# 实践 05 · Pi Agent 源码泛读与功能定位

- 姓名：罗奕峰
- GitHub：`nexus610`
- 实验环境：Windows，Node.js 24.22.3，Pi 1.5.2，`deepseek/deepseek-v4-pro`
- 小组源码基础提交：`c3cc91632d0a1950fb122605b190233f2aa0006e`，本地编译产物保留实践 04 的两处 `[P4]` 日志。[小组源码仓库](https://github.com/sys-analysis-design-team/pi-hunnu-2026)

## A. 单轮交互与工具调用

在独立的实验目录创建 `handoff.txt`，写入待办「补充 README 的启动说明」和启动命令 `npm run demo -- --profile lyf-14989`。从实验目录用 Node.js 启动本地编译的 `packages/coding-agent/dist/cli.js`，使用独立的 `--session-dir` 保存对话。我在 RPC 模式提交提示词并记录工具事件；图 1 同时标出交互界面里对应的输入与渲染位置。

实际输入为「请用 read 工具读取当前目录的 handoff.txt，原样告诉我待办和启动命令。」输入只给了文件名。Pi 成功调用了一次 `read`，回答中包含待办和完整的启动命令。一次用户输入触发了两轮模型请求。

```text
[P4] Agent 即将请求模型
[P4] 工具结果已加入上下文
[P4] Agent 即将请求模型
```

### 图 1 · 读取交接文件的代码级时序

```mermaid
sequenceDiagram
    participant U as 用户
    participant I as 交互与会话层
    participant A as Agent 执行层
    participant M as 模型请求层
    participant R as read 工具
    U->>I: 提交读取 handoff.txt 的指令
    Note over I: TUI 入口 defaultEditor.onSubmit → session.prompt(text)<br/>本次取证入口为 rpc-mode 的 prompt 命令
    I->>A: agent-session.ts → this.agent.prompt(messages)
    A->>A: agent.ts → runAgentLoop → agent-loop.ts 的 runLoop
    A->>M: streamAssistantResponse → await streamFunction(model, llmContext, ...)
    Note over M: modelRuntime.streamSimple 选择 openai-completions 适配器<br/>client.chat.completions.create 向 DeepSeek 请求并接收流式事件
    M-->>A: 助手消息包含 read 工具调用
    A->>R: executePreparedToolCall → prepared.tool.execute(...)
    R->>R: resolveReadPathAsync → access → createReadStream
    Note over R: readLocalFileChunks 通过 fsOpen 和 FileHandle.createReadStream 读取本地文件
    R-->>A: 返回文件内容，生成 toolResult 消息
    A->>A: currentContext.messages.push(result)
    A->>M: 再次调用 streamFunction，此次上下文含 toolResult
    M-->>A: 模型基于文件内容生成回复
    A-->>I: message_update / message_end
    I-->>U: handleEvent → streamingComponent.updateContent 显示回复
```

原始工具调用的 `offset` 和 `limit` 都是 `null`。用同一构建中的 `validateToolArguments` 和 `read` 的真实参数定义做本地验证后，传给工具的参数只剩文件路径。因此 `isStreamableRange(undefined, undefined)` 成立，文本读取走 `selectLinesFromStream` 和 `ops.createReadStream`。实际读取发生在本地，模型从第二轮上下文中的工具结果获得文件内容。[参数校验](https://github.com/sys-analysis-design-team/pi-hunnu-2026/blob/c3cc91632d0a1950fb122605b190233f2aa0006e/packages/ai/src/utils/validation.ts#L46) · [read 的分支](https://github.com/sys-analysis-design-team/pi-hunnu-2026/blob/c3cc91632d0a1950fb122605b190233f2aa0006e/packages/coding-agent/src/core/tools/read.ts#L296)

| 定位点 | 对应源码 |
| --- | --- |
| 输入与输出 | [`interactive-mode.ts` 的 `onSubmit`](https://github.com/sys-analysis-design-team/pi-hunnu-2026/blob/c3cc91632d0a1950fb122605b190233f2aa0006e/packages/coding-agent/src/modes/interactive/interactive-mode.ts#L3325) · [`subscribeToAgent`](https://github.com/sys-analysis-design-team/pi-hunnu-2026/blob/c3cc91632d0a1950fb122605b190233f2aa0006e/packages/coding-agent/src/modes/interactive/interactive-mode.ts#L3525) · [`message_update`](https://github.com/sys-analysis-design-team/pi-hunnu-2026/blob/c3cc91632d0a1950fb122605b190233f2aa0006e/packages/coding-agent/src/modes/interactive/interactive-mode.ts#L3649) |
| 执行循环 | [`Agent.prompt`](https://github.com/sys-analysis-design-team/pi-hunnu-2026/blob/c3cc91632d0a1950fb122605b190233f2aa0006e/packages/agent/src/agent.ts#L380) · [工具结果写入上下文](https://github.com/sys-analysis-design-team/pi-hunnu-2026/blob/c3cc91632d0a1950fb122605b190233f2aa0006e/packages/agent/src/agent-loop.ts#L283) |
| 模型与文件边界 | [`sdk.ts` 的模型运行时](https://github.com/sys-analysis-design-team/pi-hunnu-2026/blob/c3cc91632d0a1950fb122605b190233f2aa0006e/packages/coding-agent/src/core/sdk.ts#L422) · [模型请求适配器](https://github.com/sys-analysis-design-team/pi-hunnu-2026/blob/c3cc91632d0a1950fb122605b190233f2aa0006e/packages/ai/src/api/openai-completions.ts#L385) · [本地文件流](https://github.com/sys-analysis-design-team/pi-hunnu-2026/blob/c3cc91632d0a1950fb122605b190233f2aa0006e/packages/coding-agent/src/core/tools/read.ts#L76) |

## B. 恢复会话与新建会话

A 完成后退出 Pi，再在同一实验目录用同一个本地 CLI、同一个模型和会话目录，带 `-c` 重新启动。用 `get_messages` 确认历史中仍有 A 的用户问题、工具调用、`toolResult` 和最终答复，且保留了完整命令。随后通过 RPC `new_session` 触发新会话，该命令与交互界面的 `/new` 都调用 `runtimeHost.newSession()`。新会话在提问前的聊天消息数为 0。

两次提交了完全相同的问题：

> 接着上次的待办，给 README 起草一条启动说明，写出交接单里的完整启动命令。只用这次会话已有的信息。不要调用工具。如果缺少命令，请直接说明。

| 观察项 | 加 `-c` 恢复旧会话 | `new_session` 新会话 |
| --- | --- | --- |
| 提问前聊天历史 | 保留 A 的问题、工具调用、工具结果和回答 | 0 条聊天消息 |
| 工具调用 | 0 次 | 0 次 |
| 模型请求 | 1 次 | 1 次 |
| 实际回答节选 | 在 README 启动草稿中写出 `npm run demo -- --profile lyf-14989` | 「根据本次会话已有的信息，我无法写出交接单里完整的启动命令。」 |

### 图 2 · 恢复和新建会话时发给模型的聊天消息

```mermaid
flowchart TD
    A["A 结束：Pi 已读文件并回答启动命令"]
    A --> C["退出 Pi，带 -c 重启"]
    A --> N["交互 /new 或 RPC new_session"]
    C --> F["continueRecent 选择旧会话文件<br/>loadEntriesFromFile 加载 JSONL"]
    F --> H["buildSessionContext 恢复 A 的聊天消息"]
    H --> Q1["输入 README 启动说明的问题"]
    Q1 --> R1["模型上下文包含 A 的工具结果和回复<br/>再加本次用户问题"]
    N --> E["runtimeHost.newSession 创建空聊天历史"]
    E --> Q2["输入完全相同的问题"]
    Q2 --> R2["模型收到本次用户问题<br/>聊天历史中没有 A 的命令"]
```

两条路径都有正常的系统提示。差别在于是否把 A 的历史消息加回模型上下文。恢复路径从 CLI 的 `createSessionManager` 进入 `SessionManager.continueRecent`，再由 SDK 将 `existingSession.messages` 初始化到 Agent。新建路径创建新的 `SessionManager` 并替换当前会话运行时。[CLI 的恢复入口](https://github.com/sys-analysis-design-team/pi-hunnu-2026/blob/c3cc91632d0a1950fb122605b190233f2aa0006e/packages/coding-agent/src/main.ts#L480) · [历史恢复](https://github.com/sys-analysis-design-team/pi-hunnu-2026/blob/c3cc91632d0a1950fb122605b190233f2aa0006e/packages/coding-agent/src/core/sdk.ts#L409) · [交互 `/new`](https://github.com/sys-analysis-design-team/pi-hunnu-2026/blob/c3cc91632d0a1950fb122605b190233f2aa0006e/packages/coding-agent/src/modes/interactive/interactive-mode.ts#L3429) · [RPC 新建会话](https://github.com/sys-analysis-design-team/pi-hunnu-2026/blob/c3cc91632d0a1950fb122605b190233f2aa0006e/packages/coding-agent/src/modes/rpc/rpc-mode.ts#L456)

要改输入处理或回复样式，可以从交互层入手。要改 `read` 的读取规则，可以查看 `core/tools/read.ts`。要改恢复策略，应在会话管理层检查文件选择和历史重建，再验证 `-c` 与新会话两条路径。
