# DiCode 本地 Agent 记忆能力设计方案

## 1. 目标

在不要求用户安装 PostgreSQL、SQLite 服务、Docker 或其他外部组件的前提下，让用户仅通过安装 DiCode VSIX 即可获得最小但完整的本地 Agent 记忆能力。

本方案重点解决：

- 用户偏好和项目规则如何跨任务保留；
- 记忆如何经历候选、验证、生效、替代、撤销和过期；
- 如何避免把模型推测、密钥或大量原始内容错误地保存为长期记忆；
- 如何在未来平滑接入 ChatRAG 后端，而不重写 Agent 核心逻辑；
- 如何保持现有前端、扩展宿主、Provider 和网络协议边界不变。

## 2. 核心原则

1. **本地优先、后端可选**：第一版完全在 VSIX 内运行，后续可增加远端同步。
2. **记忆不是聊天归档**：不默认保存全部对话，只保存有价值、可治理的信息。
3. **模型只能提出候选**：Agent 自动提取的内容默认是 `candidate`，不能自行宣布为可信事实。
4. **证据驱动**：正式记忆必须保留来源、作用域、置信度和验证状态。
5. **可撤销、可过期、可导出**：用户始终拥有查看、禁用和删除权。
6. **最小上下文注入**：每次任务只召回少量相关记忆，避免上下文污染。
7. **接口隔离**：记忆能力通过内部服务接入，不向冻结的 ChatRAG completion 请求随意增加字段。

## 3. 总体架构

```text
DiCode Agent Runtime
        │
        ▼
AgentMemoryService
   ├── LocalAgentMemoryService
   │     ├── JSONL 事件日志
   │     ├── 原子索引快照
   │     └── 内容寻址证据目录
   │
   └── RemoteAgentMemoryService（未来）
         └── HTTPS Memory API
                ├── PostgreSQL
                ├── 检索索引
                └── 对象存储
```

第一版只启用 `LocalAgentMemoryService`。Agent Runtime 依赖统一接口，而不依赖具体存储技术。

建议支持四种运行模式：

```ts
type MemoryMode = "disabled" | "local" | "remote" | "hybrid"
```

第一版默认使用 `local`，并允许用户完全关闭。

## 4. 本地数据存储位置

使用 VS Code 为扩展提供的专属全局存储目录：

```ts
extensionContext.globalStorageUri
```

不得硬编码用户主目录、Windows 用户名或固定盘符。逻辑结构如下：

```text
<VS Code globalStorage>/<DiCode extension-id>/
└── agent-memory/
    ├── manifest.json
    ├── journal/
    │   ├── memories.jsonl
    │   ├── evidence.jsonl
    │   ├── feedback.jsonl
    │   └── audit.jsonl
    ├── snapshots/
    │   └── memory-index-v1.json
    ├── artifacts/
    │   └── <sha256-prefix>/
    │       └── <sha256>.json
    └── locks/
```

访问令牌等秘密不进入上述目录，应使用 VS Code `SecretStorage`。记忆正文不应存入 `SecretStorage`，因为它不适合较大结构化数据。

## 5. 为什么第一版使用 JSONL，而不是原生 SQLite

SQLite 适合长期演进，但常见 Node SQLite 驱动包含平台相关二进制，需要处理：

- Windows、macOS、Linux；
- x64、arm64；
- VS Code Electron/Node ABI；
- Remote SSH、WSL、Dev Container；
- VSIX 打包、重建和升级兼容。

为了实现“只安装 VSIX 即可使用”并降低第一版跨平台风险，建议采用：

```text
JSONL 追加事件日志 + 内存索引 + 原子快照
```

该实现仅依赖扩展宿主已有的 Node 文件能力，不需要用户安装任何数据库服务。

当记忆量达到数万条、需要复杂组合查询、FTS5、多进程访问或更强事务时，再评估迁移至随 VSIX 打包的嵌入式 SQLite。

## 6. 记忆数据模型

```ts
interface AgentMemory {
  id: string
  schemaVersion: 1

  scope: {
    tenantId?: string
    userId?: string
    projectId?: string
    level: "user" | "project"
  }

  kind:
    | "preference"
    | "fact"
    | "procedure"
    | "decision"
    | "failure_pattern"

  content: string
  status: "candidate" | "verified" | "superseded" | "revoked" | "expired"
  confidence: number
  sensitivity: "public" | "internal" | "confidential" | "restricted"

  sourceEvidence: EvidenceRef[]
  createdAt: string
  updatedAt: string
  lastVerifiedAt?: string
  expiresAt?: string
  supersededBy?: string
}
```

