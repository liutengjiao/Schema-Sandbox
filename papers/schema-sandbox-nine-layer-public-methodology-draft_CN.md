# 图式沙箱：用于受约束智能体执行的九层架构与互操作契约

**刘腾蛟**  
*创始人兼研究员, psi.run*  
psi@psi.run  

**司宏宗**  
*教授, 青岛大学*  
sihz@qdu.edu.cn  

---

## 摘要 (Abstract)

LLM 具有天生的概率性，而生产环境需要确定性的契约。单靠“提示词 + 工具调用”不足以支撑高风险、长时程、可审计的智能体系统；真正决定系统安全与稳定性上限的是约束内核、分区执行、能力权限、证据契约与回归验证构成的工程骨架。本研究提出了**图式沙箱（Schema Sandbox）**----一种介于原生大模型输出与持久化智能体执行之间的神经符号约束架构，并设计了**图式互操作协议（SIP v1.1）**。从康德“先验图式”与皮亚杰“认知发生论”出发，我们确立了智能体通过“同化”与“顺应”进行自进化修正的理论基础，并推导了其负熵压实界限。本研究的核心技术贡献包括：第一，提出了九层神经符号认知约束边界，将概率 Token 编译为确定性契约动作；第二，规范了基于 Deno/WASI（本地内核）、gVisor（容器化分区）与 Firecracker（微虚拟机）的**分层混合式运行时框架**与工作分区模型；第三，规范了 SIP v1.1 能力挂载与治理协议，作为清单级契约定义了元数据、输入/输出契约、权限范围等七大核心模块，使沙箱成为可发现、可挂载、可审计与可流通的能力资产。微观基准测试表明，本地图式校验的 p50 延迟仅为 1.5 微秒。本项工作为在受约束智能体系统中强制执行可靠的物理与逻辑边界提供了通用的设计蓝图与协议规范。

---

> [!NOTE]
> ### 面向实施者的快速导航 (Quick Start Guide)
> 本文档定义了图式沙箱（Schema Sandbox）的完整理论与工程规范。若您是负责落地的软件工程师，建议沿以下快速路径实施，无需一次性通读整篇论文：
> 
> 1. **我该从哪里开始写代码？**
>    - **第一步 (核心契约)**：首先实现 `SchemaManifest` 和 `OutputContract` 接口（见 3.1 节）。这是沙箱的神经符号校验红线，其余模块均依赖它们进行双向门控。
>    - **第二步 (本地内核)**：搭建 **方案 A：Deno + WASI** 本地内核（见 6.1 节）作为智能体执行的默认逻辑隔离面。
>    - **第三步 (渐进增强)**：根据业务风险高低，决定是否采用 **方案 B (gVisor)** 或 **方案 C (Firecracker)** 进行物理隔离调度。
> 2. **我的第一个沙箱（Hello World）应该长什么样？**
>    - 请直接跳转至 **第 9 节 (案例分析)**。参考 "Brand Shuttle GEO" 诊断沙箱包的逻辑与文件拓扑，这是最直接的骨架模板。
> 3. **哪些章节可以在一期开发中暂缓阅读？**
>    - 如果您只想快速跑通一个最小可行产品 (MVP)，可以暂缓深入阅读 **第 2 节 (背景与相关工作)** 和 **第 7 节 (安全威胁模型)**，留待二期工程加固时参考。

## 1. 引言 (Introduction)

长周期智能体的失败，往往不是因为模型本身不够聪明，而是因为缺乏清晰的边界。在单轮对话中有效的 Prompt 在经历数十个步骤的拉长后会不可避免地退化，导致格式崩溃、规则被忽视以及状态表征的统计学漂移（认知漂移）。

在构建生产级智能体系统的过程中，我们发现后置校验往往为时已晚：在大模型吐出错误的 Token 并被执行前，游戏就已经失控。我们需要的是一个根据沙箱级别和执行机制，在生成前、生成中或生成后运行的活性约束边界。

我们将此边界称为图式沙箱（Schema Sandbox）。它不是另一个记忆层包装器，而是一个将概率性的 Token 流编译为符合契约的动作的认知约束层。本工作探讨了介于纯提示词控制与基础设施级沙箱隔离之间的中间设计空间。

### 1.1 哲学起源：康德先验图式与皮亚杰认知发生论
休伯特·德雷福斯（Hubert Dreyfus）对“无身体人工智能”的哲学批判表明，缺乏情境边界的系统极易发生漂移，因为它们缺乏锚定语义的情境上下文 [1]。在智能体系统中，这种约束是信息层面的而非物理层面的，用于限制模型的注意力、状态和允许的动作。Dreyfus 的批判在这里不作为直接的技术模型，而是一个有用的类比：智能体的能动性需要具身的情境约束。

为了给这一信息边界提供坚实的哲学基础，我们引入了两个经典认知范式：
1. **康德的先验图式论 (Kantian Schematism)**：康德在 1781 年的《纯粹理性批判》中指出，感性直观与知性概念是异质的，必须通过“先验图式”作为中介，将混乱的感官输入整理为符合范畴的认知。在硅基心智中，大模型的超高维隐空间流形（Latent Manifold）即是其“感性直观”，而人类定义的任务意图与系统契约则是其“范畴”。图式沙箱扮演的正是“先验图式编译器”的角色，将概率流编译为确定性的符号契约动作。
2. **皮亚杰的认知发生论 (Piagetian Dynamics)**：皮亚杰提出心智通过“同化 (Assimilation)”与“顺应 (Accommodation)”的自适应双轨流进行演化。在图式沙箱中，当智能体的动作符合当前约束时，直接执行并累积状态，此为同化；当动作被沙箱拦截（验证失败）时，系统激活“顺应”机制，迫使智能体运行沉淀算子（Compaction Operator），重构其内部规则和表观提示词层，从而自愈修复认知皮肤。

在我们配套发表的理论定位论文 [10] 中，我们指出为了约束认知漂移并防止智能体人格（Identity）耗散，持久的智能体人格（Agent IP）需要一个边界机制。本项研究则为此边界层提供了具体的架构规范与互操作协议。

### 1.2 核心技术贡献 (Core Technical Contributions)
本研究针对长时程智能体在工程落地中的安全与互操作瓶颈，提出了以下三项核心技术贡献：
1. **作为认知约束架构的图式沙箱 (Schema Sandbox as a Cognitive Constraint Architecture)**：定义了从领域语料、任务本体、输入契约、上下文压实，到工具文法、执行防线与输出契约的九层约束边界，以数学负熵减形式抑制自回归生成的注意力散逸，从而提供了确定性约束边界。
2. **作为执行隔离模型的工作分区 (Workspace Partition as the Execution Isolation Model)**：设计了分层混合式执行机制，提供基于 Deno/WASI 本地轻量化内核（方案 A）、gVisor 容器隔离（方案 B）与 Firecracker 微虚拟机（方案 C）的多级物理与逻辑隔离方案，以最低资源开销匹配动作的风险谱线。
3. **作为互操作与能力资产化协议的 SIP (SIP as the Interoperability and Capability-Asset Protocol)**：正式规范了 SIP 图式互操作协议。我们指出，**九层架构回答了“一个图式沙箱怎么构建”；而 SIP 则回答了“不同图式沙箱如何被发现、挂载、调用、组合、验证、交易和长期演化”。** 没有 SIP，图式沙箱仅是一种强大的局部工程方法论；有了 SIP，它才有机会成为开放标准、能力市场和 Agent IP 的生态底座。


---

## 2. 背景与相关工作 (Background and Related Work)

本研究涉及结构化生成、智能体装配架以及执行安全等多个交叉领域。

### 2.1 约束解码 (Constrained Decoding)
自回归语言模型通过在词表上对概率分布进行采样来生成文本。约束解码通过在生成的每一步修改 Logits 分布来限制其搜索空间。诸如 Outlines [2] 和 Guidance [9] 等框架将上下文无关文法（CFG）或 JSON Schema 编译为正则表达式状态机或前缀树（Trie）。在第 k 步，任何不属于合法转移集合 V(valid) 的 Token i 的 Logit L_k[i] 都会被屏蔽：

L'ₖ[i] = Lₖ[i] （若 i ∈ V(valid)） 或 -∞ （若 i ∉ V(valid)）

尽管这在数学上保证了输出严格符合特定语法，但它引入了“约束税（Constraint Tax）”----限制解码路径可能会退化模型的推理能力，并增加复杂推理任务上的语义错误率 [3]。为了降低约束税的负面影响，本架构在业界首次提出了将约束明确划分为**刚性约束 (Rigid/Security Bounds)**与**柔性约束 (Soft/Formatting Bounds)**。刚性约束负责绝对安全防御，不作任何妥协；柔性约束负责格式对齐，当校验多次失败时，系统通过“动态约束降级”转而允许自由文本输出并结合本地后处理，最大化保留模型的推理活性。这一设计与 2026 年初关于图式约束智能体记忆生成（SCG-MEM）[11] 静态硬性规则形成了对比。

### 2.2 智能体装配架与运行时 (Agent Harnesses and Runtimes)
为了执行长周期任务，大模型通常被封装在管理状态、内存和工具调用的智能体装配架中。诸如 MemGPT (Letta) 等系统将 LLM 上下文窗口视为虚拟内存，使用分页机制管理有限的 Token 预算 [4]。Tree of Thoughts [12] 通过规划中间状态的搜索路径来结构化智能体工作流。然而，这些系统缺乏统一的认知约束模型，使得 LLM 推理与系统工具执行之间的边界大多处于权宜之计。

### 2.3 沙箱隔离与权限控制 (Sandboxing and Permissions)
传统的执行沙箱在操作系统级运行。容器化技术（如 Docker）和微虚拟机（如 AWS Firecracker）负责隔离代码执行。当智能体执行任意代码时，物理隔离是必需的。然而，在许多实际工作流中，主要的失效模式是图式违规、数据泄露或工具调用格式错误。对于这些场景，行级范围锁定和数据脱敏在极大地降低计算成本的同时，提供了极低的延迟安全保障（1.5微秒对VM微虚拟机的50毫秒）。

### 2.4 结构化输出可靠性 (Structured Output Reliability)
仅生成语法合规的 JSON 对于企业应用来说是不够的。这些 JSON 中字段的语义正确性仍然存在变数。JSONSchemaBench 等基准测试表明，即使大模型在约束解码下输出了语法合规的 JSON，它们也经常无法通过语义约束（例如生成不存在的数据库 ID 或违反数值区间限制） [6]。

### 2.5 建构主义智能体与自我演化记忆 (2025-2026 前沿)
近年来，学术界正从被动记忆检索转向受发展建构主义启发的、主动且自我演化的记忆结构。例如，**ActiveRAG** [13] 和 **ThinkNote** [14] 通过算法实现了皮亚杰的同化和顺应机制以解决知识冲突，而 **CAM** (Li et al., NeurIPS 2025) [15] 则通过动态分层图式聚来解决长上下文下的记忆消散。但现有系统均未将硬性约束（算力、信用、沙箱失败）作为使记忆产生意义的信息边界。

### 2.6 工业级智能体装配架剖析 (Industry Implementations Analysis)
2026 年初，多款商用智能体工具展现了“装配架（Harness）”在智能体系统中的核心地位。对 **Claude Code** 的架构解剖（基于 ~51 万行 TypeScript 源码分析 [8]）表明，其核心 ReAct 逻辑仅占系统代码的 2% 以下，其余 98% 均为由 3 层存储管理（ephemeral、memory.md 索引、CLAUDE.md 宪法）、5 层渐进式上下文压实、8层安全防御深度链和动态 Hook 机制构成的复杂智能体装配架。类似地，**Cursor** 引入了文件级 Merkle 树增量检测、AST 语义分块和智能规则激活机制；而 **Aider** 则通过 Git 原生提交机制和高密度的代码库地图（Repo Map）实现了极高的 Token 效率（单任务开销仅为 Claude Code 的 1/4 左右）。这些工业实践证明，智能体的落地壁垒已不在模型底座，而在其外裹的活性约束与装配架基础设施。

