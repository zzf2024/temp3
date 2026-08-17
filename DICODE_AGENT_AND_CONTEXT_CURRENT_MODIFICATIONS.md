# DiCode Agent 与上下文改造现状

## 1. 文档范围

本文说明当前工作区和最新未 rebrand VSIX 中，围绕 DiCode Agent、上下文管理、记忆和运行证据已经完成的改造，并区分尚未实施的设计。

当前详细证据版发布物：

```text
dicode-3.0.16-detailed-agent-evidence-no-redaction-no-review-skill-target-compatible.vsix
扩展 ID：atad-apts.zgsm
命令空间：costrict.*
```

该包没有执行 rebrand，可覆盖安装现有 `atad-apts.zgsm`，默认数据目录仍属于 `atad-apts.zgsm`。

## 2. Agent 运行机制方面的改造

### 2.1 结构化 AgentAction 生命周期

Agent 运行过程中会产生结构化动作事件，覆盖：

```text
任务开始和结束
Agent 迭代开始
模型请求开始和结束
工具调用开始和结束
重试调度
上下文压缩结束
```

事件带有 `taskId`、`instanceId`、`runId`、事件 ID、单调序号、时间、状态、耗时和结果分类，可用于还原一次任务的执行顺序。

同一任务恢复或扩展重启后使用新的 `instanceId`，避免把不同运行实例混成一条时间线。

### 2.2 影子 Agent 状态机

每个任务都会创建只观察、不干预执行的影子状态机，维护：

```text
idle
running
completed
failed
cancelled
```

它检查事件序号倒退或跳跃、任务开始重复、终态后继续执行、模型或工具结束事件缺少开始事件、任务结束时仍有未关闭动作等生命周期矛盾。

证据保存在：

```text
<存储根目录>/tasks/<taskId>/agent-state/<instanceId>.json
```

`findings` 为空只代表没有发现这些生命周期矛盾，不代表 Agent 已经真正满足用户目标。状态机不会阻止工具、自动重试或修改 Agent 决策。

### 2.3 本地 Agent 生命周期时间线

所有 AgentAction 事件现在都会追加保存到：

```text
<存储根目录>/tasks/<taskId>/agent-actions/<instanceId>.jsonl
```

每个动作占 JSONL 的一行，而不是每个动作创建一个 JSON 文件。写入按顺序串行执行，单次失败不会让普通任务失败。

### 2.4 本地详细运行证据

新增独立的详细时间线：

```text
<存储根目录>/tasks/<taskId>/agent-runtime/<instanceId>.jsonl
```

它记录未经脱敏的原始内容：

- `tool_invocation`：工具名、`toolCallId`、完整参数；
- `tool_result`：相同 `toolCallId` 对应的完整结果；
- `user_response`：用户确认、拒绝或输入的文本，以及图片和附件数量。

`attempt_completion` 也按工具调用记录，因此可以看到 Agent 提交了什么完成声明，以及用户随后是否确认。

这层证据与 ChatRAG 的区别是：ChatRAG 主要反映模型请求和响应，本地证据还能关联任务实例、工具生命周期、本地 UI 响应、工作记忆和影子状态。

### 2.5 用户主动标记问题和导出证据

聊天任务操作区新增“标记问题并提取证据”按钮。用户点击后：

1. 记录精确事故时间和当前任务 ID；
2. 等待 Agent 证据写入完成；
3. 生成 `costrict-diagnostics-*.json`；
4. 在 VS Code 中打开该文件。

诊断文件收集当前任务中存在的：

```text
api_conversation_history.json
ui_messages.json
task_metadata.json
memory/working-state.json
agent-actions/*
agent-runtime/*
agent-state/*
```

证据默认保存在本地，不自动上传。当前版本按照用户要求没有敏感信息脱敏。

## 3. Agent 记忆方面的改造

### 3.1 单任务短期工作记忆

每个新建或恢复的任务可以维护：

```text
<存储根目录>/tasks/<taskId>/memory/working-state.json
```