关键字段不是 embedding，而是作用域、来源证据、状态、置信度、新鲜度和敏感等级。

## 7. 事件日志与生命周期

`memories.jsonl` 使用追加事件，不原地修改历史：

```json
{"event":"proposed","memoryId":"m1","kind":"procedure","content":"修改支付模块后必须运行集成测试","projectId":"p1","at":"2026-08-15T10:00:00Z"}
{"event":"verified","memoryId":"m1","evidence":["e1"],"at":"2026-08-15T10:10:00Z"}
{"event":"superseded","memoryId":"m1","by":"m2","at":"2026-09-01T08:00:00Z"}
{"event":"revoked","memoryId":"m2","reason":"用户撤销","at":"2026-09-02T08:00:00Z"}
```

生命周期为：

```text
proposed
   ↓
candidate
   ├── verify ───────→ verified
   ├── reject/revoke → revoked
   └── timeout ──────→ expired

verified
   ├── replace ──────→ superseded
   ├── revoke ───────→ revoked
   └── expire ───────→ expired
```

Agent 自动提取的经验只能进入 `candidate`。以下信息可直接进入 `verified`：

- 用户明确使用“记住”“以后都这样做”等稳定表达；
- 已存在于受信项目规则中的事实；
- 经过明确验证规则和证据门禁的程序性经验。

## 8. 索引与崩溃恢复

`memory-index-v1.json` 是可重建快照：

```json
{
  "schemaVersion": 1,
  "journalOffset": 18240,
  "generatedAt": "2026-08-15T10:15:00Z",
  "memories": {
    "m1": {
      "status": "verified",
      "projectId": "p1",
      "kind": "procedure"
    }
  }
}
```

写入流程：

1. 追加并刷写 journal 事件；
2. 在内存中应用事件；
3. 写入临时快照文件；
4. 校验快照；
5. 原子替换正式快照。

索引损坏或版本不兼容时，从 JSONL journal 重建。快照不是唯一事实源。

## 9. Agent Loop 接入点

### 9.1 任务开始前检索

```text
用户请求
  → 确定用户和项目作用域
  → 查询 verified memory
  → 权限、敏感度、新鲜度过滤
  → 相关性排序和 Token 预算裁剪
  → 生成只读 Memory Context
  → 进入现有 Agent Loop
```

第一版建议最多召回 5 至 8 条、总计不超过约 1200 tokens。

注入内容必须标记来源和状态：

```text
<project_memory>
- [verified, confidence=0.94, id=m1] 修改支付模块后必须运行支付集成测试。
</project_memory>
```

记忆优先级必须低于安全规则、当前用户明确要求和项目内规则。

### 9.2 执行过程中记录引用

记录本次任务检索、注入和实际采用了哪些 `memoryId`。未被实际采用的召回结果不能被算作有效经验。

### 9.3 任务结束后提出候选

```text
任务执行
  → OutcomeVerifier 验证
  → 提取候选经验
  → 敏感信息检查
  → 去重与冲突检测
  → 写入 candidate
  → 用户确认或证据门禁
  → verified
```

不能因为 Agent 调用了 `attempt_completion` 就认定任务成功，也不能从失败但未归因的轨迹直接提炼正式规则。

## 10. 内部服务接口

```ts
interface AgentMemoryService {
  search(query: MemorySearchQuery): Promise<MemoryItem[]>
  propose(candidate: MemoryCandidate): Promise<MemoryItem>
  verify(id: string, evidence: EvidenceRef[]): Promise<void>
  revoke(id: string, reason: string): Promise<void>
  recordUsage(usage: MemoryUsage): Promise<void>
  recordOutcome(outcome: TaskOutcome): Promise<void>
  export(scope: MemoryScope): Promise<MemoryExport>
  deleteScope(scope: MemoryScope): Promise<void>
}
```

第一版本地实现和未来远端实现必须遵循同一接口。

## 11. 项目标识

不能只使用本地绝对路径，否则项目移动后会丢失关联，并可能暴露用户名。

建议：

```text
projectId = hash(
  sanitizedRepositoryRemote
  + repositoryRootIdentity
  + optionalWorkspaceIdentity
)
```

要求：

- Git remote 在参与哈希前必须去除用户名、Token 和密码；
- 无 Git 仓库时使用本地生成的 workspace identity；
- 多根工作区分别建立作用域；
- 支持用户重绑定、合并或删除项目记忆。

## 12. 隐私与安全默认值

### 默认允许保存

- 用户明确确认的稳定偏好；
- 项目规则和业务术语；
- 已验证的构建、测试和排障流程；
- 用户对 Agent 结果的明确纠正；
- 有证据支撑的失败模式。

