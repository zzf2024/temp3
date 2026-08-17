# DiCode Agent 本地运行证据提取说明

## 1. 适用版本

本文适用于未 rebrand、扩展 ID 为 `atad-apts.zgsm` 的详细证据版本：

```text
dicode-3.0.16-detailed-agent-evidence-no-redaction-no-review-skill-target-compatible.vsix
```

本版本记录原始任务内容和 Agent 运行证据，不做敏感信息脱敏。证据默认只保存在用户本地，不会自动上传。

## 2. 用户发现问题后如何提取

1. 不要立即删除当前任务。
2. 在 DiCode 聊天界面的任务操作区点击“标记问题并提取证据”（圆形感叹号图标）。
3. 扩展记录点击时的精确时间和当前任务 ID，并等待当前证据写入完成。
4. 扩展生成诊断 JSON，并自动在 VS Code 编辑器中打开。
5. 使用“另存为”保存该文件，或者从系统临时目录复制。

诊断文件名格式：

```text
costrict-diagnostics-<taskId前8位>-<毫秒时间戳>.json
```

Windows 临时目录通常为 `%TEMP%`。可以在文件资源管理器地址栏输入 `%TEMP%`，按修改时间倒序排列，查找最新的 `costrict-diagnostics-*.json`。

反馈问题时，优先提供该诊断 JSON，并说明问题发生时间、任务要求、预期行为和实际行为。

## 3. 原始证据保存在哪个目录

### Windows 桌面版 VS Code

默认存储根目录：

```text
%APPDATA%\Code\User\globalStorage\atad-apts.zgsm
```

本机用户 `hlb` 通常对应：

```text
C:\Users\hlb\AppData\Roaming\Code\User\globalStorage\atad-apts.zgsm
```

注意：目录是 `atad-apts.zgsm`，不是 `atad-apts.dicode`。

VS Code Insiders 通常使用：

```text
%APPDATA%\Code - Insiders\User\globalStorage\atad-apts.zgsm
```

### WSL Remote

```text
~/.vscode-server/data/User/globalStorage/atad-apts.zgsm
```

Insiders Remote 可能使用：

```text
~/.vscode-server-insiders/data/User/globalStorage/atad-apts.zgsm
```

### 配置了自定义存储目录

如果设置了 `costrict.customStoragePath`，证据保存在该自定义目录。任务目录仍然是：

```text
<customStoragePath>\tasks\<taskId>
```

自定义目录不可用时，扩展会回退到默认全局存储目录。

## 4. 每个任务的证据结构

所有任务位于：

```text
<实际存储根目录>\tasks\<taskId>
```

任务目录可能包含：

```text
tasks\<taskId>\
├── api_conversation_history.json
├── ui_messages.json
├── task_metadata.json
├── agent-actions\
│   ├── <instanceId-1>.jsonl
│   └── <instanceId-2>.jsonl
├── agent-runtime\
│   ├── <instanceId-1>.jsonl
│   └── <instanceId-2>.jsonl
├── agent-state\
│   ├── <instanceId-1>.json
│   └── <instanceId-2>.json
└── memory\
    └── working-state.json
```

同一个任务恢复或扩展重启后会创建新的 `instanceId` 文件，不覆盖之前实例的证据。

## 5. 各文件记录什么

### `api_conversation_history.json`

记录模型使用的对话历史，包括用户消息、Assistant 内容、工具调用以及回传给模型的工具结果。

### `ui_messages.json`

记录聊天界面展示的消息，用于判断用户实际看到了什么、何时出现完成结果或错误。

### `task_metadata.json`

记录任务基础元数据。并非所有历史任务或运行阶段都一定存在。

### `agent-actions/<instanceId>.jsonl`

这是 Agent 生命周期时间线。每个动作写入 JSONL 文件的一行，不是每个动作创建一个文件。事件包括：

```text
task_started
iteration_started
model_request_started
model_request_finished
tool_started
tool_finished
retry_scheduled
compaction_finished
task_finished
```

它记录动作顺序、时间、状态和耗时，但不保存完整工具参数和结果。

### `agent-runtime/<instanceId>.jsonl`

这是详细原始运行证据，每个事件占一行 JSON：

- `tool_invocation`：工具名、`toolCallId` 和完整工具参数；
- `tool_result`：相同 `toolCallId` 对应的完整工具结果；
- `user_response`：用户确认、拒绝或输入的文本，以及图片和附件数量。

`attempt_completion` 也是工具调用，因此 Agent 提交的完整完成结果会保存在 `tool_invocation.parameters`。用户随后是否确认会记录为 `user_response`。

### `agent-state/<instanceId>.json`

记录影子状态机的最新生命周期快照和发现的问题。`findings` 为空只表示没有发现生命周期矛盾，不代表任务一定真正完成。

### `memory/working-state.json`

记录当前任务的短期工作状态、步骤和召回的长期记忆引用，不是完整聊天记录。

## 6. 点击按钮生成的诊断 JSON

诊断 JSON 包含：

```text
schemaVersion
incident.recordedAt
incident.taskId
error.version
error.provider
error.details
history
evidenceFiles
```

`evidenceFiles` 会收集当前任务中存在的：

```text
api_conversation_history.json
ui_messages.json
task_metadata.json
memory/working-state.json
agent-actions/*
agent-runtime/*
agent-state/*
```

文件以原始字符串嵌入诊断 JSON。不存在的文件会跳过，不会阻止导出。

## 7. 没有点击按钮时如何手工提取

如果用户当时没有点击按钮，但记得问题发生的大概时间：

1. 打开实际存储根目录下的 `tasks`。
2. 按文件夹修改时间排序，检查问题时间附近更新的任务目录。
3. 查看候选目录的 `ui_messages.json`，确认是否为目标任务。
4. 检查 `agent-actions/*.jsonl` 和 `agent-runtime/*.jsonl` 的 `timestamp`。
5. 找到与问题时间相符的 `instanceId`。
6. 复制整个 `<taskId>` 目录，不要只复制单个 `agent-actions` 文件。

推荐交付顺序：

```text
首选：点击按钮生成的 costrict-diagnostics-*.json
备选：完整的 tasks\<taskId> 目录
```

## 8. 如何分析典型问题

对于“Agent 只读取说明文档，没有继续读取记忆 JSON 就宣布完成”：

1. 在 `agent-runtime` 找到说明文档对应的 `read_file` 调用和结果。
2. 确认结果中是否已经出现记忆 JSON 的路径。
3. 向后查找是否有以该路径为参数的 `tool_invocation`。
4. 如果没有，检查是否直接出现 `attempt_completion`。
5. 使用 `agent-actions` 确认工具执行和任务结束的顺序。
6. 使用 `ui_messages.json` 确认用户看到的完成声明。
7. 使用 `user_response` 判断用户是否确认、拒绝或补充要求。

分析重复循环时，可按 `toolName + parameters + result` 比较连续的 `agent-runtime` 事件，确认是否反复执行相同或等价操作并得到相同结果。

## 9. 重要边界

- 本版本没有脱敏，诊断文件可能包含完整对话、代码、命令、路径和工具结果。
- 当前不会自动上传证据，需要用户主动提供诊断文件或任务目录。
- 当前没有为详细 JSONL 设置独立的天数或容量上限；证据跟随任务目录保留，删除任务时会随任务数据一起删除。
- `agent-actions` 主要记录生命周期；完整工具参数和结果应查看 `agent-runtime`。
- ChatRAG 日志不能完全替代本地证据，因为它通常缺少本地 UI 状态、任务实例、用户按钮响应、工作记忆和影子状态。