内容包括任务目标、Todo 步骤、当前步骤和本次任务召回的长期记忆来源。Todo 更新时会同步工作状态；任务恢复时可用短期状态恢复步骤。

短期记忆仍以任务状态为主，不是独立于聊天历史的完整认知记忆。历史任务不会自动批量回填该目录。

### 3.2 用户确认的长期记忆

长期记忆采用“只有用户明确确认才能写入”的方式，提供命令：

```text
DiCode: Remember Verified Memory
DiCode: View Verified Memories
DiCode: View Current Task Recall
DiCode: Revoke Verified Memory
```

长期记忆使用追加日志和索引快照，支持作用域、来源、版本和撤销。首次没有写入已验证记忆时，长期记忆文件可以尚未创建，这是懒创建行为。

新任务启动时会进行有界召回，并把召回来源记录到 `working-state.json`。模型请求前会重新验证已召回记忆；如果记忆已被撤销或版本失效，会停止继续注入。

### 3.3 当前记忆边界

目前没有实现：

- 自动把每次聊天总结成长期记忆；
- 对旧任务批量生成短期记忆；
- 向量数据库语义检索；
- 多设备同步；
- Agent 自动审核并激活自己生成的长期记忆。

## 4. 上下文管理方面的改造

### 4.1 V2 上下文压缩架构

上下文压缩 V2 已完成到 Milestone 9，并经过手动和自动 UAT。它没有简单覆盖原始对话，而是区分：

```text
Canonical History：持久化的权威历史
Effective Model Context：发送给模型的上下文投影
UI History：用户界面展示的消息
```

压缩只改变模型上下文投影，不把压缩结果当成新的权威聊天历史，从而保留恢复、审计和回滚能力。

### 4.2 统一 Token 计量和预算判断

V2 在压缩决策前建立不可变的使用量快照，统一计算系统提示、对话、工具结果、图片和输出预留，减少不同代码路径对“是否超限”的判断不一致。

自动压缩使用统一全局阈值。旧的 Profile 阈值字段仍保留用于持久化和降级兼容，但当前运行时不再按 Profile 覆盖全局值。

### 4.3 分阶段压缩流水线

当前流水线按以下顺序处理：

```text
无损清理和裁剪
重新计量
结构化语义摘要
重新构建模型上下文
必要时进入安全回退
```

压缩保留最近的完整轮次，把较早历史转换为结构化摘要，尽量保留任务目标、用户约束、文件、决策、未完成工作、工具状态和继续执行所需信息。

### 4.4 工具协议完整性保护

压缩不能为了节省 Token 破坏工具调用协议。V2 会成组处理工具调用与结果，并处理：

- 缺失或孤立的工具结果；
- 延迟到达的结果；
- 重复结果；
- 压缩边界跨越工具调用的问题；
- Provider 发送前的有效性检查和修复。

目标是保证发送给模型的 `tool_use` 与 `tool_result` 关系仍然合法。

### 4.5 失败回退、熔断和无进展保护

当摘要失败、压缩没有释放足够 Token 或反复产生相同结果时，上下文压缩模块具有有界尝试、无进展检测、冷却和安全回退逻辑，可选择 Legacy、截断或停止，而不是无限重复压缩。

这些保护目前只属于上下文压缩流程，尚未推广为通用 Agent 工具循环恢复机制。

### 4.6 V2 发布控制与回滚

上下文压缩具有 `legacy`、`shadow`、`v2` 路由和确定性分桶、用户覆盖、门槛指标、生命周期遥测及 UAT 报告能力。

Legacy 仍是仓库范围的运行时回滚和兼容依赖。当前没有删除 Legacy、迁移旧持久化记录或清理旧 parent-ID 语义；这些工作必须作为单独授权的 Milestone 10 执行。

## 5. 输入和任务状态相关改造

当前任务包含输入准入协调器和 rollout 路由，用于规范新任务、恢复任务和后续用户输入的提交路径，避免不同入口绕过相同的准备与校验流程。