### 默认禁止保存

- API Key、访问令牌、刷新令牌和密码；
- 私钥、证书私密材料；
- 未脱敏的客户敏感数据；
- 模型未经验证的推测；
- 完整源码副本；
- 完整终端输出和完整聊天归档。

写入前应执行内容分类和秘密扫描。命中高风险内容时拒绝写入，并生成不包含秘密正文的本地诊断事件。

## 13. 用户可见能力

用户仅安装 VSIX 后即可使用：

- “记住这条规则”；
- 查看当前项目或当前用户的记忆；
- 查看某次任务使用了哪些记忆；
- 确认或拒绝候选记忆；
- 撤销、替代和删除记忆；
- 清空当前项目或全部本地记忆；
- 导出和导入记忆；
- 完全关闭记忆能力；
- 查看容量、保留期和最近清理结果。

第一版可以先通过命令面板和内部服务提供能力，后续再设计完整 UI。任何 Webview 接口改动必须单独进行兼容性评审，不能作为本地存储实现的附带改动。

## 14. 容量和保留策略

建议初始硬限制：

```text
结构化记忆总量：50 MB
证据产物总量：200 MB
单条记忆正文：16 KB
单个证据产物：1 MB
候选记忆默认保留：30 天
普通审计日志：滚动保留 90 天
verified 记忆：不因容量压力静默删除
```

达到限制时按以下顺序处理：

1. 删除可重建索引并重建；
2. 清理过期候选；
3. 清理无引用证据；
4. 滚动清理过期审计日志；
5. 提醒用户管理或导出；
6. 不静默删除仍有效的 verified memory。

## 15. 更新、迁移和卸载

每次扩展启动读取 `manifest.schemaVersion`：

```text
读取版本
  → 校验 journal
  → 备份旧快照
  → 执行逐版本迁移
  → 重建并校验新索引
  → 原子切换
```

VSIX 更新通常应保留 `globalStorage`，但不能承诺它是永久备份。用户清理 VS Code 数据、磁盘损坏或某些卸载流程仍可能造成丢失。因此必须提供显式导出能力。

“删除全部记忆”必须删除 journal、快照和相关 artifacts，并报告删除范围；不得只清理索引而保留正文。

## 16. 后端演进路径

### 阶段 1：本地最小闭环

- `AgentMemoryService`；
- JSONL 本地存储；
- 项目作用域；
- 明确记忆、检索、撤销和删除；
- 最小使用反馈。

### 阶段 2：候选经验和验证

- `OutcomeVerifier` 接入；
- candidate 提取；
- 冲突和重复检测；
- 证据引用；
- 本地离线评测。

### 阶段 3：可选远端 Memory API

- `RemoteAgentMemoryService`；
- 后端 PostgreSQL；
- 对象存储；
- 租户、用户、项目隔离；
- 明确授权的导入和同步。

### 阶段 4：策略优化闭环

- 脱敏轨迹聚合；
- 候选策略生成；
- 离线 replay；
- Shadow；
- 灰度、晋升和回滚。

ChatRAG 可以在未来负责检索和上下文组装，但 PostgreSQL 应负责记忆事实和生命周期，对象存储负责大型证据。RAG 索引不能成为唯一事实源。

## 17. 第一版验收标准

1. 新用户只安装 VSIX 即可使用，无额外服务和安装步骤。
2. 扩展重启后 verified memory 能恢复。
3. journal 写入中断不会破坏已有记忆。
4. 索引损坏后能够从 journal 重建。
5. 不同项目的记忆默认互不可见。
6. 撤销或过期记忆不会再次注入。
7. Agent 自动提取内容不会直接成为 verified。
8. Token、密码和密钥不会写入记忆正文、证据或日志。
9. 用户可以查看、导出、删除和禁用记忆。
10. 禁用记忆后不读、不写、不注入。
11. 记忆注入受数量和 Token 预算限制。
12. 不改变现有 ChatRAG completion、Provider、Webview 或 IPC 冻结契约。

## 18. 最终建议

第一版采用：

```text
VS Code globalStorage
+ JSONL 事件日志
+ 原子索引快照
+ 内容寻址证据目录
+ SecretStorage 保存未来的后端凭证
```

这能让用户只安装 DiCode VSIX 就获得本地记忆能力，同时保留向嵌入式 SQLite、ChatRAG Memory API、PostgreSQL 和对象存储演进的空间。

最重要的产品边界是：

> Agent 可以提出值得记住的内容，但只有用户确认或证据门禁能够让它成为正式记忆；正式记忆必须可解释、可撤销、可过期，并且不能突破用户、项目和安全作用域。
