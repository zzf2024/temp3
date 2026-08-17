# 自进化 Agent 技术调研报告

> 版本：2026-08-17\
> 重点：各方案的**进化对象、Feedback、自进化闭环、是否依赖固定测评集**。

## 1. 核心结论

自进化 Agent
可按被改变对象分为：输出、记忆/技能、Prompt/Context、Workflow/架构、源码、模型参数六层。

``` text
当前Agent → 执行任务 → trajectory → Evaluator/Environment
   ↑                                    ↓
保存有效改进 ← Selection ← 新版本 ← Feedback + Evolution
```

**自进化不一定依赖固定 benchmark，但几乎一定依赖 Feedback。**
没有评价、选择和持久化，只有"自修改"，不能证明发生了"进化"。

## 2. 闭环的六个要素

  要素                 问题
  -------------------- -----------------------
  State                什么允许改变？
  Task                 在哪种任务/环境运行？
  Feedback             怎么知道好不好？
  Evolution Operator   谁提出修改？
  Selection            怎么判断值得保留？
  Persistence          能否跨轮次/任务保存？

## 3. 代表性方案

### 3.1 Self-Refine

**进化对象：当前输出。** 同一 LLM 生成、自评、再
refinement，不更新权重。

``` text
任务 → 初始答案 → LLM自评 → Feedback → 修改答案 → 再自评
```

Feedback：LLM 自评。固定测评集：运行时不需要。持久化：弱，更接近
test-time refinement。

### 3.2 Reflexion

**进化对象：Episodic Memory。**

``` text
Agent+Memory → 执行 → 环境/测试反馈 → Reflection → 写入Memory → 再尝试
```

Feedback：环境
reward、测试/执行结果。固定测评集：不必须，但必须有反馈。权重不更新，经验可持久化。

### 3.3 Voyager

**进化对象：Skill Library。**

``` text
世界状态+技能 → Automatic Curriculum → 新任务 → 生成程序
→ 环境执行 → 错误/反馈/Self-Verification → 修正
→ 成功 → 存入Skill Library → 复用/组合 → 更难任务
```

Feedback：Minecraft 真实环境。固定测评集：自进化过程不需要。这里
`Environment ≈ Evaluator`，属于较强的长期能力积累。

### 3.4 GEPA

**进化对象：Prompt/Instruction/Text。**

``` text
Prompt → 执行并收集trajectory → Evaluator → Score+文本反馈
→ LLM Reflection → Prompt Mutation → 重评估 → Pareto Selection
```

通常需要 evaluation
inputs，但不一定需要大量标签。优势是利用"为什么失败"的轨迹信息，而非只有
scalar reward。 ", \### 3.9 Darwin Gödel Machine（DGM）
**进化对象：Agent 自身源码。** DGM 允许 Agent 修改自己的代码库，并用
coding benchmarks 实证验证；同时维护多分支 Agent archive。

``` text
Agent Archive → 选Parent → Self-Modification → Child Agent
→ Coding Benchmark → Empirical Performance → 写入Archive
→ 基于性能+探索选新Parent → 继续修改
```

Feedback：coding benchmark。**原始实现高度依赖 benchmark。**
核心限制是开放世界没有可靠 success/fail 时，selection 很困难。

### 3.10 OpenEvolve

**进化对象：程序/算法，也可把 Agent implementation 当成被进化程序。**

``` text
Initial Program → LLM Mutation → Candidate → Evaluator
→ Score/Multi-objective Metrics → Population → Selection → 下一轮
```

不一定需要传统数据集，但**必须有可计算 objective**。尤其适合 GPU
kernel：

``` text
修改Kernel → 编译 → Correctness Test → GPU Benchmark
→ latency/throughput → 保留更优版本
```

### 3.11 Agent0

**进化对象：Curriculum + Executor 模型能力。**

``` text
Curriculum Agent → 自动产生Frontier Tasks → Executor多次求解+Tools
→ 一致性/任务筛选 → 训练信号 → 训练Executor
→ Executor变强 → Curriculum生成更难任务 → 循环
```

训练阶段不需要 human-curated
data；**会更新模型能力/权重**。但最终仍应使用独立 benchmark
验证泛化提升。

### 3.12 ACE

**进化对象：Context / Playbook / Memory。**

``` text
Playbook_t → 执行真实任务 → Execution Feedback → Reflection
→ 策略/规则抽取 → Curation → Playbook_(t+1) → 后续继续使用
```

在线适应可不依赖固定测评集；强调长期、结构化、增量维护的经验库。 ", \##
6. 五种典型闭环范式

1.  **Self-Reflection**：执行→失败→Reflection→Memory→再执行。代表
    Reflexion。
2.  **Environment-Driven Skill Evolution**：探索→环境反馈→修正技能→Skill
    Library→更难任务。代表 Voyager。
3.  **Prompt/Workflow
    Search**：候选→Benchmark→Score→搜索/变异→新候选。代表
    GEPA、DSPy、AFlow、ADAS、EvoAgent。
4.  **Self-Code Evolution**：Agent
    Code→自修改→Benchmark→Selection→Archive。代表 DGM；OpenEvolve
    可作通用引擎。
5.  **Model
    Co-Evolution**：自动出题→Agent求解→自动训练信号→训练→更强Agent→更难题。代表
    Agent0。

## 7. 关键研究难点

-   **Evaluator 是瓶颈**：代码/数学/GPU
    容易自动验证；Research、规划、办公 Agent 很难定义唯一正确答案。
-   **Reward Hacking**：Agent 可能学会提高评分，而非提高真实能力。
-   **能力回归**：任务 A 变强可能导致任务 B 退化，需要 regression suite
    和版本回滚。
-   **自生成任务错误放大**：Agent0 类方案需处理题目错误、难度控制和
    pseudo-label 污染。
-   **长期记忆膨胀**：应把 trajectory
    转成抽象规则，做去重、版本化和按需检索。

## 8. 推荐的工程化架构

``` text
Task Pool → Current Agent → Execution Trace → Evaluator
                ↑                              ↓
             Archive ← Selection ← Candidate ← Evolver
                ↓                              ↑
        Validation/Regression ← Prompt/Memory/Workflow/Code Mutation
```

建议模块：`agent.py`、`evaluator.py`、`evolver.py`、`archive.py`、`benchmark.py`、`runner.py`。

原则：Evaluator 与 Evolver 分离；同时保存 scalar score 和 textual
feedback；使用 held-out regression；所有修改版本化可回滚；先做
Prompt/Workflow，再考虑源码和模型权重；代码执行必须 sandbox。 ",