Todo 状态可以持久化并恢复，也可以用于可选的完成门槛。但是 Todo 由模型更新，不能单独作为任务真实完成的证据。

## 6. 当前仍未完成的 Agent 行为改造

以下内容已经审查和设计，但尚未改变真实 Agent 行为：

### 6.1 尚无 AcceptanceContract

任务开始时还没有自动冻结“必须满足哪些条件才算完成”的结构化验收合同。

### 6.2 尚无 EvidenceLedger 和 CompletionVerifier

当前 `attempt_completion` 主要检查：

- 当前轮工具是否失败；
- 结果是否为空；
- 可选的未完成 Todo；
- 受管终端是否仍忙碌。

它还不会根据用户目标检查必要证据。例如用户要求读取说明文档后继续检查某个 JSON，当前运行时不会因为缺少第二次读取证据而自动拒绝完成。

### 6.3 完成确认 UI 语义仍需改造

后端存在 `completion_result` 和 `yesButtonClicked` 确认路径，但当前聊天 UI 的主要按钮偏向“开始新任务”。“完成声明、用户确认、真正 TaskCompleted”之间仍需统一语义和事件顺序。

### 6.4 通用 Agent 循环恢复尚未实现

当前工具重复检测主要比较连续的相同工具输入，能够阻止精确重复，但还不能识别：

- 参数略有不同但语义相同的命令；
- 不同工具反复得到同一事实；
- 工具执行成功但任务证据没有增加；
- 工作区状态长期没有变化。

目前也没有通用 RecoveryCoordinator 自动执行“警告、阻止、重新规划、换方法、请求用户”的恢复阶梯。上下文压缩内部的无进展保护不能被误认为整个 Agent 已经具备循环恢复。

### 6.5 用户满意度问卷尚未实现

“Agent 提交结果后询问任务是否完成、用户是否满意”的 UI 方案已经讨论，但尚未作为正式产品功能实现。当前详细证据只能记录已有 Ask 响应，不能替代结构化满意度问卷。

## 7. 测试和发布方式的改进

当前采用两类测试思路：

1. 确定性机制测试：使用 Fake AI 和外部断言验证运行时保护，不依赖强模型主动避免问题；
2. 模型行为基线：后续应以实际 30B 模型作为高风险样本、强模型作为对照，使用同一任务集和确定性外部判定器。

详细证据版本已经完成：

- Agent 证据、诊断导出和 UI 按钮聚焦测试；
- Task 与统一工具执行路径测试；
- Extension 与 Webview 类型检查；
- 冻结接口保护路径检查；
- 未 rebrand VSIX 生产构建；
- 包内扩展身份、命令空间、功能 marker、压缩包完整性和 SHA-256 检查。

打包只能使用：

```text
REVIEW_SKILLS_OFFLINE=true pnpm --dir src vsix
```

然后从 `bin/zgsm-3.0.16.vsix` 原样复制并重命名交付文件。禁止执行 `pnpm vsix:dicode` 或 `make-dicode-vsix.js`，因为它们会 rebrand 并改变扩展身份。

## 8. 总结

当前改造已经建立了四个基础层：

```text
可回滚的 V2 上下文压缩
任务短期记忆与用户确认的长期记忆
Agent 生命周期影子观察
可由用户主动提取的本地完整运行证据
```

这些能力解决的是“保留上下文、恢复任务、记录 Agent 实际做了什么、出现问题后能够还原”的基础问题。

尚未解决的核心问题仍是：

```text
Agent 是否真的满足任务目标
缺少证据时是否阻止完成
陷入语义循环时是否能够自主恢复
用户是否确认完成以及是否满意
```

下一阶段应先使用现场证据形成可复现缺陷和 Phase 1A 特征化测试，再实现 AcceptanceContract、EvidenceLedger、CompletionVerifier、ProgressMonitor 和 RecoveryCoordinator，而不是直接让 Agent 根据单次现场问题修改自身策略。