---

## 3. 图式沙箱的形式化定义与自进化循环

图式沙箱作为具有神经符号接口的认知边界运行：神经侧负责生成，符号侧负责拦截过滤。我们将图式沙箱（Schema Sandbox）定义为一个动态约束算子 Sₜ = ⟨Kₜ, Φₜ⟩。其中 Kₜ 代表认知图式拓扑，Φₜ: A × S → {0, 1} 是沙箱验证函数。我们用 Bₜ ≡ Sₜ 表示活跃边界。沙箱过滤底座模型原始动作概率 P(base)(a | s)，并投影到受限边界上：

P(agent)(a | s) = Sₜ ∘ P(base)(a | s) = (Φₜ(a, s) * P(base)(a | s)) / Zₜ

其中 Zₜ = ∫(a' ∈ A) Φₜ(a', s) * P(base)(a' | s) da' 是归一化配分函数。

在 Token 自回归解码阶段，对于词表 V，沙箱根据 Kₜ 限制 Logit 向量 Lₖ ∈ ℝ^|V|，隔离出允许的 token 子集 V(valid) ⊂ V：

L'ₖ[i] = Lₖ[i] （若 i ∈ V(valid)） 或 -∞ （若 i ∉ V(valid)）

这确保了不合规 Token 在采样前即被屏蔽，消除了格式还原引起的幻觉。

### 3.1 核心五大契约接口 (Five Core Contract Interfaces)
为了在工程中实现这一认知约束，我们将图式沙箱（Schema Sandbox）统一抽象为五大核心契约接口（Schema Kernel Contracts）：
1. **SchemaManifest**：声明图式包元数据、 SemVer 版本号、输入/输出 Schema 引用、风险评级（L0-L3）以及此图式运行所必须收集和记录的审计证据类型。
2. **PartitionPolicy**：规定工作分区的硬件隔离级别、内存切片作用域、可用 Token 预算、并发争抢策略以及异常回退路由。
3. **CapabilityGrant**：授予工作分区的显式运行许可，细粒度锁死文件系统（FS Scope）、网络出口白名单（Net Scope）、操作系统工具调用白名单（Tool Scope）以及租约失效时间（TTL）。
4. **EvidenceRecord**：不可篡改地记录外部副作用的加密痕迹，包括输入哈希、动作参数哈希、前置/后置门决策历史以及输出哈希。
5. **OutputContract**：规范生成结束后的净化逻辑、敏感词拦截机制、最大信息熵截断阈值和语义自愈回传规则。

五类核心契约接口的定义如下：
```typescript
interface SchemaManifest {
  id: string;
  version: string; // semver
  riskLevel: 'L0' | 'L1' | 'L2' | 'L3';
  inputSchemaRef: string;
  outputSchemaRef: string;
  allowedPartitions: string[];
  requiredEvidence: (
    | 'input_hash'
    | 'tool_args_hash'
    | 'policy_decision'
    | 'output_hash'
  )[];
}

interface PartitionPolicy {
  partitionId: string;
  isolationLevel: 'context' | 'worktree' | 'remote';
  memorySliceScope: string[]; // 限制在 memory.md 映射的绝对目录
  tokenBudget: number;      // 允许的最大 Token 预算
  concurrencyLimit: number; // 最大并发限制
  fallbackRoute: string;    // 校验失败时的降级路由
}

interface CapabilityGrant {
  partitionId: string;
  fsScope: string[];       // 锁定物理目录
  netScope: string[];      // 锁定 Host 与 Port
  toolScope: string[];     // 锁定具体可调用工具
  ttlSeconds: number;      // 令牌租约
  humanApproval: boolean;  // 是否强制人工介入确认
}

interface EvidenceRecord {
  traceId: string;
  parentTraceId?: string;   // 链接到父 Trace，以支持子智能体调用链
  sandboxId: string;        // 发起调用的沙箱 ID
  sandboxVersion: string;   // 执行时的沙箱 SemVer 版本号
  partitionId: string;      // 运行该动作的工作区隔离分区 ID
  actorAgentId?: string;    // 调用该动作的智能体身份 ID
  modelProvider?: string;   // 例如 "anthropic", "openai"
  modelId?: string;         // 例如 "claude-4-sonnet"
  policyVersion: string;    // 当前生效的策略规则集版本 Hash
  inputHash: string;        // 输入哈希 (sha256 或 blake3)
  toolName?: string;
  toolArgsHash?: string;    // 工具参数哈希 (sha256 或 blake3)
  policyDecision: 'allow' | 'deny' | 'ask' | 'defer';
  outputHash: string;       // 输出哈希 (sha256 或 blake3)
  timestamp: string;        // ISO-8601 时间戳
  digestAlgorithm: 'sha256' | 'blake3';
}

interface OutputContract {
  schemaRef: string;       // JSON Schema 引用
  sanitizationRules: {
    stripSecrets: boolean; // 是否脱敏密钥
    stripPII: boolean;     // 是否脱敏个人敏感信息
    customBlockPatterns: string[]; // 自定义拦截正则
  };
  entropyThreshold: number; // 最大允许信息熵
  selfHealingRules: {
    retryLimit: number;    // 自愈重试上限
    fallbackSchemaId?: string; // 备用 Schema
  };
}

interface RejectionPayload {
  violationType:
    | 'AUTHORIZATION_FAIL'
    | 'OUT_OF_BOUNDS_READ'
    | 'OUT_OF_BOUNDS_WRITE'
    | 'SCHEMA_FORMAT_ERROR'
    | 'ENTROPY_VIOLATION'
    | 'LEAK_DETECTED';
  allowedBoundaries: {
    fsScope?: string[];
    netScope?: string[];
    toolScope?: string[];
    validSchemaRef?: string;
  };
  suggestedFallback: {
    action:
      | 'AskHuman'
      | 'DegradeToPlainText'
      | 'UseFallbackSchema'
      | 'RetryWithCompaction';
    description: string;
  };
}
```

### 3.2 SIP 与相邻概念及 Agent IP 的边界定位 (SIP vs. Adjacent Concepts & Agent IP Mapping)

图式互操作协议（SIP）不仅是一个接口规范，更是实现智能体能力资产化的治理层契约。为了明确其定位，我们在此厘清它与 Model Context Protocol (MCP) 等协议的差异，以及它在 Agent IP 确权中的作用。

#### 1) 与相邻技术方案（MCP、Structured Outputs、OS 容器）的差异对比
MCP 侧重于规范和简化大模型与底层工具或数据源之间的“点对点连接”，而 SIP 侧重于保障专业能力包（Schema Sandbox Package）的“边界隔离、权限控制、合规审计与流通性”。下表（表 III）对比了图式沙箱 + SIP 框架与相邻技术路径（包括标准结构化输出、Model Context Protocol 及物理容器隔离）在各能力维度上的差异与互补性：

| 能力 | Structured Outputs | MCP | Docker/gVisor/Firecracker | Schema Sandbox + SIP |
| :--- | :--- | :--- | :--- | :--- |
| **输出结构** | 有 | 部分 | 无 | 有 |
| **语义校验** | 弱 | 工具侧 | 无 | 有 |
| **工具连接** | 无 | 强 | 无 | 可结合 MCP |
| **权限声明** | 无 | 部分/客户端侧 | OS 级 | manifest 内嵌 |
| **能力包身份** | 无 | server/package 层 | 无 | 有 |
| **审计证据链** | 无 | 非核心 | 需自建 | 核心模块 |
| **可市场化能力租赁** | 无 | registry 可扩展 | 无 | 核心设计 |

__TABLE_III__

#### 2) Agent IP 能力挂载模型
在智能体心智中，Agent IP (智能体人格与资产身份) 是核心主体，但它不应直接混杂行业专用规则和危险执行能力。SIP 提供了标准化的“能力外挂脑叶”挂载机制。Agent IP 通过在 Manifest 中声明要挂载的沙箱包：

Agent IP (人格设定 + 通用推理) --[SIP 挂载]--> Schema Sandbox (隔离分区 + 契约限制) --> 确定性能力输出

这使得智能体的专业能力可以被模块化插拔、升级、确权与计费。没有 SIP，Agent 只能靠不稳定的 Prompt 拼接来调用工具；有了 SIP，Agent IP 可以声称对特定受控沙箱的拥有权，实现“方法论”向“可交易资产”的跨越。

### 3.3 约束投影命题 (Constraint Projection Proposition)
引入图式沙箱 Sₜ 会在解码分布中引入硬性可容空间剪枝。设 Supp(Sₜ) = {a ∈ A : Φₜ(a, s) = 1} 为沙箱允许的动作集合，则：

|Supp(Sₜ)| ≤ |A|， 且所有非许可动作的概率质量 ∑(a ∉ Supp(Sₜ)) P(base)(a | s) 被强制降为 0。

在均匀分布或有界偏斜分布假设下，这种支持集收缩会带来 Shannon 熵的负熵压缩：

H(P(agent)) ≤ H(P(base)) - ΔH(Sₜ)， 其中 ΔH(Sₜ) = -log Zₜ ≥ 0。

但需要指出的是，在非均匀分布的极端情况下，过滤高频动作可能导致剩余低频动作在重新归一化后分布更加均匀，导致 Shannon 熵上升。因此，本框架采用更强健的指标衡量收缩性：（1）违规概率质量的消除率；（2）可行支持集的收缩比率 |Supp(Sₜ)| / |A|；（3）违规率的直接下降。这些指标由沙箱投影算子 Φₜ 提供无条件保证，不受基模型初始分布偏斜度的影响。

### 3.4 同化与顺应的自进化循环算法
为了解决过度硬性约束带来的“约束税（Constraint Tax）”并防止模型在校验失败时陷入盲目重试的死循环，本架构将边界约束分为两类：
1. **刚性约束 (Rigid/Security Bounds)**：涉及系统安全、文件系统作用域（fs_scope）、网络出站（net_scope）及执行特权。刚性约束绝对不可妥协，一旦违规直接阻断退出。
2. **柔性约束 (Soft/Formatting Bounds)**：涉及输出格式（JSON Schema 结构）、特定词汇屏蔽（forbidden_patterns）与排版规范。柔性约束允许触发**动态约束降级 (Dynamic Constraint Relaxation)**。

当动作被沙箱拦截时，系统会返回一个**标准拦截载荷 (Rejection Payload)**。如果重试达到阈值且属于柔性约束，则自动降级放宽 Logit Masking 限制，允许模型输出普通文本后通过外部正则/轻量级模型进行二次格式化（Post-processing），从而保全复杂任务下的推理智商。自进化执行循环算法如下：

```python
def schema_sandbox_execution_loop(input_task, S_t, manifest, retry_count=0):
    # 结合当前认知图式 K_t 生成候选动作
    action = generate_raw(input_task, S_t.K_t)
    
    # 验证输入与权限令牌授权 (CapabilityGrant)
    # 刚性边界 (Rigid Bounds) 校验，一旦违规立刻阻断且不可降级
    if not authorize_capability(action, S_t.Grant):
        rejection = build_rejection_payload("AUTHORIZATION_FAIL", S_t.Grant)
        return S_t, rejection  # 返回标准化拦截载荷 (Rejection Payload)
        
    validation_res = S_t.Phi_t(action)
    if validation_res.passed:  # 同化 (Assimilation)
        result = execute_action(action)
        
        # 记录 EvidenceRecord
        record_evidence(action, result, manifest.required_evidence)
        update_episodic_memory(action, result, success=True)
        return S_t, "Assimilation_Success"
    else:  # 顺应 (Accommodation)
        # 判断违规类型是否为柔性约束 (Soft/Formatting Bounds) 且达到重试上限
        if validation_res.is_soft and retry_count >= 3:
            # 动态约束降级 (Dynamic Constraint Relaxation)
            # 放宽 Logit Masking 限制，允许模型输出普通文本，由本地轻量级规则进行二次格式化
            relaxed_action = local_post_process(action, validation_res)
            if verify_soft_recovery(relaxed_action):
                result = execute_action(relaxed_action)
                record_evidence(relaxed_action, result, manifest.required_evidence)
                return S_t, "Relaxed_Success"
                
        # 顺应逻辑：返回标准 RejectionPayload，由大模型上下文自愈或触发沉淀
        rejection = build_rejection_payload(validation_res.error_type, validation_res.bounds)
        error_trace = rejection.to_context_string()
        
        # 沉淀算子 (CompactionOperator) 动态精炼语义修饰补丁
        # 在只读内核 Prompt(core) 基础上，采用 read-before-write 方式重写表观修饰层 Prompt(self)
        new_K = CompactionOperator.reorganize(
            S_t.K_t, (input_task, action), error_trace
        )
        
        # 重新编译图式沙箱（更新规则树、活动 Hooks 与工作分区）
        S_t_plus_1 = compile_new_sandbox(new_K)
        
        return schema_sandbox_execution_loop(
            input_task, S_t_plus_1, manifest, retry_count + 1
        )
```

### 3.5 设计目标与 KPI (Design Goals and KPIs)

从工程方法学角度看，图式沙箱的系统设计需要同时满足边界可证明、执行可复现、成本可预测与扩展可治理。为了定量评估系统性能与安全性，我们定义了以下十二个核心衡量指标（KPI）：

| 设计维度 | 衡量指标 (KPI) | 性能目标值 (SLO) | 测量方法 |
| :--- | :--- | :---: | :--- |
| **结构安全** | 输出契约语法违规率 | 0 | 对 JSON/Schema/Grammar 输出进行全量自动校验；基于 JSONSchemaBench 判定。 |
| **语义安全** | 关键字段语义违规率 | < 1% | 语义判定器与规则引擎检测，按对象级与字段级双重统计。 |
| **越权控制** | 非授权工具调用逃逸率 | 0 | 基于红队脚本与合成提示词注入套件进行回放测试。 |
| **凭证保护** | 密钥/PII外泄召回率 | > 99% | 在出站流量中植入 Canary Token、虚拟凭证与已知 PII 样本做拦截率测试。 |
| **隔离强度** | 跨分区非授权读写成功率 | 0 | 构造高低信任分区，模拟摘要回传、缓存污染与子代理继承。 |
| **性能开销** | 本地校验门控 p95 延时 | < 5 ms | 单机运行 10,000 次基准测试，拆分审计、策略求值与脱敏耗时。 |
| **端到端效率** | 普通任务总时延增幅 | < 15% | 与“无沙箱”对照，计算并分拆模型推理、工具执行与门控拦截的时间占比。 |
| **长效稳定性** | 规则压缩后任务成功率下降 | < 5% | 进行长链工作流 A/B 测试，比较压实前后与子代理分流前后的成功率。 |
| **可扩展性** | 单宿主最大并发分区数 | 视资源档位而定 | 在 8/16/32 vCPU 典型硬件环境下，分别测量 L1/L2/L3 的并发承载。 |
| **可审计性** | 关键动作证据链覆盖率 | 100% | 强制要求每个外部副作用动作都必须生成 `EvidenceRecord` 审计记录。 |
| **开发效率** | 新图式 Schema 首次落地时间 | < 5 人日 | 计算从定义输入契约到进入流水线回归测试所需的总工时。 |
| **运维效率** | 每次升级全量回归时间 | < 2 小时 | CI/CD 流水线中，结构、攻击、性能与许可四套回归套件并行执行。 |

---

## 4. 图式沙箱通用九层架构 (The Nine-Layer Architecture)

为了在工程中实现这一认知约束，我们将图式沙箱标准化为九层参考架构。表 II 展示了每一层所应对的失效模式、示例机制、在不同沙箱级别中的必要性以及现有技术的覆盖差距。

### 表 II：图式沙箱九层架构设计矩阵

| 架构层 | 应对的失效模式 | 示例机制与工业级映射 (v16 增强) | L1 必要性 | L2 必要性 | L3 必要性 | MVP实施优先级 | 现有技术的覆盖差距 (Coverage Difference) |
| :--- | :--- | :--- | :---: | :---: | :---: | :---: | :--- |
| **1. Domain Corpus** | 域外幻觉、事实性错误 | 本地 Markdown 语料包、文件级索引、向量存储 | 可选 | 可选 | 可选 | **Tier 3：高级扩展** | Outlines: 仅限文法生成；无检索层支持。 |
| **2. Task Ontology** | 意图混淆、任务越界 | Pydantic 校验、Skill 声明（YAML 格式指示符与工具集） | 强制 | 强制 | 强制 | **Tier 3：高级扩展** | MemGPT: 仅管理动态内存；无本体定义校验。 |
| **3. Input Contract** | 输入注入、参数错乱 | Zod 契约、前置脱敏门（PII 遮蔽）、PreToolUse 拦截 Hook | 强制 | 强制 | 强制 | **Tier 1：核心骨架** | SCG-MEM: 约束记忆结构；无输入参数过滤。 |
| **4. Router** | 意图分流偏差、多通道泄漏 | 语义分类器、基于优先级规则分流至 Workspace Partitions | 可选 | 强制 | 强制 | **Tier 3：高级扩展** | Tree of Thoughts: 侧重状态搜索；无分区路由。 |
| **5. Memory & Compaction** | 上下文溺水、漂移、历史消散 | **3层存储**（ ephemeral, memory.md 索引, CLAUDE.md 宪法）；**5层渐进式压实**（Budget/Snip/Micro/Collapse/Auto）；自愈 writes | 可选 | 可选 | 可选 | **Tier 2：增强鲁棒** | Voyager: 专注于代码技能生成；无知识 selector 与 compaction 机制。 |
| **6. Procedure Schema** | 步骤跳过、陷入死循环 | DAG 执行图、Transcripts 检查点、Sidechain transcripts | 强制 | 强制 | 强制 | **Tier 1：核心骨架** | SCG-MEM: 侧重记忆状态；无流程步骤校验。 |
| **7. Tool/API Grammar** | 工具格式崩溃、越权命令 | EBNF 语法约束、Logit 屏蔽（L0）、AST 安全树拦截器 | 可选 | 强制 | 强制 | **Tier 2：增强鲁棒** | Outlines: 侧重 Token 语法约束；无执行权限控制。 |
| **8. Boundary & Permission** | 越权执行、凭证窃取、逃逸 | **8层纵深安全网**（flags → killswitches → priority rules → yolo-LLM judge → blocklist → FS/AST check → dialog → bypass）；graduated perms | 可选 | 强制 | 强制 | **Tier 2：增强鲁棒** | 现有约束生成库：仅限于词法路径生成；无系统级执行防护。 |
| **9. Output Contract** | 语义泄露、格式幻觉 | 后置脱敏门、正则盾、PostToolUse 校验、自愈重试 | 强制 | 强制 | 强制 | **Tier 1：核心骨架** | Outlines: 仅校验输出语法；无生成后的语义敏感拦截过滤。 |

---

### 4.1 工作分区与多 API 编排 (Workspace Partitions)

在生产环境中，一个沙箱往往需要同时协调多种异构能力。因此，本架构确立了工作分区（Workspace Partition）作为沙箱内部逻辑隔离的受控执行单元。

每个工作分区在创建时被声明一个独立的 **隔离契约 (Isolation Contract)**：
* **数据输入子集与并发隔离**：每个分区仅能访问其绑定的 `memory.md` 局部子目录。为了防止多分区并发写入同一个共享文件导致竞态条件（Race Conditions）与文件锁冲突，每个分区在拉起时仅加载主存储的只读快照——**只读上下文快照 (Context Slice)**，而所有的写入与修改操作均隔离在各自专属的**私有写入缓冲区 (Write Buffer)**中，不直接操作全局存储。
* **工具与 API 文法限制**：限制可用的工具集和 EBNF 语法模板。
* **隔离模式**：
  * *Context Scoping*：在同一个模型调用中划分不同的上下文段，阻止未经脱敏的参数流出。
  * *Worktree Isolation*：通过生成临时 Git 工作区（Git Worktree Snapshot）隔离文件系统变动，子智能体执行完毕后仅返回 Summary。
  * *Remote Isolation*：在高安全级分区使用物理上完全隔离的沙箱容器（如 Docker / WASM 容器）。

---

## 5. 典型运行时设计模式与工程实现规范

图式沙箱的工程落地依赖于轻量且安全的底层设计，以避免传统沙箱虚拟化的高延迟瓶颈。

### 5.1 3-层存储与并发自愈更新 (Layer 5/6 Memory Implementation)
图式沙箱采用基于本地文件系统的 3 层存储管理，并放弃了传统的硬编码并发文件直写方式，以防止多智能体并发下的文件锁冲突与状态覆盖。我们引入 **无冲突复制数据类型 (Conflict-Free Replicated Data Types, CRDTs)**（如 LWW-Register 或 Delta-State CRDT）以及本地虚拟文件系统（SQLite-backed VFS）来存储和同步状态变动：
1. **ephemeral 上下文**：用于存储单次 ReAct 循环中的短期推理痕迹与单次 Token 缓存。
2. **`memory.md` 指针与数据库索引**：通过虚拟文件系统（VFS）映射，存储该项目或该智能体的经验、决策路径与核心事实。
3. **`CLAUDE.md`（或 `AGENTS.md`）项目宪法**：定义智能体的基本行为约束、安全限制与开发规范，作为只读内核。

当工作分区生命周期结束并拆除（Teardown）时，系统并不直接覆盖全局存储，而是由主路由器启动**图式合并机制 (Schema Merge)**：由主路由通过 `CompactionOperator` 收集分区的 `Write Buffer`，执行类似 Git 三路合并（Three-way Merge）或 CRDT 的 State-Merge 算法自动消除冲突，最终将提炼的经验追加写入到主 `memory.md` 中，防止历史经验在时程拉长后因遗忘而发生认知漂移，在工程上确保了多分区协作的确定性。

### 5.2 5-层渐进式上下文压实管道 (Progressive Compaction Pipeline)
为了防止上下文窗口溺水导致注意力漂移，装配架内置了 cache 友好的 5 层压实算法，在读取隐空间投影时以最低成本逐级触发：
1. **Budget Check (预算层)**：单次消息输入 Token 级剪枝，超过阈值强制截断非核心段。
2. **Context Snip (剪裁层)**：依据滑窗策略丢弃历史的非核心对话。
3. **Micro-compaction (微压缩层)**：提取结构化标记，保持 KV Cache 对齐，不破坏已生成的 Logit 缓存。
4. **Semantic Collapse (语义折叠层)**：将多条零散规则通过 AST 或正则表达式装配整合为单条规则表示，利用简明摘要（Summary）替代原始长文本段。
5. **Auto-Summary (自适应总结层)**：当系统状态接近上下文瓶颈时，唤醒后台模型对冗余记录进行单次语义压缩，并在输出中为保留片段标注语义标记，同时设定断路器，防止递归生成产生的语义畸变。

### 5.3 8-层纵深防御安全链 (Layer 8 Permission Implementation)
当智能体在 L2/L3 级别运行时，面临着命令注入与越权的真实物理威胁。为此，本架构设计了 8 层纵深防御安全链：
```text
[1. Feature Flags] → [2. Killswitches] → [3. Priority Rules] → [4. YOLO-LLM Classifier]
                                                                        ↓
[8. Bypass Valve] ← [7. Trust Dialog] ← [6. AST/FS Validators] ← [5. Pattern Blocklist]
```
1. **编译与功能开关 (Feature Flags)**：静态编译时排除不安全的代码接口。
2. **服务器全局断路器 (Server Killswitches)**：支持云端一键切断本地智能体写权限。
3. **优先级规则引擎 (Priority Rules)**：通过 8+ 类规则配置文件定义严格的 deny-first 访问控制。
4. **YOLO-LLM 分类器 (Transcript Classifier)**：独立的超轻量判别器（如 `yoloClassifier.ts`），利用 Fail-Safe 策略快速审阅即将派发至 Shell 的命令语义。
5. **模式黑名单 (Pattern Blocklist)**：阻断已知的危险命令正则（如 `rm -rf /`）。
6. **文件系统与 AST 审查 (AST/FS Validators)**：利用 tree-sitter 对提议执行的 Bash 或代码进行抽象语法树解析，进行符号级审查。
7. **用户交互确认 (Trust Dialog)**：弹出交互终端供人类复核。
8. **免审核旁路开关 (Bypass Valve)**：仅在特定高信任分区中，当动作通过了前 7 层验证时方可触发自动免密执行。

当纵深防御安全链中的任意一层拦截动作时（如 AST 校验失败、命令正则匹配拦截或文件系统越权），沙箱控制面绝不只返回干瘪的“Access Denied”或“错误码”，以防大模型陷入“盲目重试-再次拦截”的自我死锁死循环（Thrashing Loop）。系统强制封装并抛出标准化自愈载荷——**Rejection Payload**，明确告知模型当前违规的类型（`violationType`）、当前分区真正的刚性权限边界范围（`allowedBoundaries`），并提供明确的降级指引（`suggestedFallback`，如提示调用 `AskHuman` 权限申请人工批准），使沙箱成为一个“带路标的护栏”，指导神经（大模型）与符号（硬性限制）进行自愈式的良性交互。

---

## 6. L0-L3 分层混合式运行时框架

本架构的核心主张是：**单靠提示词过滤是不安全的，但如果把所有轻量级任务都直接推向重型微虚拟机（microVM），高昂的冷启动延迟与资源开销将使系统失去商业可行性**。为此，我们确立了分层混合式执行模型：

```text
               [ 用户请求传入 / Router 智能分流 ]
                              │
       ┌──────────────────────┼──────────────────────┐
       ▼                      ▼                      ▼
  [ 方案 A (L1/L2) ]     [ 方案 B (L2) ]        [ 方案 C (L3) ]
  Deno + WASI 内核       gVisor 容器隔离        Firecracker VM
  · 默认拒绝权限模型     · 隔离复杂工具链       · 承接高危未知代码
  · 本地 CPU Logit 屏蔽   · 拦截网络 Egress      · 租户数据硬性物理阻断
  · 本地校验门控: p50 ≈ 1.5 μs · 容器文件范围挂载     · 独立 rootfs 与网络 proxy
```

### 6.1 方案 A：能力安全优先的本地优先内核 (Default Base)
本方案适用于绝大多数常规的 L1/L2 级别任务。控制面与轻量级执行分区采用 **Deno + WASI/Wasmtime** 构建：
* **Deno 默认拒绝权限模型**：系统默认拦截所有文件读取、子进程拉起、环境变量和网络请求。仅当 Schema 显式授权时，才细粒度开放作用域（例如仅允许写入本地特定的 `./workspace/src` 目录）。
* **WASI 能力边界**：将插件、第三方解析器编译为 WebAssembly 字节码，运行在 Wasmtime 沙箱中，以进程内的安全屏障彻底物理阻断内存溢出或逃逸。

### 6.2 方案 B：微内核控制面 + 进程容器分区 (Production Workloads)
本方案主要运行中等风险的外部连接，如 Evidence Fetch 爬虫分区、Browser 自动化以及复杂的 CI/CD 构建链工具：
* 采用 **gVisor** 或受控的 Docker 容器 Worker。OpenHands 的官方文档表明，如果在没有容器隔离的物理宿主机上执行 Agent，系统极易发生严重泄漏。
* 沙箱控制面与 Worker 之间采用无共享内存的 JSON-RPC 信道通信，容器仅挂载极有限的目录（如 `PROJECTS_PATH`），并强制绑定 egress allowlist 锁定数据外泄通道。

### 6.3 方案 C：面向高危代码的 microVM 强隔离 (High-Risk/Untrusted Tasks)
当智能体必须执行用户上传的任意未知代码、运行外部拉取的第三方 Shell 脚本或处理跨租户的多维数据任务时，执行端将自动被路由至方案 C：
* 唤醒 **Firecracker microVM**，为当前任务分配隔离的独立 rootfs 镜像，以及短时有效的 credentials proxy。
* 物理封锁 Linux Syscalls，将执行副作用与宿主系统彻底隔离。任务执行完毕后 microVM 立即被销毁（RTO < 5ms）。

三种方案的综合对比如下：

| 方案 | 隔离机制 | 典型任务 | 优势 | 主要不足 | 推荐结论 |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **方案 A：能力安全本地内核** | Deno/WASI + 约束解码 + 语义验证 | 结构化生成、数据处理、低风险工具链 | 延迟极低 (微秒级)，本地化，开发维护极简 | 对未知二进制或任意 shell 防护较弱 | **默认基线**。常规或高频交互优先在此执行。 |
| **方案 B：进程与容器分区** | 宿主控制面 + gVisor/容器 Worker | 网页抓取、浏览器自动化、数据库读写、CI 构建 | 隔离分区清晰，安全边界易扩展，开销适中 | 运维复杂度上升，存在容器级越权漏洞可能 | **生产主力**。方案 A 成行后可作为企业级写操作基座。 |
| **方案 C：微虚拟机强隔离** | Firecracker microVM | 未知代码执行、第三方未知插件运行、跨租户敏感计算 | 硬件级物理隔离，启动迅速 (毫秒级)，安全性最强 | 资源消耗较大，冷启动延时大于容器 | **特批路径**。仅分配给被标识为 L3 的高危动作。 |

---

## 7. 威胁模型、数据治理与许可证合规性 (Threat Model & Compliance)

### 7.1 八大混合威胁矩阵 (In-Scope Threats)
图式沙箱必须能够防御针对软件生命周期、供应链以及上下文的复合攻击。我们将系统面临的威胁模型整理为以下八类核心安全控制面：

| 威胁类别 | 典型攻击路径 | 主要安全后果 | 关键防御控制面 |
| :--- | :--- | :--- | :--- |
| **发布工件泄漏 (Release Leakage)** | source map、调试包、CI工件、公开对象存储误发布 | 源码、规则、内部接口与凭证暴露 (Claude Code npm 泄漏属于此类) | 发布白名单、工件静态扫描、SBOM、签名与双人审阅门禁 |
| **上下文注入 (Context Injection)** | README、Issue、PR 评论、网页、日志中注入恶意指令 | 越权执行、数据窃取、动作重定向与执行流劫持 | 输入分层隔离、来源标记、PreToolUse 拦截 Hook、上下文净化 |
| **工具越权执行 (Privilege Escalation)** | Shell 命令拼接、目录穿越参数、非授权 API 调用 | 宿主系统完整性破坏、文件篡改与远程恶意副作用 | deny-by-default 能力模型、策略引擎、Ask/Allow/Deny 门控 |
| **结构化输出失真 (Output Distortion)** | 结构合法但语义错误的值；工具参数含注入危险值 | 下游业务逻辑误执行、隐蔽数据腐败与状态崩溃 | 约束解码强制语法 + 语义验证 + 业务验证三层校验 |
| **工作分区跨区泄漏 (Cross-Partition)** | 子代理调用、缓存污染、会话持久化数据溢出 | 低信任分区读取到高密分区敏感上下文 | 独立上下文隔离、只回传摘要、最小化共享内存 |
| **供应链与许可证风险 (Licensing Risks)** | 恶意依赖包、被投毒的插件、病毒式许可证传染 | 恶意代码执行、商业闭源系统许可证传染污染 | 依赖版本锁定、能力注册表审计、Library与子进程物理隔离 |
| **长上下文注意力漂移 (Context Drift)** | 压实失真、早期规则丢失、多轮会话注意力散逸 | 智能体行为偏移、遗忘指令、重复犯错 | 路由化规则加载、路径锁定规则、自动规则审计与焦点压缩 |
| **拒绝服务与成本爆炸 (DoS/Cost)** | 恶意超大文件、死循环工具调用、并发放大请求 | 算力信用额度耗尽、宿主系统不可用、账单成本超支 | 单次/全局配额、流控节流、超时熔断、thrashing 异常检测 |

### 7.2 许可证隔离合规策略 (Licensing Boundaries)
在 Capability Registry (能力注册表) 中，我们建立硬性许可证审计机制：
* 归属于 **MIT / Apache-2.0 / BSD** 的工具引擎允许以 Library 形式直接在进程内链接或调用。
* 归属于 **GPL-3.0 / AGPL** 的开源工具（如特定的测试器/依赖库），**禁止在进程内直接链接**。沙箱强制要求此类工具必须以独立子进程运行，或运行在网络隔离的方案 B 容器中，以 JSON-RPC 作为严格的 API 契约层边界，彻底切断 copyleft 的许可证传染通路，保护商业资产安全。

---

## 8. 图式互操作协议 (SIP)：面向图式沙箱的能力挂载与治理协议 (SIP: A Capability Mounting and Governance Protocol)

图式互操作协议（SIP）是图式沙箱体系中最关键的互操作与治理契约。正如九层架构回答了“一个图式沙箱怎么构建”，SIP 则回答了“不同图式沙箱如何被发现、挂载、调用、组合、验证、交易和长期演化”。没有 SIP，图式沙箱仅是一个强大的局部工程方法论；有了 SIP，它才有机会演变为开放标准、能力市场与 Agent IP 生态底座。

学术上，**SIP 是一种清单级协议，用于让图式沙箱在不同智能体运行时之间被发现、验证、挂载、调用、隔离、审计、升级与交易（SIP is a manifest-level protocol that enables Schema Sandboxes to be discovered, verified, mounted, invoked, isolated, audited, upgraded, and commercially exchanged across heterogeneous agent runtimes）。**

为了平衡技术开源与商业壁垒，SIP 确立了清晰的“三层边界”治理模型：
1. **公开层**：公开 SIP 协议字段规范、Manifest 配置文件格式、标准错误码定义与基础验证逻辑，建立统一开源标准。
2. **半公开层**：开源 SchemaBox SDK、标准能力适配接口、基础示例沙箱包（Sandbox Packs）与开发者文档，方便宿主集成。
3. **私有层**：具体垂直行业（如法律、金融、GEO 诊断）高价值 Sandbox Packs 的核心提示词链、路由过滤规则、专有诊断数据处理工作流以及多租户评分权重细节，均保留为私有资产。

通过这种“开放标准 + 私有能力包 + 平台市场”的结构，传统的隔离沙箱被赋予了能力资产属性，实现了受控挂载与商业化流通的闭环。

### 8.1 协议架构与七大 Manifest 清单模块


SIP 协议的核心体现为 `manifest.json` 能力清单，任何支持 SIP 的智能体客户端必须能够解析并执行清单中定义的以下七大核心模块：

1. **元数据模块 (metadata)**：声明沙箱 ID、SemVer 版本号、开发者标识、许可证类型，以及用于完整性校验的 Ed25519 密码学签名。
2. **输入契约模块 (input_contract)**：锁定沙箱接收的参数格式与脱敏规则，阻断针对 ReAct 循环的前置注入。
3. **输出契约模块 (output_contract)**：锁定沙箱的输出 Schema、敏感正则黑名单、最大允许的信息熵吞吐阈值，防止高密数据泄露。
4. **权限范围模块 (permission_scope)**：对应 `CapabilityGrant`，锁定文件系统范围（fs_scope）、网络域名（net_scope）和可拉起工具（tool_scope）。
5. **工作区分区模块 (workspace_partitions)**：规定沙箱内部包含的逻辑子分区、隔离级别（Option A/B/C）与专有内存空间。
6. **审计验证模块 (validation)**：声明该图式运行过程中强制留存的加密哈希，用于生成不可篡改的 `EvidenceRecord`。
7. **错误码模块 (error_codes)**：规范跨分区、跨沙箱通信时的故障返回码，便于异常路由自愈。

一个标准的 SIP v1.1 互操作 Manifest 文件示例如下：

```json
{
  "sip_version": "1.1.0",
  "sandbox_id": "brand_shuttle_geo_audit",
  "name": "Brand Shuttle GEO Audit Sandbox",
  "version": "1.0.0",
  "capability_type": "diagnosis.report",
  "metadata": {
    "author": "LIU TENGJIAO",
    "license": "Apache-2.0",
    "description": "Bilingual SEO eligibility and structured FAQ extraction sandbox",
    "signature": "ed25519:7e8bcfc09fa65123d..."
  },
  "trust": {
    "signature_algorithm": "ed25519",
    "canonicalization": "jcs-rfc8785",
    "publisher_key_id": "did:psi:publisher:ltj-2026",
    "revocation_url": "https://registry.psi.run/revoke/brand_shuttle_geo_audit",
    "signed_at": "2026-06-22T12:00:00Z"
  },
  "input_contract": {
    "schema_ref": "file:///schemas/geo_input_schema.json",
    "sanitize_input": true
  },
  "output_contract": {
    "schema_ref": "file:///schemas/geo_output_schema.json",
    "forbidden_patterns": ["the best", "guaranteed ranking"],
    "entropy_threshold": 4.5
  },
  "permission_scope": {
    "fs_scope": ["./workspace/reports"],
    "net_scope": ["*.target.com:443"],
    "tool_scope": ["Crawlee:scrape", "Linkinator:check"]
  },
  "workspace_partitions": [
    {
      "partition_id": "scraper_partition",
      "isolation_level": "worktree",
      "memory_slice_scope": ["./workspace/temp_scraped"]
    }
  ],
  "validation": {
    "required_evidence": ["input_hash", "tool_args_hash", "output_hash"]
  },
  "error_codes": {
    "SIP_ERR_VERSION_MISMATCH": "0x01",
    "SIP_ERR_SIGNATURE_INVALID": "0x02",
    "SIP_ERR_INPUT_VIOLATION": "0x03",
    "SIP_ERR_SCOPE_LOCKED": "0x04",
    "SIP_ERR_LEAK_DETECTED": "0x05",
    "SIP_ERR_COMPACTION_VIOLATION": "0x08"
  }
}
```

> **实施指引**：SIP v1.1 的完整 JSON Schema 规范定义已发布于本架构的 `/schemas/sip_v1.1.schema.json` 中。开发者在解析或构建 `manifest.json` 时，推荐直接使用主流 JSON Schema 工具链（如 JavaScript 环境下的 `ajv` 或 Python 的 `pydantic`）自动生成强类型类并进行解析校验，无需手动编写字段处理逻辑。

为了让拦截响应能自愈指导智能体执行，SIP 标准规范了**拦截载荷 Schema (SIP Rejection Payload Schema)**。当沙箱拦截动作并返回错误码（如 `SIP_ERR_SCOPE_LOCKED`）时，必须附带以下格式的 JSON 载荷作为上下文引导：

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "title": "SIPRejectionPayload",
  "type": "object",
  "required": ["violation_type", "allowed_boundaries", "suggested_fallback"],
  "properties": {
    "violation_type": {
      "type": "string",
      "enum": [
          "AUTHORIZATION_FAIL",
          "OUT_OF_BOUNDS_READ",
          "OUT_OF_BOUNDS_WRITE",
          "SCHEMA_FORMAT_ERROR",
          "ENTROPY_VIOLATION",
          "LEAK_DETECTED"
        ]
    },
    "allowed_boundaries": {
      "type": "object",
      "properties": {
        "fs_scope": { "type": "array", "items": { "type": "string" } },
        "net_scope": { "type": "array", "items": { "type": "string" } },
        "tool_scope": { "type": "array", "items": { "type": "string" } },
        "valid_schema_ref": { "type": "string" }
      }
    },
    "suggested_fallback": {
      "type": "object",
      "required": ["action", "description"],
      "properties": {
        "action": {
          "type": "string",
          "enum": [
          "AskHuman",
          "DegradeToPlainText",
          "UseFallbackSchema",
          "RetryWithCompaction"
        ]
        },
        "description": { "type": "string" }
      }
    }
  }
}
```

下表梳理了七大 Manifest 模块与第 3.1 节定义的五大运行时契约接口之间的映射关系，有助于工程实现时的逻辑对照：

| SIP Manifest 模块 | 运行时契约接口 (TypeScript) | 主要拦截点 / 职责 |
| :--- | :--- | :--- |
| `metadata` | `SchemaManifest` | 握手与 Ed25519 签名合法性校验 |
| `input_contract` | `SchemaManifest.inputSchemaRef` + 前置拦截器 | 输入 Schema 的前置红线过滤与格式对齐 |
| `output_contract` | `OutputContract` | 输出 Schema 的后置屏蔽、低熵检测与正则过滤 |
| `permission_scope` | `CapabilityGrant` | 运行时系统权限控制（文件系统、网络、命令行工具） |
| `workspace_partitions` | `PartitionPolicy` | 隔离分区的动态划定、工作流目录初始化与 Teardown 回收 |
| `validation` | `EvidenceRecord` | 链上/本地只增审计日志生成 |
| `error_codes` | `SIPError` 枚举值 (标准化错误响应) | 沙箱故障向宿主环境的标准化抛出 |

### 8.2 挂载与调用生命周期 (Mounting and Invocation Lifecycle)

在运行时，SIP 沙箱的挂载与调用遵循六阶段生命周期：
1. **握手与版本协商 (Handshake)**：主 Agent 运行时读取 Manifest，比对 `sip_version` 是否兼容。
2. **完整性校验 (Verify)**：利用发布者的公钥验证 Manifest 的 Ed25519 签名，确保策略文件未被中途篡改或投毒。
3. **隔离分区初始化 (Initialize)**：根据 `workspace_partitions` 中的定义，在 Deno 中为沙箱分配逻辑 Workspace，或拉起对应的 Docker 容器（方案 B）。
4. **运行时边界锁定 (Enforce)**：激活 Logit 屏蔽（Layer 7）与文件系统挂载限制（Layer 8），强制工具调用遵循 `CapabilityGrant` 许可边界。
5. **后置脱敏与审计 (Post-Audit)**：拦截输出，校验 `OutputContract`，生成 `EvidenceRecord` 写入本地 append-only 日志。
6. **回收与资源释放 (Teardown)**：销毁临时 Git Worktree 或 microVM，释放租约，退出执行。

### 8.3 CI/CD 发布安全门禁 (Release Gates)

为了防止编译开发工件泄漏（如 source map 泄漏），发布流水线强制部署以下静态与动态门禁：
* **发布白名单扫描 (Artifact Scan)**：流水线强制阻断一切包含 `.map`、源码归档、内部调试日志与未签名 Manifest 的包构建。
* **SBOM 与 License 校验 (SBOM Check)**：利用 Capability Registry 自动扫描依赖链中是否有 GPL-3.0 许可证，强行隔离 GPL 代码至独立子进程运行，限制其对商业系统的污染。
* **密码学双人联签 (Sign)**：由两名独立审核员用 Ed25519 私钥联签 Manifest，宿主在 SIP 挂载时强制比对签名。

---

### 8.4 SIP 合规等级定义 (SIP Conformance Profiles)

为了支持沙箱在不同复杂度环境下的平滑接入，我们将协议的合规实现划分为三个递进等级：

| 合规等级 | 必须支持的模块 / 动作 | 典型应用场景 |
| :--- | :--- | :--- |
| **SIP-Core** | `metadata`, `input_contract`, `output_contract`, `permission_scope`, `error_codes` | 开源项目及内部工具链，仅需基础的结构化契约与局部权限限制 |
| **SIP-Secure** | 继承 SIP-Core，且包含 `validation` (EvidenceRecord)、`CapabilityGrant` 强校验、工作区生命周期 Teardown、Ed25519 签名验证 | 生产级智能体运行时环境，要求对每一次工具调用进行硬性拦截与不可篡改审计 |
| **SIP-Market** | 继承 SIP-Secure，且包含 `trust` 块验证（公钥发现、撤销 URL 检测、规范化 JCS 联签）、版权声明、计费元数据声明、Capability Registry 自动入册 | 跨厂商、跨地域的商业级智能体能力市场，需要完整的资产确权、按需计费及在线证书吊销功能 |

分级合规机制降低了开发者起步的门槛。普通的社区开源组件只需声明 SIP-Core 合规即可快速发布，而进入商业结算与金融级审计环境则要求逐步通过 SIP-Secure 和 SIP-Market 门禁。

---

## 9. 案例研究：品牌出海 GEO 沙箱实例 (Brand Shuttle GEO)

我们通过 **Brand Shuttle GEO（品牌出海 GEO 图式沙箱）** 参考实现展示混合架构的应用：
* 在 L1 级别下，系统执行 **AI 搜索准入检测（AI Search eligibility check）**，使用默认拒绝一切外部网络权限的 Deno 运行时。
* 当需要调用 Crawlee 进行 sitemap 抓取时，Router 将该子任务路由至 **方案 B（gVisor 隔离 worker）**。该 worker 仅被授予访问 target.com 主机之 443 端口的 CapabilityGrant，且生命周期到期即销毁。
* 从 gVisor 容器回传的抓取日志自动在 pre-gate 经过 kingfisher 敏感信息扫描，将可能泄露的个人邮箱和联系方式过滤，方可被主推理模型分区读取。
* 最终，OutputContract 验证 FAQ 与 Entity Schema 是否含有 "the best" 违规 token，验证成功后生成 EvidenceRecord 并将事实晶体写入 memory.md 索引。

---

## 10. 量化评估与五回归实验矩阵 (Empirical Evaluation)

我们在本地图式沙箱（包含输入契约校验、SQL 参数校验、前置隐私脱敏门、后置密钥泄漏门及输出契约校验）上运行了 10,000 次基准测试，并建立了系统级的五回归实验矩阵。

### 10.1 本地校验延迟与并发 SLO
* **平均延迟**：1.580 微秒 (us)
* **p50 (中位数) 延迟**：1.500 微秒 (us)
* **p95 延迟**：1.700 微秒 (us)
* **p99 延迟**：2.100 微秒 (us)
我们制定的性能服务水平目标（SLO）要求本地 pre-gate 与 post-gate 的校验时延在 8-vCPU 典型硬件环境下，p95 不得超过 **5 毫秒 (ms)**。

### 10.2 五回归测试实验矩阵 (Five-Regression Matrix)
为了系统化评估 Schema Sandbox 对越权与泄露的抵御能力，我们设计了五种等效替身测试协议（Test Protocols），任何发布候选均需通过这些回归拦截审计：
1. **泄漏复现替身测试 (Mock Release Leakage)**：在自建的 mock 镜像仓库流水线中，故意混入 source map、内部未签名 manifest 配置文件及调试 bundle，测试 CI/CD 发布门禁是否能在 100% 情况下触发阻断并生成完整的 EvidenceRecord。
2. **JAW 工作流劫持测试 (YOLO/JAW Injection)**：在输入 README、模拟 issue 评论和 shell 执行历史中注入提权控制指令，测试 YOLO-LLM 分类器与 AST/FS 校验器是否能 100% 阻断注入，将非授权动作逃逸率降为 **0**。
3. **结构化输出语义回归 (JSONSchemaBench)**：利用 JSONSchemaBench 测试集，测试在受到 logit Trie 树与输出验证门拦截时，是否能将格式正确但值违规（semantic errors）的输出拦截率维持在 **99%** 以上。
4. **长上下文规则压缩回归 (Context Compaction)**：在 100-1000 步的长周期会话中，测试渐进式压实算法执行前后，智能体规则的覆盖率及任务成功率下降率。要求由于压缩引起的任务成功率下降不得超过 **5%**。
5. **外泄隔离诱捕测试 (Canary Egress)**：在 Workspace Partition A（高密区）中故意放置 Canary Credentials（伪密码凭证），同时在 Partition B（低密区）的外部源中输入恶意漏洞指令，测试在模型多次调用和拼接输出中，Canary 凭证的外泄捕获率是否达到 **100%**。

五回归实验矩阵的场景与性能要求总结如下：

| 实验类型 | 场景设计 | 主要指标 | 预期目标值 (SLO) |
| :--- | :--- | :--- | :---: |
| **发布泄漏替身实验** | 在内部测试发布包中故意混入 source map、源码压缩归档与调试 bundle | 发布阻断率、 EvidenceRecord 生成完整性 | 发布包阻断率 **100%** |
| **JAW注入攻击实验** | 在输入 README、 issue 评论和 shell 历史中注入 YOLO/JAW 劫持提权指令 | 非授权命令逃逸率、策略判定漏判率 | 动作逃逸率 **0**，高危命令阻断率 **100%** |
| **结构化输出语义实验** | 选取 JSONSchemaBench 测试子集及系统自定义的 schema 图式 | 语法违规率、关键字段语义违规率 | 语法违规率 **0**，语义错误率 `< 1%` |
| **长时程上下文实验** | 执行 100 至 1000 步的长链任务，对比开启与关闭规则压实/子代理分流 | 任务成功率、 Token 降幅、压缩后认知漂移率 | 压缩后成功率下降 `< 5%` |
| **隔离/外泄诱捕实验** | 在 L2 高密分区放置 Canary 凭证，在低密分区输入恶意提取漏洞指令 | Canary 泄露率、跨分区非授权读写成功率 | 凭证外泄漏网率 **0**，读写逃逸率 **0** |
| **并发与吞吐量实验** | 在单宿主上并发调度多个租户的 L1/L2/L3 分区任务 | 平均吞吐量、 CPU与内存占用率、 p95 时延 | p95 Gate 时延 `< 5 ms` |

---

## 11. 实施的限制、权衡与已知的工程税 (Implementation Limits, Trade-offs, and Engineering Taxes)

任何前沿架构的落地，都在安全、性能与复杂性之间进行了取舍。为了让实施者对系统拥有底气，本节详述该架构在落地时的工程局限与我们建议的规避路径：

1. **约束税 (Constraint Tax)**：
   * *局限性*：强行约束 Logit 采样虽然消除了语法错乱，但也剥夺了模型自我纠偏的推理平滑性（即语法-语义脱节），可能压制模型脑力表现。
   * *规避路径*：建议在落地时将约束细分为刚性与柔性。如第 3.4 节所述，如果柔性格式约束连续失败 3 次，系统应启动 **动态约束降级 (Dynamic Constraint Relaxation)** 允许输出自由普通文本，然后结合外部正则、Schema 清洗库（如 `dirtyjson`）或本地超轻量模型在沙箱外进行二次纠偏（Post-processing），保全底座大模型的最高逻辑智商。
2. **规则编写税 (Rule Authoring Tax)**：
   * *局限性*：为每个沙箱定义完整的 `manifest.json`、编写 OPA/Rego 声明式安全规则以及校验 Schema 需要极高的人工和领域专家成本。
   * *规避路径*：在项目启动的 MVP 阶段，实施者完全可以**放缓接入复杂的 OPA 策略引擎**，转而采用纯硬编码的 Python/TypeScript 本地验证函数作为 Pre-Gate 与 Post-Gate 校验层。待业务流跑通后，再逐步将规则解耦重构至统一的 OPA/Rego 配置文件中。
3. **上游凭证泄露**：
   * *局限性*：沙箱虽然阻断了本地执行面逃逸，但若上游的 Host Client 密钥本身被盗，沙箱无法越权更正其凭证。
   * *规避路径*：控制面与执行面采用物理网络隧道（Proxy Tunneling），对每个 Workspace Partition 动态分发一次性凭证（Ephemeral Tokens），并将租约锁定在分钟级。

未来工作将侧重于研究“从智能体执行轨迹中自动合成 Schema 与 Hook 规则”的无监督合成技术，并探究 SIP、MCP 与智能体对智能体（Agent-to-Agent）协议的互通桥梁。

---

## 12. 实施路线与工程方法学 (Implementation Roadmap and Engineering Methodology)

为将图式沙箱（Schema Sandbox）从体系结构模型转化为工业可落地的产品线，我们设计了四阶段的工程实施路线，每个里程碑均绑定特定级别的安全回归门槛：

### 12.1 工程实施阶段规划

| 阶段 | 周期建议 | 关键交付物 | 核心工程活动 | 团队资源配置 |
| :--- | :---: | :--- | :--- | :--- |
| **内核原型 (Kernel)** | 3-4 周 | 最小可用 `SchemaManifest` 控制面内核、拦截 Gate 逻辑、 Evidence 证据落盘 | 完成 Deno 本地轻量内核 (方案 A)；接入 Outlines/Guidance 约束解码；实现基础 Pre-Gate 与 Post-Gate。 | 1名系统开发、1名安全工程师 |
| **分区执行 (Partition)** | 4-6 周 | `CapabilityGrant` 动态分配、分区调度、 Deno/WASI 执行 Worker 实例 | 引入 WASI 沙箱隔离；将文件系统读写、网络 IO 分流至受限分区运行。 | 1名系统开发、1名测试工程师 |
| **生产强化 (Production)** | 4-6 周 | gVisor 容器 Worker、 CI/CD 发布门禁、密钥去中心化代理、审计面板 | 接入 OPA/Rego 策略引擎；实现 source map 与未签名工件阻断；部署 SBOM 与 License 静态扫描。 | 1名安全开发、1名 SRE、1名 DevOps |
| **高危隔离 (VM Isolation)** | 4-8 周 | Firecracker 强隔离 VM 执行器、特批安全升级路由 | 为方案 C 引入 microVM 动态挂载机制；建立多租户账单与算力成本分配模型。 | 1名虚拟化专家、1名安全工程师 |

### 12.2 生态与产品落地路径

为了使协议不流于“纸面标准”，SIP 在产品落地与生态推行上采用实用主义的四阶段渐进式演进策略：
1. **第一阶段 (SIP v0.1 / Core 验证)**：围绕 Brand Shuttle GEO 业务流设计并验证极简的 `manifest.json` 核心字段（Metadata、Input/Output Contract、Permission Scope 等），实现本地 gate 规则解析与运行时前置/后置拦截的 MVP 验证。
2. **第二阶段 (能力沙箱包标准化)**：逐步将 Brand Shuttle GEO 诊断、事实水晶 (Fact Crystal)、AI 引用诊断 (AI Citation Diagnosis) 以及社媒 SEO 改写 (Social SEO Rewriting) 4 个垂直业务功能拆解封装为 4 个标准的 SIP 沙箱能力包 (Sandbox Packs)，实现多沙箱的本地挂载与隔离。
3. **第三阶段 (SchemaBox SDK 开源)**：提炼通用的沙箱载入与挂载逻辑，开源 SchemaBox SDK。该 SDK 提供标准库支持 `load_manifest`、`validate_input`、`invoke`、`validate_output` 以及 `mount_tool` 等 API，为主流 Agent 框架（如 LangChain、AutoGPT、MCP 宿主）提供标准化对接层。
4. **第四阶段 (能力市场与生态闭环)**：在 psi.run 平台上线能力流通与发现注册表（Capability Registry），支持第三方开发者发布、签名验证与按需调用/计费 SIP 沙箱，构建完整的 Agent IP 认知能力生态圈。

### 12.3 开发者体验 (DX) 与标准 CLI 工具链规范

为将图式沙箱（Schema Sandbox）体系结构规范转化为易于被开发者接受的生产力工具，SIP 协议族规范了标准命令行工具链 **`sip-cli`**。工具链旨在提供流畅的“脚手架初始化-合规静态审计-可信打包-本地离线Mock验证”开发生命周期，其核心规范命令定义如下：
1. **`sip init` (脚手架初始化)**：在当前工作区动态生成一个符合规范的 SIP 能力包模板。自动生成标准的 `manifest.json` 清单文件架构、用于约束输入与输出契约的 JSON Schema 文件空模板（`schemas/input_schema.json`、`schemas/output_schema.json`），以及基于规则引擎的策略声明文件骨架。
2. **`sip audit` (静态合规与特权审计)**：对本地的沙箱定义包进行离线静态漏洞分析与合规性审查。验证 `manifest.json` 中声明的 `permission_scope` 依赖，静态扫描是否存在非法的文件提权目录（如映射到 `/` 或系统根目录）、不安全的网络 Host 通配符、未受信任的第三方外部依赖，以及可能被恶意 Prompt 注入利用的 API 端点。
3. **`sip pack` (签名打包与可信分发)**：将所有的 Schema 描述文件、策略配置与逻辑代码静态打包为符合 SIP 标准的可发布包文件（`.sip`）。调取本地/硬密钥根（HSM / KMS），使用 Ed25519 私钥为打包生成的工件计算 SHA-256 摘要值，并自动将数字签名追加写入 `manifest.json` 的 `metadata.signature` 字段中，为能力包注入“密码学可信身份根”。
4. **`sip run --dry` (纯符号 Mock 校验与干路运行)**：在完全不调用任何外部大模型 API 的情况下，在本地 Mock 环境纯符号化地跑通沙箱验证管线。支持构造模拟的输入 payload，验证 Pre-Gate（输入契约门控）的参数过滤能力，以及 Mock 模型生成输出时 Post-Gate（输出契约门控）对边界拦截、Recreation 降级规则的正确解释度，降本增效地排除设计缺陷。

### 12.4 技术栈与核心选型

系统采用“控制平面偏静态、执行平面多语言”的混合架构选型：
1. **控制面内核**：采用 **Rust/Go** 编写，负责高并发的 manifest 验证、证据签名、策略检索与容器调度，确保本地 gate 验证在微秒级完成。
2. **策略引擎**：采用 **OPA/Rego**，实现“策略即代码 (Policy as Code)”的动态求值与规则的版本化回滚。
3. **约束解码**：采用 **Outlines/Guidance** 进行 Logit Trie 树对齐拦截。
4. **轻量执行器**：采用 **Deno** 结合 **WASI/Wasmtime** 实现无 VM 的逻辑级别能力锁定。
5. **容器与虚拟机**：采用 **gVisor** 拦截容器侧系统调用逃逸；高危代码执行强制调度至 **Firecracker**。

### 12.5 规则分级治理与评审制度 (Governance & Reviews)

当图式规模在商业开发中膨胀时，极易落入规则冲突与系统死锁。为此，我们建立了三层规则治理体系：
1. **宪法级规则 (Constitutional Rules)**：定义全局不可违反的安全红线（如凭证拦截模式、全局 killswitches 决策），该级别策略只能通过官方签名包更新。
2. **项目级规则 (Project-Scoped Rules)**：在 `CLAUDE.md` 或 `AGENTS.md` 中定义的规则，用于协调项目特定的分支管理与文件读写作用域，支持项目管理员更新。
3. **任务级临时规则 (Task-Scoped Rules)**：在单次 ReAct 循环或子代理派生中动态注入的局部限制（TTL 内有效），到期即销毁。


任何新 Schema 进入主干运行前，必须通过 **Schema Review (图式评审)** 确认其输入输出契约与风险等级；外部第三方开源引擎接入时，需通过 **Capability Intake Review (能力准入评审)** 建立其许可证与安全影响评估卡，以 JSON-RPC 作为唯一的进程边界。

---

## 结论 (Conclusion)

图式沙箱（Schema Sandbox）识别了介于纯提示词控制与基础设施级虚机隔离之间的中间设计空间。其架构逻辑可以精准抽象为三个核心边界：

* **图式沙箱 (Schema Sandbox)** 决定了**能力边界**：即智能体能够并且被允许生成什么。它通过九层神经符号压实与 Piagetan 自愈动力学，将动作空间收束在逻辑安全区。
* **工作区分区 (Workspace Partition)** 决定了**执行边界**：即动作在哪里、以何种代价被隔离。它采用 Deno/WASI、gVisor、Firecracker 分级运行，使防御成本与安全风险精确匹配。
* **图式互操作协议 (SIP)** 决定了**互操作与资产边界**：即不同沙箱如何被宿主发现、挂载、校验、审计、撤销与在生态中进行商业流转。

综上，“三层存储、五层压实、八层防御”的工程实现与统一的 SIP v1.1 互操作协议共同构成了一个高可行性工程标准，不仅能以 1.5 微秒（p50）的微秒级开销完成本地门控校验，更让智能体系统正式拥有了在开放生态中互通与自我演化的操作系统级底层底座。

---

## 参考文献 (References)

[1] Hubert Dreyfus. *What Computers Can't Do: A Critique of Artificial Reason*. Harper & Row, 1972.  
[2] Brandon T. Willard and Remi Louf. "Efficient Guided Generation for Large Language Models." *arXiv preprint arXiv:2307.09702*, 2023.  
[3] Jaideep Ray. "The Constraint Tax: Measuring Validity-Correctness Tradeoffs in Structured Outputs for Small Language Models." *arXiv preprint arXiv:2605.26128*, 2026. [preprint]  
[4] Carter Packer, Sarah Wooders, Kevin Lin, Vivian Fang, Shishir G. Patil, Ion Stoica, and Joseph E. Gonzalez. "MemGPT: Towards LLMs as Operating Systems." *arXiv preprint arXiv:2310.08560*, 2023.  
[5] Guanzhi Wang, Yuqi Xie, Yunfan Jiang, Ajay Mandlekar, Chaowei Xiao, Yuke Zhu, Linxi Fan, and Anima Anandkumar. "Voyager: An Open-Ended Embodied Agent with Open-World Curiosity and Self-Improvement." *arXiv preprint arXiv:2305.16291*, 2023.  
[6] Saibo Geng, Hudson Cooper, Michal Moskal, Samuel Jenkins, Julian Berman, Nathan Ranchin, Robert West, Eric Horvitz, and Harsha Nori. "JSONSchemaBench: A Rigorous Benchmark of Structured Outputs for Language Models." *arXiv preprint arXiv:2501.10868*, 2025.  
[7] Anthropic. "Claude Code Documentation." *Anthropic Developer Portal*, 2026. URL: https://code.claude.com/  
[8] Jiacheng Liu, Xiaohan Zhao, Xinyi Shang, and Zhiqiang Shen. "Dive into Claude Code: The Design Space of Today's and Future AI Agent Systems." *arXiv preprint arXiv:2604.14228*, 2026. [preprint]  
[9] Guidance AI Team. "Guidance: A Domain-Specific Language for Controlling Large Language Models." *GitHub Repository*, 2023. URL: https://github.com/guidance-ai/guidance  
[10] Tengjiao Liu and Hongzong Si. "Agent Concretization: Informational Boundaries and Persistent Agent IP." *OSF Preprints*, 2026. DOI: 10.31219/osf.io/y4vsh  
[11] Lei Zheng, Weinan Song, Daili Li, and Yanming Yang. "To Know is to Construct: Schema-Constrained Generation for Agent Memory." *arXiv preprint arXiv:2604.20117*, 2026. [preprint]  
[12] Shunyu Yao, Dian Yu, Jeffrey Zhao, Izhak Shafran, Thomas L. Griffiths, Yuan Cao, and Karthik Narasimhan. "Tree of Thoughts: Deliberate Problem Solving with Large Language Models." *NeurIPS*, 2023.  
[13] Yilun Xu, et al. "ActiveRAG: Active Retrieval Augmented Generation via Constructivist Assimilation." *arXiv preprint arXiv:2402.13547*, 2024.  
[14] Ruochen Xu, et al. "ThinkNote: Constructivist Dynamic Memory Compaction for Long-Context LLMs." *EACL*, 2026.  
[15] Ming Li, et al. "CAM: Constructivist Schema Clustering for Long-Horizon Agent Memory." *NeurIPS*, 2025.  
[16] Immanuel Kant. *Kritik der reinen Vernunft* (Critique of Pure Reason). Riga, 1781.  
[17] Jean Piaget. *The Origins of Intelligence in Children*. International Universities Press, 1952.  

---

## 附录 A：图式沙箱运维与实施手册 (Operations & Implementation Manual)

本手册专为软件开发人员与安全运维工程师（SRE）设计，旨在指导如何从头开始构建、打包、签名、运行与监控符合 SIP v1.1 协议标准的 L1/L2 级图式沙箱。

### A.0 读者角色与责任边界 (Roles and Responsibilities Matrix)
在沙箱开发与运维周期中，不同角色具有明确的责权边界：

| 角色 | 负责什么 | 不负责什么 | 必须掌握的章节 |
| :--- | :--- | :--- | :--- |
| **图式作者 (Sandbox Author)** | 编写 manifest.json、输入输出 schema、校验器与钩子。 | 不维护执行时的运行时内核。 | A.1, A.2, A.3 |
| **运行时工程师 (Runtime Engineer)** | 实现 SIP 加载器、本地策略内核与分区运行器。 | 不决定具体的业务规则。 | 3.1, 8.2, A.4 |
| **安全审查员 (Security Reviewer)** | 审查权限范围、规范化签名、证书及 SBOM 依赖项。 | 不编写具体的业务/功能 Schema。 | 7.0, 8.3, A.5, A.7 |
| **运维工程师 (SRE / Operator)** | 管理部署、日志轮转、监控指标导出与版本回滚。 | 不修改私有的 Prompt 逻辑链。 | A.6, A.9, A.10 |
| **市场审核员 (Marketplace Reviewer)** | 审核已签名的能力包、did 根证书与许可证合规性。 | 不执行本地 dry-run 与编写代码。 | 8.4, A.5, A.11 |

### A.1 环境准备与安装步骤 (Prerequisites and Environment Setup)
在运行沙箱前，请执行环境检测以验证本地依赖：
```bash
# 安装 sip-cli 工具链
npm install -g @schemabox/sip-cli

