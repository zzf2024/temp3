# DiCode 本地记忆使用指南

DiCode 的短期记忆和已验证长期记忆随 VSIX 运行在 VS Code 扩展宿主中。用户不需要另外安装 PostgreSQL、对象存储、向量数据库或 ChatRAG。

## 记忆如何工作

- 短期记忆属于单个任务，保存任务目标、Todo 工作流状态和本次任务召回的长期记忆来源。
- 长期记忆跨任务保存，但只有用户明确确认的内容才能进入“已验证”状态。
- 新任务启动时，DiCode 只召回少量、范围匹配的已验证记忆。
- 每次模型请求前，DiCode 都会重新检查已召回记忆。记忆被撤销、删除或修改后，会在下一次模型请求前停止影响当前任务。
- 新增的长期记忆不会在任务运行中途自动加入；它从下一个新任务开始参与召回。

长期记忆不会替代当前任务要求。当前用户指令和更高优先级指令始终优先。

## 用户操作

打开命令面板（Windows/Linux：`Ctrl+Shift+P`），可以使用：

- `DiCode: Remember Verified Memory`：录入候选内容，选择类型、用户或项目范围，并在最终确认后保存。
- `DiCode: View Verified Memories`：查看当前有效的已验证长期记忆。
- `DiCode: View Current Task Recall`：查看哪些长期记忆影响了当前任务，以及它们的范围、版本和当前状态。
- `DiCode: Revoke Verified Memory`：选择一条记忆、填写原因并确认撤销。撤销记录保留在审计日志中，内容不会再被后续任务召回。

请不要把 API Key、访问令牌、刷新令牌、密码或其他秘密保存为记忆；DiCode 会拒绝明显的秘密内容，但这不能替代用户自己的数据安全判断。

## 数据保存在哪里

默认情况下，数据位于 VS Code 为扩展提供的 `globalStorageUri`。常见位置如下，实际路径会随 VS Code 渠道和远程运行方式变化：

- Windows 桌面版：`%APPDATA%\Code\User\globalStorage\atad-apts.dicode`
- Ubuntu/Linux 桌面版：`~/.config/Code/User/globalStorage/atad-apts.dicode`
- WSL Remote：`~/.vscode-server/data/User/globalStorage/atad-apts.dicode`

主要文件结构：

```text
<存储根目录>/
├── agent-memory/
│   ├── journal/memories.jsonl
│   └── snapshots/memory-index-v1.json
└── tasks/<taskId>/
    ├── memory/working-state.json
    └── agent-state/<instanceId>.json
```

`memories.jsonl` 是长期记忆的追加式事实日志；快照可以从日志恢复。`working-state.json` 是任务级短期记忆，不复制长期记忆正文，只记录必要的召回来源信息。

## 设置自定义存储路径

在命令面板运行 `Set Custom Storage Path`（中文界面显示“设置自定义存储路径”），输入一个绝对路径。也可以在设置 JSON 中配置：

```json
{
	"dicode.customStoragePath": "/absolute/path/to/dicode-storage"
}
```

Windows 示例：`D:\DiCodeStorage`。Ubuntu/WSL 示例：`/home/<user>/dicode-storage`。

留空会恢复默认 `globalStorageUri`。如果自定义目录无法创建或没有读、写、进入权限，DiCode 会提示错误并回退到默认目录。

设置新的路径不会自动搬迁旧数据。如果需要迁移，请先关闭所有使用 DiCode 的 VS Code 窗口，将整个旧存储根目录复制到新目录，再配置新路径。迁移前建议保留备份。

## 隐私、备份与限制

- 这些记忆保存在本机，不依赖后端同步，也不会因为“记住”操作自动发送给 ChatRAG。
- 被召回的记忆会成为模型请求的上下文，因此它的内容会随该次模型调用发送给当前配置的模型服务。
- 备份时应在 VS Code 关闭后复制整个存储根目录，以同时保留长期日志、快照和任务短期状态。
- 当前版本采用有界、确定性的本地检索，不提供向量语义搜索、多设备同步、团队共享或云端恢复。

## 影子状态机与工作流证据

影子状态机是随 VSIX 一起运行的本地诊断能力。每个任务创建时都会自动启用，不需要用户打开设置，也不依赖后端、ChatRAG 或网络。

它只观察 DiCode 已有的 Agent 工作流事件，包括：

- 任务启动、完成、失败或取消；
- 模型请求开始与结束；
- 工具调用开始与结束；
- Agent 迭代、重试和上下文压缩事件的顺序。

状态机维护 `idle`、`running`、`completed`、`failed` 和 `cancelled` 五种阶段，并检查以下异常：

- 事件发生在任务启动前或终止后；
- 任务重复启动，或不可恢复的终态后重新启动；
- 事件序号倒退、重复或出现缺口；
- 模型请求或工具调用缺少 ID、重复开始，或者没有开始就结束；
- 任务结束时仍有未关闭的模型请求或工具调用；
- 事件所属的任务或运行实例与当前状态机不一致。

这些发现仅用于诊断。影子状态机不会修改 Agent 的决策，不会阻止工具执行，也不会因为发现异常而自动重试、终止任务或宣布系统已经“进化”。证据写入失败时，普通任务仍会继续运行。

### 证据保存位置

证据与短期记忆使用同一个默认或自定义存储根目录：

```text
<存储根目录>/tasks/<taskId>/agent-state/<instanceId>.json
```

每个任务运行实例使用一个 JSON 文件。文件包含：

- `phase`：当前或最终任务阶段；
- `lastSequence`：最后观察到的事件序号；
- `openModelRequestIds` 和 `openToolCallIds`：尚未闭合的动作；
- `findingCount` 和 `findings`：发现的矛盾及其事件类型、序号和相关动作 ID；
- `updatedAt`、`taskId` 和 `instanceId`：证据更新时间及身份。

证据文件不保存用户提示词、模型回复、工具输入输出、API Key 或访问令牌。它会在任务启动、任务结束或发现新异常时原子更新；正常完成的任务通常是终态快照且 `findings` 为空。

### 用户如何查看

当前版本没有单独的“查看影子状态机发现”命令或告警面板。需要诊断时，可以关闭或暂停相关任务，然后直接打开上述 JSON 文件。若配置了自定义存储路径，就从该目录的 `tasks` 子目录查找；否则从 VS Code 的扩展 `globalStorageUri` 查找。

DiCode UI 中现有的 Agent 工作流显示功能可以展示工作步骤和动作事件，但它不是影子状态机诊断报告，也不应把“流程看起来正常”等同于 `findings` 为空。排查非法或矛盾状态时，以 `agent-state/<instanceId>.json` 为准。

实时向扩展外部发布 Agent 动作事件仍受调试开关控制；这不会关闭本地影子观察和证据落盘。普通用户不需要设置 `COSTRICT_AGENT_ACTIONS` 环境变量。

### 当前限制

- 影子状态机目前是观察和取证机制，不是执行门禁或自动修复器。
- 证据按本地任务实例保存，不会自动上传、聚合或同步到其他设备。
- 当前没有按用户展示异常趋势的 UI，也没有自动把诊断结果晋升为新策略。
- 单个任务在内存中最多保留 200 条发现，避免诊断数据无限增长。
