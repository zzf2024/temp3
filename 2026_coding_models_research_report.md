# 2026 年几十B量级开放 Coding 模型调研报告

> 调研截止：2026-08-17\
> 范围：仅保留 2026 年发布/公开、约 20B--80B 总参数、开放权重，并以
> Coding / Software Engineering / Agentic Coding 为核心能力的模型。

## 1. 摘要

2026 年 Coding 模型的竞争重点已从 HumanEval、MBPP 等单轮代码生成，转向
**SWE-bench Verified、SWE-bench Pro、Terminal-Bench 2.0**
等真实软件工程与 Agent 基准。

本次重新检索后，重点纳入：**Qwen3.6-27B、Qwen3.6-35B-A3B、Poolside
Laguna XS.2、Cohere North Mini Code
1.0、Mocha-Coder-32B、SERA-32B、Qwen3-Coder-Next**。其中 Laguna XS.2 和
North Mini Code 是上一版需要重点补充的模型。

> 注意：不同厂商可能使用不同 Agent harness、context、iteration
> budget、system prompt 和推理配置，因此官方 SWE-bench
> 数字不能全部视为严格同条件横评。

## 2. 总览

  --------------------------------------------------------------------------------------------------------------
  模型                   发布时间   总参数   激活参数 架构    License              SWE-V    SWE-Pro Terminal 2.0
  ------------------ ------------ -------- ---------- ------- ------------- ------------ ---------- ------------
  SERA-32B                2026 H1      32B        32B Dense   开放              49.5±1.9        ---          ---

  Qwen3-Coder-Next        2026 H1      80B        ≈3B MoE     Open-weight     见技术报告        ---   见技术报告

  Qwen3.6-35B-A3B      2026-04-15      35B         3B MoE     Apache 2.0        **73.4**   **49.5**     **51.5**

  Qwen3.6-27B          2026-04-21      27B        27B Dense   Apache 2.0        **77.2**   **53.5**     **59.3**

  Laguna XS.2          2026-04-28    33.4B         3B MoE     Apache 2.0        **69.9**   **46.3**     **35.7**
                             左右                                                                   

  Mocha-Coder-32B         2026 H1      32B        32B Dense   MIT               **62.6**   **35.3**     **23.6**

  North Mini Code      2026-06-09      30B         3B MoE     Apache 2.0        **67.6**   **40.2**     **36.0**
  1.0                                                                                               
  --------------------------------------------------------------------------------------------------------------

## 3. Benchmark 选择

真实 Coding Agent 的工作链路更接近：

`Issue → 搜索仓库 → 阅读文件 → 定位问题 → 修改代码 → 运行测试 → 分析报错 → 再修改 → 最终 Patch`

因此本文重点看：

-   **SWE-bench Verified**：真实 GitHub Issue 修复。
-   **SWE-bench Pro**：更困难的 Repository-level Software Engineering。
-   **Terminal-Bench 2.0**：Shell、Terminal、工具调用与长周期任务。

对于 Claude Code、Cline、OpenHands、SWE-Agent、Qwen Code
一类系统，这些指标通常比 HumanEval 更有参考价值。

## 4. Qwen3.6-27B

Qwen 于 **2026-04-21** 开源 Qwen3.6-27B。它采用 **27B Dense**
架构，Apache 2.0 权重，强调旗舰级 Agentic Coding，并避免 MoE routing
的部署复杂度。

  Benchmark                     Score
  ------------------------ ----------
  SWE-bench Verified         **77.2**
  SWE-bench Pro              **53.5**
  SWE-bench Multilingual     **71.3**
  Terminal-Bench 2.0         **59.3**
  SkillsBench Avg5           **48.2**

优势是 **高 Agentic Coding 能力 + Dense 部署简单**；不足是每 Token
基本激活完整 27B 参数，高并发计算成本高于 3B-active MoE。

**推荐：★★★★★**