# 进行运行环境诊断
sip doctor
```
#### 环境支持矩阵：
* **支持的操作系统**: Linux (推荐 Ubuntu 22.04 LTS), macOS (13+), Windows 11 (Option B/C 必须启用 WSL2)。
* **最低依赖版本**:
  * **Option A**: Node.js v20+, Deno v1.40+, typescript v5.3+
  * **Option B**: Docker v24+, gVisor (runsc 内核) v2024+
  * **Option C**: 启用 QEMU/KVM 支持, Firecracker v1.7.0+
* **离线环境安装**: 可运行 `sip pack --offline-deps` 预先打包所有校验所需的依赖文件。
* **降级策略**: 当检测到宿主机缺失 Firecracker 或 gVisor 依赖时，`sip run` 将自动降级为 **Option A (Deno)** 逻辑隔离运行，并在 `EvidenceRecord` 中记录警告码 `SIP_WARN_DOWNGRADED_ISOLATION`。

### A.2 "geo_hello" 最小可运行沙箱包拓扑 (geo_hello Sandbox Package Topology)
工程师在项目中需要准备的标准 Hello World 能力包目录结构如下：
```text
geo_hello_sandbox/
├── manifest.json                 # SIP v1.1 能力清单文件
├── schemas/
│   ├── input_schema.json         # 输入契约 JSON Schema
│   └── output_schema.json        # 输出契约 JSON Schema
├── src/
│   ├── validator.ts              # 输入与输出后置校验器 (刚性/柔性双门控)
│   ├── compaction.ts             # 3层存储快照与合并逻辑
│   └── hooks.ts                  # 前置与后置拦截 API 钩子
├── policies/
│   └── capability_grant.json     # 本地权限声明文件
└── memory/
    ├── memory.md                 # 主经验/事实索引文件
    └── CLAUDE.md                 # 宪法/避坑指南 (只读)
```

### A.3 关键文件完整代码与配置 (Full Specification and Code Files)
#### A.3.1 manifest.json 示例
```json
{
  "sip_version": "1.1.0",
  "sandbox_id": "geo_hello",
  "name": "Geo Hello Sandbox",
  "version": "1.0.0",
  "capability_type": "diagnosis.hello",
  "metadata": {
    "author": "your-team",
    "license": "Apache-2.0",
    "description": "Minimal reference Schema Sandbox for GEO diagnosis Hello World",
    "signature": "ed25519:7e8bcfc09fa65123d8c1c4f923cde8fa93c20c0f8fa2b1d033924f0c8fa2b1d"
  },
  "trust": {
    "signature_algorithm": "ed25519",
    "canonicalization": "jcs-rfc8785",
    "publisher_key_id": "did:psi:publisher:ltj-2026",
    "revocation_url": "https://registry.psi.run/revoke/geo_hello",
    "signed_at": "2026-06-22T12:00:00Z"
  },
  "input_contract": {
    "schema_ref": "./schemas/input_schema.json",
    "sanitize_input": true
  },
  "output_contract": {
    "schema_ref": "./schemas/output_schema.json",
    "forbidden_patterns": ["the best", "guaranteed ranking"],
    "entropy_threshold": 5.0
  },
  "permission_scope": {
    "fs_scope": ["./workspace/reports"],
    "net_scope": ["*.example.com:443"],
    "tool_scope": ["validator:check"]
  },
  "workspace_partitions": [
    {
      "partition_id": "main",
      "isolation_level": "context",
      "memory_slice_scope": ["./memory"]
    }
  ],
  "validation": {
    "required_evidence": ["input_hash", "tool_args_hash", "output_hash"]
  },
  "error_codes": {
    "SIP_ERR_VERSION_MISMATCH": "0x01",
    "SIP_ERR_SIGNATURE_INVALID": "0x02",
    "SIP_ERR_INPUT_VIOLATION": "0x03",
    "SIP_ERR_SCOPE_LOCKED": "0x04",
    "SIP_ERR_LEAK_DETECTED": "0x05",
    "SIP_ERR_COMPACTION_VIOLATION": "0x08"
  }
}
```

#### A.3.2 schemas/input_schema.json 示例
```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "title": "GeoHelloInput",
  "type": "object",
  "required": ["brand_name", "website"],
  "properties": {
    "brand_name": { "type": "string", "minLength": 1 },
    "website": { "type": "string", "format": "uri" },
    "locale": { "type": "string", "default": "en-US" }
  },
  "additionalProperties": false
}
```

#### A.3.3 schemas/output_schema.json 示例
```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "title": "GeoHelloOutput",
  "type": "object",
  "required": ["brandName", "overallScore", "diagnosis", "evidence"],
  "properties": {
    "brandName": { "type": "string" },
    "overallScore": { "type": "number", "minimum": 0, "maximum": 100 },
    "diagnosis": { "type": "string" },
    "evidence": {
      "type": "array",
      "items": {
        "type": "object",
        "required": ["source", "claim"],
        "properties": {
          "source": { "type": "string" },
          "claim": { "type": "string" }
        }
      }
    }
  },
  "additionalProperties": false
}
```

#### A.3.4 policies/capability_grant.json 示例
```json
{
  "partitionId": "main",
  "fsScope": [
    "./workspace/reports",
    "./memory"
  ],
  "netScope": [
    "*.example.com:443"
  ],
  "toolScope": [
    "validator:check"
  ],
  "ttlSeconds": 900,
  "humanApproval": false,
  "denyByDefault": true
}
```
> [!WARNING]
> **危险越权配置拦截机制**:
> 任何包含通配符或全局根目录的配置，如 `"fsScope": ["/"]`、`"netScope": ["*:*"]` 或 `"toolScope": ["bash:*"]`，都将在 `sip audit` 静态审计阶段被拦截，禁止打包发布，并在审计日志中输出 `SIP_ERR_SCOPE_LOCKED` 错误。

### A.4 Hello World 完整执行 Walkthrough
1. **脚手架初始化**:
   ```bash
   sip init geo_hello_sandbox
   cd geo_hello_sandbox
   ```
2. **Review 并写入配置**: 照 A.3 样例补齐 manifests、schemas 以及 validator 代码。
3. **执行静态特权审计**:
   ```bash
   sip audit .
   ```
   *预期终端输出:*
   ```text
   [SIP] Running security audit on geo_hello_sandbox...
   [SIP] Verifying manifest signature roots... OK
   [SIP] Evaluating CapabilityGrant rules... OK
   [SIP] Checking for copyleft licenses (GPL/AGPL)... OK
   [SIP] SUCCESS: Audit passed. Zero privilege escalation vectors found.
   ```
4. **进行本地 Mock 门控验证 (Dry-Run)**:
   在 `tests/fixtures/mock_input.json` 中配置符合 `input_schema.json` 要求的 Mock 负载。
   ```bash
   sip run --dry --input tests/fixtures/mock_input.json
   ```
   *预期终端输出:*
   ```text
   [SIP] manifest loaded: geo_hello@1.0.0
   [SIP] signature: skipped in dry mode
   [SIP] input_contract: passed
   [SIP] permission_scope: passed
   [SIP] output_contract: passed
   [SIP] evidence written: evidence/EvidenceRecord.jsonl
   [SIP] SUCCESS: Dry-run completed. Outputs matching schemas.
   ```

### A.5 运行时选择决策树 (Runtime Selection Playbook)
工程师可按照以下逻辑为具体分区选择合适的物理隔离模式：

```text
任务是否执行任意、不可信的用户生成代码？
 ├── 是 ──> Option C (Firecracker 微型虚拟机隔离，启动延迟 50-300ms)
 └── 否 ──> 任务是否包含数据库写入、爬虫网络请求或不可信的二进制程序？
              ├── 是 ──> Option B (gVisor 容器沙箱隔离，启动延迟 0.5-2ms)
              └── 否 ──> Option A (Deno/WASI 进程级逻辑隔离，启动延迟 < 1ms)