来源：[Qwen 官方](https://qwen.ai/blog?id=qwen3.6-27b)

## 5. Qwen3.6-35B-A3B

Qwen 于 **2026-04-15** 发布。核心规格为 **35B Total / 3B Active / Sparse
MoE**。

  Benchmark                     Score
  ------------------------ ----------
  SWE-bench Verified         **73.4**
  SWE-bench Pro              **49.5**
  SWE-bench Multilingual     **67.2**
  Terminal-Bench 2.0         **51.5**

它代表 2026 年重要路线：**几十B模型容量 + 几B Active
Parameters**。需要注意，3B Active 不等于只存储 3B 权重；完整约 35B
权重仍需进入显存/内存体系。

**推荐：★★★★★**

来源：[Qwen 官方](https://qwen.ai/blog?id=qwen3.6-35b-a3b)

## 6. Poolside Laguna XS.2

Laguna XS.2 是本次重新调研后补充的重要模型。Poolside 在 **2026 年 4
月底**公开权重和发布材料。

核心规格：

-   **33.4B Total / 3B Active**
-   MoE，256 experts + 1 shared expert
-   262,144 Context
-   40 layers：30 层 Sliding Window Attention + 10 层 Global Attention
-   FP8 KV Cache 支持
-   Apache 2.0
-   定位：Agentic Coding / Long-horizon Work / Local deployment

当前模型卡：

  Benchmark                     Score
  ------------------------ ----------
  SWE-bench Verified         **69.9**
  SWE-bench Multilingual     **57.7**
  SWE-bench Pro              **46.3**
  Terminal-Bench 2.0         **35.7**

Poolside 4 月发布博客中的早期数字为 SWE-bench Pro 44.5、Terminal-Bench
2.0 30.1；当前模型卡结果更高，所以正式横评应固定 checkpoint 与评测日期。

**推荐：★★★★★**

来源：[模型卡](https://huggingface.co/poolside/Laguna-XS.2)；[Poolside
发布说明](https://poolside.ai/blog/laguna-a-deeper-dive)

## 7. Cohere North Mini Code 1.0

Cohere 于 **2026-06-09** 发布 North Mini Code 1.0，这是 Cohere
第一款专门面向开发者的 Agentic Coding 开放模型。

核心规格：

-   **30B Total / 3B Active**
-   Sparse MoE
-   **256K Context**
-   最大输出 **64K**
-   Apache 2.0
-   面向 Code Generation、Agentic Software Engineering、Terminal Tasks

官方 Hugging Face 评测文件：

  Benchmark                 Score
  -------------------- ----------
  SWE-bench Verified     **67.6**
  SWE-bench Pro          **40.2**
  Terminal-Bench 2.0     **36.0**

Cohere 使用 SWE-Agent v1.1.0 评测 SWE-bench，并针对 Terminal-Bench
使用基于单 Terminal Tool 的 ReAct harness。

它和 Laguna XS.2、Qwen3.6-35B-A3B 构成非常有价值的同规模比较：

  模型                Total   Active
  ----------------- ------- --------
  North Mini Code       30B       3B
  Laguna XS.2         33.4B       3B
  Qwen3.6-35B-A3B       35B       3B

**推荐：★★★★☆～★★★★★**

来源：[Cohere
官方](https://cohere.com/blog/north-mini-code)；[模型卡](https://huggingface.co/CohereLabs/North-Mini-Code-1.0)

## 8. Mocha-Coder-32B

Mocha-Coder-32B 是 2026 年很有研究价值的 Coding Agent。它基于
Qwen2.5-Coder-32B-Instruct，但通过 **300K+ Agent Trajectories**
蒸馏训练，不依赖 RL。Teacher 包括
Qwen3-Coder-480B-A35B、Kimi-K2.5、Qwen3-Coder-Next、DeepSeek-V3.2。

  Benchmark                 Score
  -------------------- ----------
  SWE-bench Verified     **62.6**
  SWE-bench Pro          **35.3**
  Terminal-Bench 2.0     **23.6**

其统一 SWE-bench Verified 实验中，Qwen3-Coder-480B-A35B 为
67.0，Mocha-Coder-32B 为 62.6，SERA-32B 为 54.2，Qwen3-Coder-30B-A3B 为
51.6。

它最重要的启示是：**高质量 Agent Trajectory
可以显著缩小几十B模型与超大模型的差距。**

**研究推荐：★★★★★**

来源：[Mocha-Coder-32B](https://huggingface.co/cocoa-org/Mocha-Coder-32B)

## 9. SERA-32B

SERA-32B 是 Ai2 Open Coding Agents 系列的 32B Coding Agent，基础模型为
Qwen3-32B。

官方结果：

**SWE-bench Verified = 49.5% ± 1.9%（32K Context）**

Mocha-Coder 团队在另一套 Agent 设置下测得 54.2，再次说明 SWE-bench
会明显受到 Harness、迭代次数、Context 与 Tool Protocol 影响。

SERA 当前绝对性能并非最高，但对"如何低成本把普通 32B LLM 训练成 Software
Engineering Agent"的研究很有价值。

**性能推荐：★★★☆☆；研究推荐：★★★★★**

来源：[SERA-32B](https://huggingface.co/allenai/SERA-32B)

## 10. Qwen3-Coder-Next

Qwen3-Coder-Next 是 2026 年公开的 Coding Agent 专用 MoE：

-   **80B Total / ≈3B Active**
-   Open-weight
-   针对 Coding Agent
-   使用可验证 Coding Tasks、Executable Environments、Mid-training 和 RL

80B
已位于"几十B"的上边界，因此本文将其作为扩展参考。它代表更激进的稀疏路线：**80B
模型容量，但每 Token 仅约 3B 激活**。优势是计算效率；问题是完整 80B
权重仍带来较高存储和内存带宽压力。

来源：[Qwen3-Coder-Next Technical Report
索引](https://huggingface.co/papers?q=Terminal+Bench+2)

## 11. 核心横向比较

### 11.1 3B Active MoE

-   North Mini Code 30B-A3B
-   Laguna XS.2 33.4B-A3B
-   Qwen3.6-35B-A3B

特点：总容量约 30--35B，每 Token 约 3B 激活，特别适合长周期 Agent 的大量
Token 生成；但权重显存需求仍按几十B模型考虑。

### 11.2 Dense

-   Qwen3.6-27B
-   Mocha-Coder-32B
-   SERA-32B

特点：推理框架支持成熟、部署结构简单，但每 Token 计算量明显高于
3B-active MoE。

### 11.3 超稀疏扩展

-   Qwen3-Coder-Next 80B-A3B

特点：模型容量更大但仍保持约 3B Active，计算效率高，权重容量压力更大。

## 12. 公布性能的快速排序

  模型                           SWE-V    SWE-Pro   Terminal 2.0
  --------------------- -------------- ---------- --------------
  **Qwen3.6-27B**             **77.2**   **53.5**       **59.3**
  **Qwen3.6-35B-A3B**         **73.4**   **49.5**       **51.5**
  **Laguna XS.2**             **69.9**   **46.3**       **35.7**
  **North Mini Code**         **67.6**   **40.2**       **36.0**
  **Mocha-Coder-32B**         **62.6**   **35.3**       **23.6**
  **SERA-32B**            **49.5±1.9**        ---            ---

这张表只适合快速筛选候选模型，**不应视为严格公平排名**。企业选型应统一
Agent Harness、System Prompt、Context、最大迭代次数、Token
Budget、温度、硬件和量化精度后重测。

## 13. 部署成本

粗略只看 Dense 权重：

  规模        BF16      INT8        INT4
  ------ --------- --------- -----------
  27B      \~54 GB   \~27 GB   \~13.5 GB
  32B      \~64 GB   \~32 GB     \~16 GB

实际还需要 KV Cache、Runtime Workspace 等。

对于 30--35B / 3B Active MoE：

-   权重容量仍接近 30--35B；
-   每 Token 主要激活约 3B；
-   优势主要体现在计算量和吞吐；
-   **不能把 3B Active 理解成显存等价 3B Dense。**

## 14. 选型建议

### 企业私有化、部署稳定优先

首选 **Qwen3.6-27B**。Dense 架构简单，公开 Agentic Coding 成绩很强。

### 高吞吐 / 推理成本优先

优先测试 **Qwen3.6-35B-A3B、Laguna XS.2、North Mini Code**。三者都是约
30--35B 总参数、3B Active 的代表。

### 自研 Coding Agent / 训练方法研究

重点研究 **Mocha-Coder-32B、SERA-32B**。前者突出大规模 Agent trajectory
distillation，后者强调开放、低成本 Coding Agent 训练。

### 更激进的 MoE

将 **Qwen3-Coder-Next 80B-A3B** 作为扩展对照。

## 15. 最终结论

2026 年几十B Coding 模型已经形成三条清晰路线：

1.  **强 Dense**：Qwen3.6-27B ------ 用较简单部署获得很强的 Agentic
    Coding。
2.  **低激活 MoE**：Qwen3.6-35B-A3B、Laguna XS.2、North Mini Code ------
    用约 3B Active 控制长周期 Agent 的推理计算量。
3.  **Agent Trajectory / Agent Training**：Mocha-Coder、SERA ------
    说明训练数据和交互轨迹质量可以显著提升真实软件工程能力。

如果只选 4 个模型进入公司 PoC，建议：

**Qwen3.6-27B + Qwen3.6-35B-A3B + Laguna XS.2 + North Mini Code 1.0**

如果还要研究后训练方法，再加入 **Mocha-Coder-32B**。

## 16. 补充：Orchard-SWE

2026 年 Microsoft Research 的 Orchard-SWE 使用 Qwen3-30B-A3B-Thinking
作为 backbone，通过 107K trajectories、SFT 和 RL，将 SWE-bench Verified
从 22.0 提升至 **64.3（SFT）/ 67.5（SFT+RL）**。

它非常值得关注，但当前更准确地说是 **Agentic Modeling Framework /
Training Recipe**，而不是本文核心表中那种独立、成熟发布的 Coding 模型
checkpoint，因此放在补充章节，不与 North Mini Code 等模型混为一类。

来源：[Microsoft Research
Orchard](https://www.microsoft.com/en-us/research/publication/orchard-an-open-source-agentic-modeling-framework/)

------------------------------------------------------------------------

## 17. 主要资料来源

-   Qwen3.6-27B: https://qwen.ai/blog?id=qwen3.6-27b
-   Qwen3.6-35B-A3B: https://qwen.ai/blog?id=qwen3.6-35b-a3b
-   Cohere North Mini Code: https://cohere.com/blog/north-mini-code
-   North Mini Code model card:
    https://huggingface.co/CohereLabs/North-Mini-Code-1.0
-   Poolside Laguna XS.2: https://huggingface.co/poolside/Laguna-XS.2
-   Poolside Laguna release:
    https://poolside.ai/blog/laguna-a-deeper-dive
-   Mocha-Coder-32B: https://huggingface.co/cocoa-org/Mocha-Coder-32B
-   SERA-32B: https://huggingface.co/allenai/SERA-32B
-   Orchard:
    https://www.microsoft.com/en-us/research/publication/orchard-an-open-source-agentic-modeling-framework/