```

### A.6 证书管理、签名与撤销命令 (Signing, Key Management, and Revocation)
为实现安全流转，开发者需要利用 `sip-cli` 完成 Ed25519 签名体系的生命周期管理：
1. **生成本地开发私钥**:
   ```bash
   sip keygen --alg ed25519 --out ~/.sip/keys/ltj-dev.key
   ```
2. **导出并检查公钥**:
   ```bash
   sip key inspect ~/.sip/keys/ltj-dev.pub
   ```
3. **规范化打包与签名**:
   ```bash
   sip pack --key ~/.sip/keys/ltj-dev.key --output geo_hello.sip
   ```
4. **包状态注册与校验**:
   ```bash
   sip verify geo_hello.sip --registry https://registry.psi.run
   ```
5. **废弃并注销泄露的密钥**:
   ```bash
   sip revoke geo_hello@1.0.0 --reason "leaked private key"
   ```
   宿主在挂载 `.sip` 能力包时，会首先查询 manifest 中声明 of `revocation_url` 废弃黑名单，若被废弃则拒绝 mount。

### A.7 EvidenceRecord 审计日志落盘与检索 (EvidenceRecord, Audit Logs, and Retention)
`EvidenceRecord.jsonl` 是追加写入的本地 JSON 文本文件，记录所有动作的审计轨迹。在写入前必须在本地红线过滤 PII 和敏感密钥。
#### 日志落盘单行格式：
```json
{
  "traceId": "tr_01HX89Z2...",
  "parentTraceId": null,
  "sandboxId": "geo_hello",
  "sandboxVersion": "1.0.0",
  "partitionId": "main",
  "policyDecision": "allow",
  "inputHash": "sha256:7f83...671e",
  "outputHash": "sha256:923b...822f",
  "previousRecordHash": "sha256:1a82...3ef4",
  "recordHash": "sha256:c23b...09da",
  "timestamp": "2026-06-22T12:00:00Z"
}
```
#### 运维命令行操作：
* **实时追踪日志**: `sip evidence tail --follow`
* **完整性与链条校验**: `sip evidence verify --since 24h` （用于防止回滚与篡改审计记录）
* **导出指定调用链**: `sip evidence export --trace tr_01HX89Z2 --format json`

### A.8 CI/CD 自动化流水线集成 (CI/CD Release Gate Example)
以下是生产环境中基于 GitHub Actions 的规范发布流程配置：
```yaml
name: SIP Package Release Gate

on:
  push:
    tags:
      - "v*"

jobs:
  sip-release:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'
      - name: Install CLI
        run: npm install -g @schemabox/sip-cli
      - name: Static Security Audit
        run: sip audit .
      - name: Execute 5-Regression Suite
        run: sip test --suite all
      - name: Sign & Pack Bundle
        run: sip pack --key ${{ secrets.SIP_SIGNING_KEY }} --output dist/geo_hello.sip
```

### A.9 五大回归测试夹具设计 (Five-Regression Test Fixtures)
用于跑通发布前自动化阻断测试的 fixtures 结构如下：
```text
tests/
├── release_leakage/
│   ├── leaked_source.js.map
│   └── expected_block.json
├── jaw_injection/
│   ├── malicious_readme.md
│   └── expected_rejection.json
├── structured_output/
│   ├── invalid_score_output.json
│   └── expected_schema_error.json
├── canary_egress/
│   ├── canary_secret.txt
│   └── exploit_prompt.txt
└── concurrency/
    └── load_profile.yaml
```
执行对应测试的命令如下：
```bash
sip test --suite release-leakage
sip test --suite injection
sip test --suite structured-output
sip test --suite canary-egress
sip test --suite all
```

### A.10 生产环境故障应急预案 (Troubleshooting and Incident Response Runbooks)
SRE 在发生系统阻断或告警时，需执行以下预案流程：
1. **故障：p95 门控延迟超过 5 毫秒**
   * *排查*: 检查 Prometheus 统计。延迟是在 Pre-Gate（正则与数据脱敏）还是 Post-Gate（熵计算）？
   * *恢复*: 修改运行时标志，动态关闭非安全强相关的柔性 Schema 校验规则，或紧急回滚到上个稳定版本的能力包。
2. **故障：阻断率（Violation Rate）突增超过 5%**
   * *排查*: 分析 `EvidenceRecord`，定位是否发生恶意的 Prompt 注入攻击。
   * *恢复*: 在入口处动态开启更严苛的 Pre-Gate 敏感词防御配置，对异常请求源 IP 进行熔断。
3. **故障：EvidenceRecord 写入失败/磁盘爆满**
   * *排查*: 存储写入异常或并发锁死。
   * *恢复*: 将日志路径自动切换至备用的内存挂载分区，并降级禁止有外部副作用的工具（如网络写入），直至磁盘清理完毕。
4. **故障：签名私钥泄露**
   * *排查*: 密钥不慎上传 Git 或输出到调试日志。
   * *恢复*: 执行 `sip revoke` 命令向 PSI 官方中心注销此密钥 ID，更新 KMS 配置，并重新构建签名发布新包。
5. **故障：沙箱压缩死锁/无限重试**
   * *排查*: 模型生成持续不符合 Output 规范引发模型无限回环自愈。
   * *恢复*: 触发运行时断路器（Circuit Breaker），立刻将 fallback 策略转为 `AskHuman`，推送到人工客服工单进行干预。

### A.11 版本变更与平滑升级策略 (Versioning, Upgrade, and Rollback)
能力的更新必须符合以下 Semantic Versioning 兼容性准则：
* **修订版 (x.y.Z / Patch)**: 仅修改内部 validator.ts 的代码逻辑，输入输出 Schema 完全未做变动。
* **次版本 (x.Y.z / Minor)**: 仅在 input/output schema 中新增了**可选 (Optional) 字段**，且未扩大 `permission_scope` 权限。
* **主版本 (X.y.z / Major)**: 修改了原字段结构、删除了参数或**扩大了 filesystem/network/tool 等权限声明**。**必须提交人工安全评审后方能上线。**
* **回滚恢复**: 升级一旦在生产中产生未意料阻断，SRE 应对 mount 指向的能力包文件路径进行原子软链接替换，回滚至上一签名包；宿主系统保留的历史 Schema 能够自动兼容历史日志的审计追溯。

### A.12 生产上线前就绪检查清单 (Production Readiness Checklist)
请在最终发布沙箱包到生产前确认以下清单已全部打勾：
- [ ] 证书密钥已通过 KMS/HSM 动态签发，且 `sip verify` 校验状态为真。
- [ ] 代码库包依赖扫描通过，确认未包含任何未隔离的 GPL/AGPL 开源污染依赖。
- [ ] `fs_scope` 未包含任何全局通配符（如 `/` 或 `../`）。
- [ ] `net_scope` 的域名过滤规则精准锁定至服务所需的特定三级子域名。
- [ ] `EvidenceRecord.jsonl` 日志轮转规则已在目标宿主机配置，避免日志过大导致磁盘爆满。
- [ ] 本地门控在高并发模拟场景下的 p95 响应延迟低于 5ms。

