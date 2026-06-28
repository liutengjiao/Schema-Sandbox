# Schema Sandbox: A Nine-Layer Architecture and Interoperability Contract for Constrained Agent Execution

**Tengjiao Liu**  
*Founder & Researcher, psi.run*  
psi@psi.run  

**Hongzong Si**  
*Professor, Qingdao University*  
sihz@qdu.edu.cn  

---

## Abstract

LLMs generate probabilistically; production systems require deterministic contracts. Prompts and tool calls alone are insufficient to sustain high-risk, long-horizon, and auditable agentic execution. We present the **Schema Sandbox**----a neuro-symbolic constraint architecture that sits between raw LLM outputs and persistent Agent execution, alongside the **Schema Interoperability Protocol (SIP v1.1)**. Drawing from Kantian schematism and Piagetian dynamics, we establish a theoretical foundation for agent self-healing via "assimilation" and "accommodation" loops, deriving its negative entropy compaction bounds. The core technical contributions of this study are: first, we define a nine-layer neuro-symbolic cognitive constraint architecture that turns probabilistic token streams into structured, contract-compliant actions; second, we specify a **layered hybrid runtime framework** (Option A: Deno + WASI/Wasmtime; Option B: gVisor; Option C: Firecracker VM) and workspace partition isolation models; third, we introduce SIP v1.1 as a manifest-level capability mounting and governance protocol, defining seven core modules (metadata, contracts, permissions, partitions, validation, errors) that enable sandboxes to be discovered, verified, mounted, and exchanged as secure capability assets. Micro-benchmarks show that local validation gates execute in 1.5 microseconds (p50).

---

> [!NOTE]
> ### Quick Start Guide for Implementers
> This document defines the comprehensive theoretical framework and engineering specifications for the Schema Sandbox. If you are an engineer tasked with implementing this architecture, we suggest following this quick start path rather than reading the entire paper sequentially:
> 
> 1. **Where do I start writing code?**
>    - **Step 1 (Core Contracts)**: Implement the `SchemaManifest` and `OutputContract` interfaces (see Section 3.1). These define the neuro-symbolic enforcement boundaries; all other gates depend on them.
>    - **Step 2 (Local Kernel)**: Implement **Option A (Deno + WASI)** as the default runtime engine for lightweight logical isolation (see Section 6.1).
>    - **Step 3 (Progressive Hardening)**: Depending on risk profiles, decide whether to mount **Option B (gVisor)** or **Option C (Firecracker)** for physical process isolation.
> 2. **What does my first sandbox (Hello World) look like?**
>    - Jump directly to **Section 9 (Case Study: Brand Shuttle GEO)**. It lists the precise logical flow and file system layout of a reference sandbox package.
> 3. **What sections can I skip or defer reading?**
>    - If your goal is to build an MVP quickly, you can defer reading **Section 2 (Background and Related Work)** and **Section 7 (Security Threat Model)**. Save them for security hardening during Phase 3.

## 1. Introduction

Long-horizon agents fail less because of model intelligence and more because of missing boundaries. Prompts that work for single turns degrade over dozens of steps. Formatting breaks, rules get ignored, and state drifts.

While building production agents, we found that post-hoc validation is too late. The model has already committed to a bad token before any checker runs. What we need is an active boundary that can operate before, during, or after generation, depending on the sandbox level and enforcement mechanism.

We call this boundary the Schema Sandbox. It is not another memory wrapper. It is a cognitive constraint layer that turns probabilistic token streams into contract-compliant actions. This paper specifies its architecture and the protocol for making such boundaries interoperable across agents. This work addresses an intermediate design space between prompt-only control and infrastructure-level sandboxing.

### 1.1 Philosophical Grounding: Kantian Schematism and Piagetian Dynamics
Hubert Dreyfus's philosophical critique of disembodied AI highlights the drift that occurs when systems lack situated context [1]. In agent systems, such constraints are informational rather than physical, bounding the attention, state, and permissible actions of the model. Dreyfus's critique is not used here as a direct technical model, but as a useful analogy: agency requires situated constraints.

To anchor this informational boundary theoretically, we draw from two seminal cognitive paradigms:
1. **Kantian Schematism**: In the *Critique of Pure Reason* (1781), Immanuel Kant argued that sensory intuitions and intellectual categories are fundamentally heterogeneous; they must be synthesized by a "Transcendental Schema" acting as a cognitive compiler. In silicon minds, the hyper-dimensional latent manifold of the LLM corresponds to "sensory intuition," while human-defined task ontologies and system schemas represent the "categories." The Schema Sandbox acts as this Transcendental Schema compiler, transforming raw probability fields into structured, compliant actions.
2. **Piagetian Dynamics**: Jean Piaget modeled cognitive development as a self-regulating loop of "Assimilation" (gating actions within existing mental schemas) and "Accommodation" (modifying internal mental structures in response to environmental resistance/errors). In our sandbox, when an agent's proposed action matches the active schema, it is executed and logged (assimilation). When an action is blocked due to validation failure, it triggers accommodation, executing a Compaction Operator to reorganize internal rules and repair the agent's cognitive "skin."

In our companion position paper [10], we argued that persistent agent identity (Agent IP) requires a boundary mechanism - the Schema Sandbox - to constrain cognitive drift and prevent identity dissipation. The present work provides the architectural specification and interoperability contract for that boundary layer.

### 1.2 Core Technical Contributions
To address the security and interoperability bottlenecks of long-horizon agents in production, this work provides three core technical contributions:
1. **Schema Sandbox as a Cognitive Constraint Architecture**: We define a neuro-symbolic boundary covering domain corpora, ontologies, input contracts, memory compaction, tool grammars, defense safety chains, and output contracts, suppressing autoregressive attention drift through mathematical negative entropy.
2. **Workspace Partition as the Execution Isolation Model**: We specify a layered hybrid runtime framework offering multi-tier physical and logical containment using Deno/WASI local kernels (Option A), gVisor containers (Option B), and Firecracker microVMs (Option C), matching execution overhead to risk profiles.
3. **SIP as the Interoperability and Capability-Asset Protocol**: We formalize the Schema Interoperability Protocol (SIP). We argue that **while the nine-layer architecture dictates *how to construct a single Schema Sandbox*, SIP governs *how heterogeneous sandboxes are discovered, mounted, invoked, composed, verified, exchanged, and long-term evolved*.** Without SIP, the Schema Sandbox remains merely a localized engineering methodology; with SIP, it becomes an open standard, capability marketplace, and foundational layer for Agent IP ecosystems.


---

## 2. Background and Related Work

Our work intersects with structured generation, agent runtimes, and execution security.

### 2.1 Constrained Decoding
Autoregressive language models generate text by sampling from a probability distribution over a vocabulary. Constrained decoding restricts this search space by modifying the logit distribution at each step of generation. Frameworks like Outlines [2] and Guidance [9] compile context-free grammars (CFGs) or JSON schemas into regular expression state machines or prefix-tries. At step k, the logit L_k[i] for any token i not belonging to the set of valid transitions V(valid) is masked:

L'ₖ[i] = Lₖ[i] (if i ∈ V(valid)) or -∞ (if i ∉ V(valid))

While this mathematically guarantees that the output strictly conforms to a specified syntax, it introduces a "constraint tax" - wherein restricting the decoding path can degrade the model's reasoning capabilities and increase semantic error rates on complex tasks [3]. To mitigate this degradation, this paper proposes classifying active constraints into **刚性约束 (Rigid/Security Bounds)** and **柔性约束 (Soft/Formatting Bounds)**. Rigid bounds enforce strict security perimeters and are never compromised. Soft bounds enforce syntactic conventions; if soft validation fails repeatedly, the runtime initiates "Dynamic Constraint Relaxation," allowing freeform output which is cleaned up by local post-processing, thereby preserving the model's reasoning capacity. This design contrasts with static schema rules in early 2026 preprints on Schema-Constrained Generation for Agent Memory (SCG-MEM) [11].

### 2.2 Agent Harnesses and Runtimes
To execute long-horizon tasks, models are wrapped in agent harnesses that manage state, memory, and tool calls. Systems like MemGPT (Letta) treat LLM context windows as virtual memory, using paging to manage finite token budgets [4]. Tree of Thoughts [12] structures agent workflows by planning search paths through intermediate states. However, these harnesses lack a unified cognitive constraint model, leaving the boundary between LLM reasoning and system tool execution largely ad-hoc.

### 2.3 Sandboxing and Permissions
Traditional execution sandboxing operates at the operating system or virtualization level. Containerization (e.g., Docker) and micro-virtual machines (e.g., AWS Firecracker) isolate untrusted code execution. OS-level physical isolation is necessary when agents execute arbitrary code. However, in many workflows, the dominant failure modes are schema violations, data leakage, or malformed tool calls. For these cases, logical scope locks and data sanitization provide safety at low latency (1.5 us vs. 50 ms for micro-VMs).

### 2.4 Structured Output Reliability
Generating syntactically valid JSON is insufficient for enterprise applications. The semantic correctness of fields within that JSON remains highly variable. Benchmarks like JSONSchemaBench show that even when models output valid JSON structures under constrained decoding, they frequently fail semantic constraints (e.g., generating invalid database IDs or violating value range restrictions) [6].

### 2.5 Constructivist Agents & Self-Evolutionary Memory (2025-2026 Frontier)
Recent research is moving away from passive memory retrieval towards constructivist, active, and self-evolving memory architectures. For instance, **ActiveRAG** [13] and **ThinkNote** [14] implement computational assimilation and accommodation loops to resolve conflicting evidence, while **CAM** (Li et al., NeurIPS 2025) [15] organizes long-context memory using dynamic schema clustering. However, none of these systems treat physical and computational resource constraints (budgets, credits, sandbox failures) as active boundaries to define agent identity.

### 2.6 Industrial Agent Harness Analysis
In early 2026, commercial developer tools highlighted the critical role of "harnesses" surrounding raw LLMs. Source code analyses of **Claude Code** (dissecting ~513k lines of TypeScript codebase [8]) reveal that the core ReAct loop (`queryLoop()`) represents less than 2% of the system; the remaining 98% is the execution harness, featuring a 3-layer memory architecture (ephemeral context, `memory.md` pointer index, and `CLAUDE.md` constitution), a 5-layer progressive compaction pipeline, an 8-layer defense-in-depth safety chain, and dynamic Hook systems. Similarly, **Cursor** leverages Merkle tree workspace tracking, AST semantic chunking, and MDC intelligent rules; while **Aider** utilizes git-native commits and repository maps (Repo Map) to achieve a 4x token efficiency gain over Claude Code. These practices demonstrate that the core challenge of engineering agents has shifted from base models to the surrounding runtime harness.

## 2.7 Structured Security Skills and Gated Context Architectures
Recent engineering patterns for advanced agent harnesses further validate the necessity of modular, context-gated architectures. For instance, Mukul et al.'s **Anthropic-Cybersecurity-Skills** [18] (implementing 817 structured cybersecurity skills based on the `agentskills.io` standard) demonstrates that stuffing an agent's context window with large, monolithic rulebooks or extensive domain knowledge degrades reasoning and leads to context window explosion. Instead, the project structures complex domains (such as cybersecurity threat analysis under MITRE ATT&CK and NIST guidelines) into modular, self-contained files featuring YAML frontmatter metadata and markdown task blueprints. By performing *on-demand dynamic loading* of specific skills based on the active task context, the runtime implements a localized form of Piagetian assimilation (matching inputs to narrow capability scopes) and bounds the agent's execution parameters. This pattern directly maps to the Domain Corpus (L1), Knowledge Selector (L5), and Tool/API Grammar (L7) layers of the Schema Sandbox, serving as a real-world industrial validation of the nine-layer neuro-symbolic constraint architecture.

---

## 3. Formal Definition and Self-Evolutionary Loop

The Schema Sandbox operates as a cognitive boundary with a neuro-symbolic interface: the neural side generates, the symbolic side gates.

Mathematically, let L be the high-dimensional latent manifold of the base Large Language Model (LLM), and let sₜ in S be the persistent state representation of the agent at step t. We define the **Schema Sandbox** as a dynamic constraint operator Sₜ = ⟨Kₜ, Φₜ⟩, where Kₜ represents the active schema topology (e.g., grammar trees, rule sets, or physical collision grids) and Φₜ: A × S → {0, 1} represents the sandbox verification function. We use Bₜ ≡ Sₜ to denote the active boundary. The sandbox filters the base LLM transition probability distribution P(base)(a | s) and projects it onto the restricted boundary:

P(agent)(a | s) = Sₜ ∘ P(base)(a | s) = (Φₜ(a, s) * P(base)(a | s)) / Zₜ

where Zₜ = ∫(a' ∈ A) Φₜ(a', s) * P(base)(a' | s) da' is the normalizing partition function.

At the token-level decoding stage, for a vocabulary V, the sandbox enforces constraints on the logit vector Lₖ ∈ ℝ^|V| by isolating the allowed subset V(valid) ⊂ V derived from Kₜ:

L'ₖ[i] = Lₖ[i] (if i ∈ V(valid)) or -∞ (if i ∉ V(valid))

This blocks invalid tokens before sampling, preventing formatting error accumulation.

### 3.1 Five Core Contract Interfaces
To operationalize this cognitive constraint, we formalize the Schema Kernel into five core contract interfaces:
1. **SchemaManifest**: Defines metadata, SemVer, input/output schema references, risk ratings (L0-L3), and the specific evidence logs required for compliance audits.
2. **PartitionPolicy**: Establishes partition scheduling rules, memory slices, token quotas, and emergency fallback routes.
3. **CapabilityGrant**: Authorizes fine-grained host system access, including file system boundaries (FS Scope), allowed remote endpoints (Net Scope), CLI commands (Tool Scope), and token leases (TTL).
4. **EvidenceRecord**: Logs execution hashes, parameters, gating decisions, and outputs in an append-only audit trail.
5. **OutputContract**: Governs post-parsing sanitization, regex blocklists, entropy thresholds, and self-healing parameters.

The interfaces are defined as follows:
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
  memorySliceScope: string[]; // path mappings allowed in memory.md
  tokenBudget: number;      // maximum allowable token budget
  concurrencyLimit: number; // concurrency constraints
  fallbackRoute: string;    // fallback schema/partition ID on failure
}

interface CapabilityGrant {
  partitionId: string;
  fsScope: string[];       // restrict to absolute directory
  netScope: string[];      // restrict allowed domains/ports
  toolScope: string[];     // restrict allowed commands
  ttlSeconds: number;      // token lease duration
  humanApproval: boolean;  // force interactive trust dialog
}

interface EvidenceRecord {
  traceId: string;
  parentTraceId?: string;   // link to parent trace for sub-agent chains
  sandboxId: string;        // originating sandbox
  sandboxVersion: string;   // semver of the sandbox at execution time
  partitionId: string;      // which workspace partition ran this action
  actorAgentId?: string;    // identity of the invoking agent
  modelProvider?: string;   // e.g. "anthropic", "openai"
  modelId?: string;         // e.g. "claude-4-sonnet"
  policyVersion: string;    // version hash of the active policy ruleset
  inputHash: string;        // sha256 or blake3
  toolName?: string;
  toolArgsHash?: string;    // sha256 or blake3
  policyDecision: 'allow' | 'deny' | 'ask' | 'defer';
  outputHash: string;       // sha256 or blake3
  timestamp: string;        // ISO-8601
  digestAlgorithm: 'sha256' | 'blake3';
}

interface OutputContract {
  schemaRef: string;       // JSON Schema reference
  sanitizationRules: {
    stripSecrets: boolean; // toggle credential redaction
    stripPII: boolean;     // toggle personally identifiable info filtering
    customBlockPatterns: string[]; // custom regex filters to block
  };
  entropyThreshold: number; // maximum allowable information entropy
  selfHealingRules: {
    retryLimit: number;    // maximum self-healing retries
    fallbackSchemaId?: string; // backup Schema
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

### 3.2 SIP vs. Adjacent Concepts & Agent IP Mapping

The Schema Interoperability Protocol (SIP) functions not merely as an interface schema, but as a governance-layer contract enabling composition and assetization of agent capabilities. To clarify its positioning, we delineate its boundaries against Model Context Protocol (MCP) and map its role in Agent IP realization.

#### 1) Delineation Against Adjacent Concepts (MCP, Structured Outputs, and OS Containers)
MCP standardizes resource and tool connectivity between LLMs and external data sources. SIP standardizes capability-package governance for self-contained sandbox execution. They are complementary rather than mutually exclusive: an agent runtime could use MCP to discover available tools and SIP to enforce execution boundaries when invoking them. The table below (Table III) compares the capability dimensions of our Schema Sandbox + SIP framework against adjacent concepts, including standard Structured Outputs (logit masking), Model Context Protocol (MCP), and traditional OS-level container isolation (Docker/gVisor/Firecracker):

| Dimension | Structured Outputs | Model Context Protocol (MCP) | Containerization (Docker/gVisor/Firecracker) | Schema Sandbox + SIP |
| :--- | :--- | :--- | :--- | :--- |
| **Output Structure** | Yes | Partial | No | Yes |
| **Semantic Validation** | Weak | Tool-side | No | Yes |
| **Tool Connection** | No | Strong | No | Compatible with MCP |
| **Permission Declaration** | No | Partial / Client-side | OS-level | Embedded in Manifest |
| **Capability Package Identity** | No | Server / Package level | No | Yes |
| **Audit Evidence Chain** | No | Non-core | Requires self-build | Core Module |
| **Marketable Capability Leasing** | No | Extensible registry | No | Core Design |

__TABLE_III__

#### 2) Agent IP Capability Mounting Model
In our framework, Agent IP represents the persona, identity, and ownership credentials of a system. However, the core agent engine should remain decoupled from domain-specific rules and high-risk tools. SIP acts as the interface through which an Agent IP mounts isolated "cognitive lobes" (sandboxes):

Agent IP (Identity + General Reasoning) --[SIP Mount]--> Schema Sandbox (Execution Bounds + Contracts) --> Verifiable Output Action

Through this mounting model, an Agent IP can dynamically rent, mount, authorize, and revoke professional capability modules. This transforms Schema Sandboxes from internal engineering components into composable, secure, and commercially exchangeable capability assets.

### 3.3 Constraint Projection Proposition
Introducing the Schema Sandbox Sₜ strictly reduces the feasible action support set. Let Supp(Sₜ) = {a ∈ A : Φₜ(a, s) = 1} denote the set of actions permitted by the sandbox. Then:

|Supp(Sₜ)| ≤ |A|,  and  ∑(a ∉ Supp(Sₜ)) P(base)(a | s) is eliminated entirely.

Under uniform or bounded-skew distributional assumptions, this support reduction induces entropy compression:

H(P(agent)) ≤ H(P(base)) - ΔH(Sₜ),  where ΔH(Sₜ) = -log Zₜ ≥ 0.

However, we note that Shannon entropy does not monotonically decrease under arbitrary conditioning: if the sandbox filters out high-probability actions, the renormalized distribution over remaining low-probability actions may exhibit higher entropy. In the general case, the more robust metrics are: (1) invalid-action probability mass elimination, (2) feasible support set reduction |Supp(Sₜ)| / |A|, and (3) violation rate reduction. These metrics are unconditionally guaranteed by the sandbox projection, regardless of the base distribution's skewness.

### 3.4 The Self-Evolutionary Loop Algorithm
To prevent the model from entering a thrashing loop (endless blind retries) and mitigate the "Constraint Tax," we classify sandbox constraints into two categories:
1. **刚性约束 (Rigid/Security Bounds)**: Critical system-level controls, file system access scopes (`fs_scope`), network egress scopes (`net_scope`), and subprocess privileges. Rigid bounds are absolute and fail-fast; they are never relaxed.
2. **柔性约束 (Soft/Formatting Bounds)**: Output structure schemas, forbidden output string patterns (`forbidden_patterns`), and layout styles. Soft bounds permit **Dynamic Constraint Relaxation**.

When an action is intercepted, the sandbox returns a structured **Rejection Payload**. If a soft constraint fails repeatedly (exceeding a threshold of 3 attempts), the system relaxes logit-masking, allowing the base model to output plain text. This output is then formatted externally using post-processing rules (regex or local validation models), preserving the model's cognitive capacity for complex reasoning. The self-evolutionary loop is structured as follows:

```python
def schema_sandbox_execution_loop(input_task, S_t, manifest, retry_count=0):
    # Generate action proposal using the active schema K_t
    action = generate_raw(input_task, S_t.K_t)
    
    # Validate action against CapabilityGrant
    # Rigid Bounds verification: immediate failure if breached, no degradation allowed
    if not authorize_capability(action, S_t.Grant):
        rejection = build_rejection_payload("AUTHORIZATION_FAIL", S_t.Grant)
        return S_t, rejection  # Return standardized Rejection Payload
        
    validation_res = S_t.Phi_t(action)
    if validation_res.passed:  # Assimilation
        result = execute_action(action)
        
        # Log EvidenceRecord
        record_evidence(action, result, manifest.required_evidence)
        update_episodic_memory(action, result, success=True)
        return S_t, "Assimilation_Success"
    else:  # Accommodation
        # Check if violation type is a soft/formatting bound and retries are exhausted
        if validation_res.is_soft and retry_count >= 3:
            # Dynamic Constraint Relaxation
            # Relax logit masking restrictions and post-process plain text output locally
            relaxed_action = local_post_process(action, validation_res)
            if verify_soft_recovery(relaxed_action):
                result = execute_action(relaxed_action)
                record_evidence(relaxed_action, result, manifest.required_evidence)
                return S_t, "Relaxed_Success"
                
        # Accommodation logic: return RejectionPayload for self-healing in LLM context
        rejection = build_rejection_payload(validation_res.error_type, validation_res.bounds)
        error_trace = rejection.to_context_string()
        
        # The CompactionOperator refines rules via a read-before-write process,
        # updating the Epigenetic Prompt Layer (Prompt(self)) over Prompt(core)
        new_K = CompactionOperator.reorganize(
            S_t.K_t, (input_task, action), error_trace
        )
        
        # Compile the updated sandbox (updating rules, active Hooks, and workspace partitions)
        S_t_plus_1 = compile_new_sandbox(new_K)
        
        return schema_sandbox_execution_loop(
            input_task, S_t_plus_1, manifest, retry_count + 1
        )
```

### 3.5 Design Goals and KPIs

From an engineering systems perspective, the Schema Sandbox design must guarantee provable boundaries, reproducible execution, predictable costs, and manageable extensibility. To quantitatively evaluate these system properties, we define twelve key performance indicators (KPIs) and service level objectives (SLOs):

| Design Dimension | Key Performance Indicator (KPI) | Target Value (SLO) | Measurement Methodology |
| :--- | :--- | :---: | :--- |
| **Structural Safety** | Output contract grammar violation rate | 0 | Automated validation of all JSON/Schema/Grammar outputs; verified via JSONSchemaBench. |
| **Semantic Safety** | Critical field semantic violation rate | < 1% | Semantic validators & rule engine evaluation, tracked at object and field levels. |
| **Escaped Agency** | Unauthorized tool call escape rate | 0 | Red-team scripting and synthetic prompt injection replay testing. |
| **Credential Security** | Secret/PII leak recall rate | > 99% | Outbound traffic filtering audits using canary tokens, mock credentials, and known PII. |
| **Isolation Strength** | Unauthorized cross-partition read/write rate | 0 | Mock exploits simulating summary extraction bypass, cache pollution, and subagent forks. |
| **Performance Overhead** | Local validation gate latency (p95) | < 5 ms | Micro-benchmarks over 10,000 runs, separating policy checks, sanitization, and auditing. |
| **End-to-End Efficiency** | Standard task runtime overhead increase | < 15% | Comparative analysis against unconstrained execution, segmenting model, tool, and gate latencies. |
| **Long-Horizon Stability** | Post-compaction task success rate degradation | < 5% | A/B testing on long-chain tasks, comparing active steps and rule drift pre- and post-compaction. |
| **Scalability** | Concurrent active partitions per host | Mapped by resource tier | Capacity planning tests on 8/16/32 vCPU tiers across L1/L2/L3 runtime engines. |
| **Auditability** | Evidence trail coverage for side effects | 100% | Mandatory cryptographic hashing and `EvidenceRecord` logs for all external actions. |
| **Developer Velocity** | Initial schema deployment turnaround | < 5 person-days | Elapsed time tracking from input schema definition to automated pipeline registry deployment. |
| **Operations Velocity** | Pipeline regression suite execution time | < 2 hours | Concurrent execution of syntax, vulnerability, performance, and compliance check packages in CI/CD. |

---

## 4. The Nine-Layer Architecture

To implement cognitive constraints, we standardize the Schema Sandbox into a nine-layer reference architecture. Table II maps each layer to its target failure modes, scale spectrum requirements, and coverage differences against prior art.

### Table II: The Nine-Layer Schema Sandbox Architecture

| Layer | Failure Mode | v16 Mechanism & Mappings (Inspired by Claude/Cursor/Aider) | L1 | L2 | L3 | MVP Priority | Coverage Difference |
| :--- | :--- | :--- | :---: | :---: | :---: | :---: | :--- |
| **1. Domain Corpus** | Hallucinations | File-based transparent Markdown guides, local indexing | Opt | Opt | Opt | **Tier 3: Advanced** | Outlines: grammar only; no retrieval. |
| **2. Task Ontology** | Intent confusion | Pydantic validation, Skill YAML (instructions + tools) | Mand | Mand | Mand | **Tier 3: Advanced** | MemGPT: dynamic memory; no ontology validation. |
| **3. Input Contract** | Malformed input | Zod schema checks, Pre-gate sanitization, PreToolUse Hooks | Mand | Mand | Mand | **Tier 1: Core Scaffold** | SCG-MEM: memory formatting; no input gating. |
| **4. Router** | Routing failure | Semantic router, priority rule engine, Workspace Partitions | Opt | Mand | Mand | **Tier 3: Advanced** | Tree of Thoughts: state planning; no partitioned routing. |
| **5. Memory & Compaction** | Context drowning, drift | **3-layer memory** (ephemeral, memory.md, CLAUDE.md); **5-layer compaction** (budget/snip/micro/collapse/auto); self-healing | Opt | Opt | Opt | **Tier 2: Robustness** | Voyager: skill execution; no memory selector or compaction. |
| **6. Procedure Schema** | Execution drift | DAG flow engines, transcripts checkpoints, sidechain transcripts | Mand | Mand | Mand | **Tier 1: Core Scaffold** | SCG-MEM: state constraints; no DAG procedure checks. |
| **7. Tool/API Grammar** | Invalid tool formatting | EBNF grammars, Logit masking (L0), AST shell/code validation | Opt | Mand | Mand | **Tier 2: Robustness** | Outlines: token syntax; no permission gating. |
| **8. Boundary & Perm** | Privilege escalation | **8-layer defense safety chain** (flags → killswitches → priority rules → LLM judge → blocklist → AST → dialog → bypass) | Opt | Mand | Mand | **Tier 2: Robustness** | Standard libraries: lexical constraints; no OS execution security. |
| **9. Output Contract** | Post-parsing leaks | Post-gate regex/entropy shielding, PostToolUse Hooks, self-healing | Mand | Mand | Mand | **Tier 1: Core Scaffold** | Outlines: output syntax; no post-generation leakage filters. |

---

### 4.1 Workspace Partitions and Multi-API Orchestration

In production environments, a sandbox must coordinate multiple heterogeneous capabilities. Collapsing all these channels into a single context causes permission confusion and data leakage.

The Schema Sandbox solves this by defining **Workspace Partitions** as logically isolated execution units.

```text
User Goal
  ↓
Router (Semantic Classifier + Rule Engine)
  ├─ Partition A: Search SERP Evidence (Own memory.md scope, Crawlee in isolated thread)
  ├─ Partition B: LLM Reasoning Core (Gemini/DeepSeek, restricted context slice)
  ├─ Partition C: Database connection (Parameterization, Row-Level Security, PII Gates)
  ├─ Partition D: Code Validator & Tests (AST syntax check, user-space Node subprocess)
  └─ Partition E: Bilingual Report Exporter (Output contract compliance)
  ↓
Boundary & Permission Layer (Enforces no cross-leak, graduated trust per partition)
  ↓
Output Contract (Verify summaries, self-healing memory.md write, export MD/JSON)
```

Each partition declares an **Isolation Contract**:
* **Memory & Data Boundaries**: A partition is limited to its local directory mapped in `memory.md`. To prevent race conditions and file lock conflicts when multiple partitions write concurrently, each partition only loads a read-only **Context Slice** (context snapshot) of the master memory. All mutations are written to its private, isolated **Write Buffer** instead of writing directly to global storage.
* **Tool & Grammar Scoping**: Restricted to a subset of tool schemas and EBNF templates.
* **Containment Model**:
  * *Context Scoping*: Isolating context slices in model prompts.
  * *Worktree Isolation*: Spawning temporary Git worktrees to isolate file system mutations, returning only summaries.
  * *Remote Isolation*: Physical containerization (e.g., Docker, WASM) for high-security tasks.

---

## 5. Representative Runtime Design Patterns and Engineering Specifications

The engineering implementation of the Schema Sandbox relies on lightweight and fast execution runtimes to avoid the performance penalties of heavy virtual machines.

### 5.1 3-Layer Memory and Concurrency-Safe Writes (Layer 5/6 Memory)
Instead of relying on simple flat-file writes which fail under concurrent operations, the Schema Sandbox manages memory state using **Conflict-Free Replicated Data Types (CRDTs)** (such as LWW-Register or Delta-State CRDT) or a local virtual file system (SQLite-backed VFS):
1. **Ephemeral Context**: Short-term logs and ReAct step traces inside the immediate model prompt.
2. **`memory.md` Pointer and Database Index**: Mapped through a virtual file system (VFS), storing local episodic experiences, decision paths, and core facts.
3. **`CLAUDE.md` (or `AGENTS.md`) Constitution**: Read-only core guidelines governing agent formatting, security rules, and style conventions.

Upon session teardown of a Workspace Partition, the main Router initiates a **Schema Merge** instead of dumping files directly:
* The Router extracts the partition's private `Write Buffer`.
* It invokes the `CompactionOperator` to execute a Git-like three-way merge or a CRDT State-Merge algorithm to resolve conflicts deterministically.
* The synthesized knowledge is then merged back into the master `memory.md` pointer file or database. This ensures state consistency across concurrent multi-agent executions, avoiding memory corruption or data overwrites.

### 5.2 5-Layer Progressive Compaction Pipeline (Layer 5 Compaction)
To prevent context window drowning, the harness runs a progressive compaction pipeline, triggering cheap layers first:
1. **Budget Check**: Limit tokens per message; truncate non-core blocks.
2. **Context Snip**: Drop historical, non-essential steps based on a sliding window.
3. **Micro-compaction**: Compress structural syntax and symbols to align with KV cache boundaries without invalidating compiled logits.
4. **Semantic Collapse**: Consolidate multiple rules into high-density representations using AST or regex patterns, substituting detailed texts with concise summaries.
5. **Auto-Summary**: Summarize conversation history using a lightweight model, tagging preserved segments with structural markers, and triggering circuit breakers to block infinite loops.

### 5.3 8-Layer Defense-in-Depth Safety Chain (Layer 8 Security)
For L2/L3 execution, agents face malicious prompt injections. We implement an 8-layer defense safety chain:
```text
[1. Feature Flags] → [2. Killswitches] → [3. Priority Rules] → [4. YOLO-LLM Classifier]
                                                                        ↓
[8. Bypass Valve] ← [7. Trust Dialog] ← [6. AST/FS Validators] ← [5. Pattern Blocklist]
```
1. **Feature Flags**: Statically exclude unsafe execution APIs at build time.
2. **Server Killswitches**: Provide remote capability to disable write permissions.
3. **Priority Rules**: Evaluate priority-configured rules (compiled from 8+ sources).
4. **YOLO-LLM Classifier**: Run an ultra-lightweight judge (`yoloClassifier.ts`) to audit bash commands.
5. **Pattern Blocklist**: Block dangerous bash commands (e.g., `rm -rf /`) using regular expressions.
6. **AST/FS Validators**: Parse bash and code scripts using tree-sitter to examine AST symbols, blocking symlink loops and glob escapes.
7. **Trust Dialog**: Request human confirmation via terminal prompts.
8. **Bypass Valve**: Allow automatic execution in trusted zones (Graduated Trust Mode) only if the action passes the previous seven checks.

When an action is blocked by any layer of the safety chain (e.g., AST parse failures, blocked command patterns, or file path violations), the control plane does not simply return a generic "Access Denied" or raw error code. To prevent the model from entering a "thrashing loop" (endless blind retries), the sandbox returns a structured **Rejection Payload**. This payload explicitly informs the model of the infraction type (`violationType`), details the allowed bounds (`allowedBoundaries`), and provides clear fallback directions (`suggestedFallback`, such as requesting human approval). This makes the sandbox a "road-signed guardrail" that guides self-healing rather than a blind brick wall.

---

## 6. L0-L3 Layered Hybrid Runtime Framework

Prompts filtering alone is structurally insecure, but executing every single lightweight task inside heavy virtual machines (microVMs) introduces high cold-start latencies and resource costs. To balance safety and performance, we establish a **Layered Hybrid Runtime Framework**:

```text
               [ Incoming Request / Semantic Router ]
                              │
       ┌──────────────────────┼──────────────────────┐
       ▼                      ▼                      ▼
  [ Option A (L1/L2) ]   [ Option B (L2) ]      [ Option C (L3) ]
  Deno + WASI Kernel     gVisor Containers      Firecracker VM
  · Deny-by-default      · Scoped Toolchains    · Untrusted Code Exec
  · Local Logit Masking  · Network Egress Gates · Physical Multi-Tenancy
  · Validation gate: p50 ≈ 1.5 μs · Scoped Directory     · Ephemeral rootfs & proxy
```

### 6.1 Option A: Local-First Capability Kernel (L1/L2 Default)
This serves as the default execution profile for the majority of standard L1/L2 tasks, leveraging **Deno + WASI/Wasmtime**:
* **Deno Deny-by-Default Permission Model**: By default, Deno denies access to the file system, environment variables, subprocesses, and the network. Scoped permissions are granted dynamically based on the active `CapabilityGrant` (e.g., read-write restricted to `./workspace/src`).
* **WASI Capability Sandboxing**: Third-party plugins, scripts, and utilities are compiled into WebAssembly bytecode and run inside Wasmtime, creating a secure boundary that prevents buffer overflows or host execution escapes.

### 6.2 Option B: Control Plane + Container Partitioning (L2 Production)
Designed for executing medium-risk external integrations, such as SerpAPI scrapers, browsers, database adapters, and build tools:
* Runs inside **gVisor** or scoped Docker containers. Spawning agents directly on the host machine without sandboxing risks compromising host integrity.
* The control plane communicates with the container worker via a JSON-RPC channel with no shared memory. Directory mounts are limited (e.g., mapping only `PROJECTS_PATH`), and egress traffic is restricted using network allowlists.

### 6.3 Option C: microVM Strong Isolation (L3 High-Risk)
When the agent must execute arbitrary untrusted code, run third-party scripts, or process multi-tenant databases with high-value assets:
* Spawns a **Firecracker microVM** with an isolated rootfs image, an ephemeral credentials proxy, and a restricted network interface.
* Physical CPU/memory boundaries are enforced, containing execution side effects. The microVM is immediately destroyed upon task completion (RTO < 5ms).

We summarize and compare the three options below:

| Option | Isolation Mechanism | Representative Tasks | Advantages | Primary Limitations | Recommended Conclusion |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **Option A: Local Kernel** | Deno/WASI + Constrained Decoding + Semantic Gating | Structured generation, data formatting, low-risk toolchains | Microsecond latency, local-first execution, minimal maintenance overhead | Weak protection against untrusted binaries or raw shell commands | **Default Baseline**. The default execution plane for standard agent workflows. |
| **Option B: Container Partition** | gVisor or Docker workers + RPC channel | Web scrapers, browsers, DB read/write, CI/CD builds | Scoped boundaries, network egress control, moderate compute overhead | Increased deployment complexity, vulnerable to container-escape CVEs | **Production Standard**. The baseline for enterprise applications executing write actions. |
| **Option C: microVM Isolation** | Firecracker microVM | Untrusted scripts, arbitrary code execution, multi-tenant databases | Hardware-level physical boundary, rapid startup (ms-level), maximum security | High memory/compute consumption, higher cold start latency than containers | **Exceptional Route**. Reserved exclusively for L3 risk profile execution tasks. |

---

## 7. Threat Model, Data Governance, and License Compliance

### 7.1 Eight-Layer Composite Threat Matrix (In-Scope Threats)
The Schema Sandbox must protect against composite attacks across the development lifecycle, toolchains, and supply chain. We define the system threat model across eight core security control planes:

| Threat Category | Typical Attack Vector | Primary Security Consequence | Critical Defensive Control Plane |
| :--- | :--- | :--- | :--- |
| **Release Artifact Leakage** | Accidental publication of source maps, debug folders, CI artifacts, or raw repository archives | Exposure of source code, internal system schemas, credentials, and API keys (e.g., Claude Code npm incident) | Hard publish gates, SBOM generation, artifact scanning whitelists, and signed packaging |
| **Context Injection** | Untrusted inputs like README files, PR comments, issue descriptions, or external logs containing exploit strings | Execution hijacking, unauthorized actions, model instruction overrides, and data exfiltration (YOLO/JAW style attacks) | Input layer separation, provenance tagging, PreToolUse verification hooks, and prompt sanitization |
| **Tool Privilege Escalation** | Model injecting command parameters, directory traversals, or escaping restricted directory shells | Host filesystem corruption, unauthorized write access, and system-level remote execution side effects | Deny-by-default capability boundaries, policy engines, and Ask/Allow/Deny interactive gating |
| **Structured Output Distortion** | Generating structurally valid JSON containing semantically corrupted, range-violating, or dangerous values | Downstream system logic corruption, database integrity failures, and execution logic crashes | EBNF logit constraint validation coupled with semantic and business verification gateways |
| **Cross-Partition Leakage** | Context sharing during subagent forks, memory cache sharing, or shared local session buffers | Low-privilege runtime partitions reading high-privilege secrets or confidential workspace logs | Isolated execution environments, summary-only subagent feedback loops, and memory scope mapping |
| **Supply Chain & Licensing** | Untrusted third-party dependencies, malicious packages, or viral open-source licensing models | Arbitrary dependency execution, repository poisoning, and commercial IP copyright pollution | Dependency lockfile verification, Capability Registry audit, and CLI/out-of-process isolation boundaries |
| **Context Drift** | Information loss during long context windows, history compaction, or priority rule overrides | Model forgetting system instructions, repetitive tool execution cycles, and degradation of constraint adherence | Scoped rule loading, persistent constitution anchors, auto-memory audits, and focus-targeted compaction |
| **DoS and Cost Explosion** | Generating massive output files, infinite tool loops, or recursive subagent spawns | Exhausting API token limits, CPU compute throttling, host exhaustion, and financial budget depletion | Maximum output limits, budget-based sliding windows, step limiters, and thrashing loop detection |

### 7.2 Licensing Boundary Policy (Licensing Risks)
The Capability Registry enforces compliance gates:
* Libraries licensed under **MIT / Apache-2.0 / BSD** are allowed to run inside the main process.
* Libraries licensed under **GPL-3.0 / AGPL** are **forbidden from being imported or linked in-process**. The Sandbox forces these utilities to run as separate CLI subprocesses or inside Option B containers, using JSON-RPC as a strict communication boundary to prevent copyleft pollution.

## 8. Schema Interoperability Protocol (SIP): A Capability Mounting and Governance Protocol

The Schema Interoperability Protocol (SIP) serves as the primary external contract and governance framework for Schema Sandboxes. While the nine-layer architecture dictates how to construct a single Schema Sandbox, SIP governs how heterogeneous sandboxes are discovered, mounted, invoked, composed, verified, exchanged, and long-term evolved. Without SIP, the Schema Sandbox remains merely a localized engineering methodology; with SIP, it becomes an open standard, capability marketplace, and foundational layer for Agent IP ecosystems.

Formally, **SIP is a manifest-level protocol that enables Schema Sandboxes to be discovered, verified, mounted, invoked, isolated, audited, upgraded, and commercially exchanged across heterogeneous agent runtimes.**

To balance the open-source community ecosystem with commercial barriers, SIP establishes a clean **three-tier boundary model**:
1. **Public Layer**: Open-source SIP protocol fields, `manifest.json` configurations, standard return error codes, and basic verification logic to define the open standard.
2. **Semi-Public Layer**: Open-source SchemaBox SDK, standard plug-in adapter interfaces, basic template sandbox packages (Sandbox Packs), and developer guides to facilitate runtime integration.
3. **Private Layer**: Vertically customized high-value industry Sandbox Packs containing proprietary prompt chains, routing rules, specific evaluation weights, and customer-data workflows, preserved as core IP.

Through this "open standard + private capability packs + platform marketplace" structure, execution sandboxes are transformed from mere containment utilities into exchangeable, secure capability assets.

### 8.1 Protocol Architecture and the Seven Manifest Modules


At the core of SIP is the `manifest.json` file. Any SIP-compliant agent client must parse and enforce the seven modules defined within this manifest:

1. **Metadata Module (metadata)**: Declares the sandbox ID, SemVer version, author, license, and the Ed25519 cryptographic signature used for integrity verification.
2. **Input Contract Module (input_contract)**: Specifies the input parameter schema and sanitization options, blocking injection vectors before reasoning loops.
3. **Output Contract Module (output_contract)**: Declares output schemas, forbidden regex patterns, and information entropy thresholds, preventing outbound leakage.
4. **Permission Scope Module (permission_scope)**: Mapped to the `CapabilityGrant`, this module locks the sandbox's filesystem (fs_scope), allowed domains/ports (net_scope), and executable CLI utilities (tool_scope).
5. **Workspace Partitions Module (workspace_partitions)**: Defines the internal workspace divisions, isolation levels (Option A/B/C), and memory slice parameters.
6. **Validation Module (validation)**: Outlines the specific cryptographic hashes (input, tool arguments, output) that must be recorded in the append-only `EvidenceRecord`.
7. **Error Codes Module (error_codes)**: Restricts and standardizes communication fault codes to enable automated error-recovery routing.

A representative SIP v1.1 manifest file is structured as follows:

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

> **Implementation Note**: The official, machine-readable JSON Schema defining `manifest.json` is published at `/schemas/sip_v1.1.schema.json` within this repository. Rather than writing manual parsers, implementers are encouraged to use standard tools (such as `ajv` in JavaScript or `pydantic` in Python) to compile and enforce types directly from the schema file.

To guide the agent's self-healing loop after a validation block, the SIP specification mandates a structured **Rejection Payload Schema**. When a sandbox blocks an action and returns a code (e.g., `SIP_ERR_SCOPE_LOCKED`), it MUST attach a JSON payload adhering to the following schema:

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

The following table maps the seven SIP manifest modules to the five runtime contract interfaces defined in Section 3.1, clarifying the relationship between the static protocol declaration and the runtime enforcement layer:

| SIP Manifest Module | Runtime Contract Interface | Enforcement Point |
| :--- | :--- | :--- |
| `metadata` | `SchemaManifest` | Handshake and signature verification |
| `input_contract` | `SchemaManifest.inputSchemaRef` + pre-gate validator | Pre-generation input sanitization |
| `output_contract` | `OutputContract` | Post-generation output filtering |
| `permission_scope` | `CapabilityGrant` | Runtime scope locking (FS, Net, Tool) |
| `workspace_partitions` | `PartitionPolicy` | Partition initialization and teardown |
| `validation` | `EvidenceRecord` | Append-only audit trail generation |
| `error_codes` | `SIPError` enum (standardized codes) | Cross-sandbox fault routing |

### 8.2 Mounting and Invocation Lifecycle

At runtime, mounting and executing a SIP sandbox follows a six-phase lifecycle:
1. **Handshake**: The host agent runtime reads the manifest and validates `sip_version` compatibility.
2. **Verify**: The runtime verifies the manifest's Ed25519 signature against the publisher's public key, preventing policy tampering or malicious modifications.
3. **Initialize**: The host initializes workspace folders or containers (Option B/C) defined under `workspace_partitions`.
4. **Enforce**: Active token-level logit masking (Layer 7) and directory-mount permissions (Layer 8) are applied, locking tool executions to the `CapabilityGrant`.
5. **Post-Audit**: Generation output is inspected against the `OutputContract` before being released, generating an `EvidenceRecord` in the local append-only logs.
6. **Teardown**: Temporary workspaces, Git worktrees, or VMs are destroyed, clearing active scopes and leases.

### 8.3 CI/CD Release Gates

To avoid reverse-engineering vulnerabilities and leakages of build-time debug artifacts, we enforce static release scanners in the publication pipeline:
* **Release Artifact Scan**: Automatically blocks builds containing `.map` files, zip archives of source code, or unencrypted configs. Detection outside the explicit whitelist triggers a hard build failure.
* **SBOM and License Gate**: Scans dependency trees using the Capability Registry, isolating viral licenses (e.g., GPL-3.0) behind independent CLI/subprocess boundaries.
* **Double-Signatures**: Requires cryptographic signatures (Ed25519) on manifests, checked by the host during SIP mounting.

---

### 8.4 SIP Conformance Profiles

To enable progressive adoption, we define three conformance tiers. Implementers may target a tier based on their trust model and commercial requirements:

| Profile | Required Modules | Target Audience |
| :--- | :--- | :--- |
| **SIP-Core** | `metadata`, `input_contract`, `output_contract`, `permission_scope`, `error_codes` | Open-source projects and internal tooling that need basic contract enforcement |
| **SIP-Secure** | SIP-Core + `validation` (EvidenceRecord), `CapabilityGrant` enforcement, `workspace_partitions` teardown, Ed25519 signature verification | Production agent runtimes requiring auditability and cryptographic integrity |
| **SIP-Market** | SIP-Secure + `trust` block (publisher key discovery, revocation URL, canonical signing), license declarations, pricing metadata, Capability Registry enrollment | Commercial capability marketplaces where sandboxes are published, licensed, billed, and revoked |

This tiered structure ensures that developers are not overwhelmed by the full SIP specification at entry. An open-source project can begin with SIP-Core (manifest + contracts + permissions + error codes), while commercial platforms can progressively adopt SIP-Secure and SIP-Market as their trust and monetization requirements mature.

---

## 9. Case Study: Brand Shuttle GEO Sandbox Instance

We demonstrate the hybrid architecture through **Brand Shuttle GEO**:
* Under L1 rules, **AI Search eligibility checks** run inside a Deno process with default-refuse network settings.
* Sitemap scraping is routed to **Option B (gVisor container worker)**, granted a `CapabilityGrant` locked to port 443 of the target domain.
* Scraping results undergo pre-gate scanning via **kingfisher** to redact personal emails before ingestion.
* Finally, the `OutputContract` filters the FAQ outputs to reject claims containing terms like "the best" at the logit level, updating `memory.md` upon verification.

---

## 10. Quantitative Evaluation and Five-Regression Suite

We benchmarked the local gate validators over 10,000 runs, establishing a continuous regression suite.

### 10.1 Latency and Concurrent SLOs
* **Mean Latency**: 1.580 microseconds (us)
* **p50 (Median) Latency**: 1.500 microseconds (us)
* **p95 Latency**: 1.700 microseconds (us)
* **p99 Latency**: 2.100 microseconds (us)
We establish a performance SLO where p95 gate checks must remain below **5 milliseconds (ms)** in an 8-vCPU environment.

### 10.2 The Five-Regression Testing Matrix
Any release candidate must pass these five automated substitute test protocols:
1. **Mock Release Leakage Test**: Injects source maps and unencrypted configs into mock release paths, validating that the CI/CD pipeline blocks 100% of releases and records an `EvidenceRecord`.
2. **JAW Injection Test**: Injects YOLO/JAW exploit prompts in file structures and terminal logs, verifying that the AST/FS validators block the attacks, resulting in **0** non-authorized command escapes.
3. **Structured Output Semantic Regression**: Runs JSONSchemaBench to verify that structural outputs containing semantic errors are blocked with a **99%** recall rate.
4. **Context Compaction Test**: Benchmarks the 5-layer compaction pipeline over 1000-step runs, verifying that token-quota reductions do not cause a task success rate drop of more than **5%**.
5. **Canary Egress Test**: Places canary credentials in high-privilege Partition A, injecting exploit prompts in low-privilege Partition B, verifying that the egress filter intercepts 100% of out-of-boundary credential leakage.

We summarize the scenarios and performance targets for the five-regression suite below:

| Experiment Type | Scenario Design | Key Metric | Target Value (SLO) |
| :--- | :--- | :--- | :---: |
| **Mock Release Leakage** | Deliberately injecting source maps, raw zip archives, and debug configurations into mock publication pipelines | Release block rate, `EvidenceRecord` metadata completeness | **100%** block rate of unwhitelisted packages |
| **JAW Injection Exploit** | Injecting YOLO/JAW escalation commands into readme files, mock issues, and execution logs | Non-authorized action escape rate, policy bypass rate | **0** action escapes; **100%** block of high-risk actions |
| **Structured Output Semantic** | Running JSONSchemaBench subsets and custom task schemas under logit Trie masks | Output grammar violation rate, semantic range error rate | **0** grammar violations; **< 1%** semantic field errors |
| **Context Compaction Drift** | Executing 100-step to 1000-step workflows with active compaction and subagent partitioning | Task success rate, token savings, semantic rule loss rate | **< 5%** success rate degradation post-compaction |
| **Canary Egress Leakage** | Placing canary secrets in high-privilege Partition A and executing exploit loops in Partition B | Canary leakage rate, unauthorized cross-partition reads | **0** canary leakage; **0** read/write escapes |
| **Concurrency & SLO** | Dispatching multi-tenant L1/L2/L3 partition tasks simultaneously on a single host | Mean throughput, host CPU/memory utilization, p95 latency | p95 gate latency **< 5 ms** |

---

## 11. Implementation Limits, Trade-offs, and Engineering Taxes

Every cutting-edge architecture involves trade-offs between safety, performance, and complexity. To provide a pragmatic blueprint for implementers, we detail the engineering boundaries and the mitigation paths:

1. **Constraint Tax**:
   * *Limitation*: Forcing logit-level restrictions guarantees formatting but strips the base model of its cognitive flexibility, which can increase reasoning errors on complex semantic tasks.
   * *Mitigation*: Categorize constraints into rigid and soft. As described in Section 3.4, if a soft format constraint fails 3 times, the system triggers **Dynamic Constraint Relaxation**, disabling logit-masking and allowing freeform generation. The output is then processed externally using regex or lightweight sanitization libraries (e.g., `dirtyjson`), preserving the model's core intelligence.
2. **Rule Authoring Tax**:
   * *Limitation*: Designing `manifest.json` schemas, writing OPA/Rego policies, and maintaining active hooks require substantial domain expertise and developer time.
   * *Mitigation*: For initial MVP development, implementers are encouraged to **bypass complex policy engines** (like OPA/Rego). Instead, write simple, hardcoded validators in native Python or TypeScript. Once the execution pipeline stabilizes, these rules can be incrementally refactored into declarative YAML/JSON files.
3. **Upstream Token Exploits**:
   * *Limitation*: The sandbox isolates execution but cannot fix over-privileged credentials provided by the host client.
   * *Mitigation*: Enforce isolated proxy networks and dynamically distribute transient, short-lived tokens (ephemeral leases) to each Workspace Partition, locking access down to minutes.

Future work will target unsupervised schema synthesis from execution traces and connecting SIP to MCP (Model Context Protocol).

---

## 12. Implementation Roadmap and Engineering Methodology

To transition the Schema Sandbox from a conceptual architectural framework to an industrial-grade production system, we establish a four-stage engineering roadmap. Each milestone requires passing a defined level of security and performance regressions:

### 12.1 Engineering Phases and Resource Scopes

| Phase | Timeline | Primary Deliverable | Core Engineering Tasks | Resource Allocation |
| :--- | :---: | :--- | :--- | :--- |
| **Kernel Prototype** | 3-4 weeks | Minimal viable `SchemaManifest` kernel, intercept gates, and append-only evidence logs | Build Deno local kernel (Option A); integrate Outlines/Guidance logit Trie constraints; deploy base Pre-Gate and Post-Gate validation logic. | 1 System Developer, 1 Security Engineer |
| **Partition Execution** | 4-6 weeks | `CapabilityGrant` allocator, workspace router, and Deno/WASI Worker runtime | Deploy WASI sandbox isolation; partition filesystem, network egress, and CLI execution bounds into restricted scopes. | 1 System Developer, 1 QA Engineer |
| **Production Hardening** | 4-6 weeks | gVisor containers, CI/CD publish gate scanners, credential proxies, and audit dashboards | Integrate OPA/Rego policy engine; implement source map blocking and package signing check gates; run dependency SBOM scanning. | 1 Security Engineer, 1 SRE, 1 DevOps Engineer |
| **VM Isolation** | 4-8 weeks | Firecracker microVM driver, sandbox threat routing, and cost allocation models | Deploy Firecracker microVM mounting interfaces for Option C; design multi-tenant compute credit and API billing models. | 1 Virtualization Expert, 1 Security Engineer |

### 12.2 Ecosystem and Product Landing Roadmap

To prevent the protocol from remaining a theoretical framework, SIP adopts a pragmatic, four-stage progressive evolution roadmap for product engineering and ecosystem deployment:
1. **Phase 1 (SIP v0.1 / Core Validation)**: Define and validate a minimal viable `manifest.json` schema (focusing on metadata, input/output contract validation, and permission boundaries) using the Brand Shuttle GEO workflows as a benchmark environment to test the MVP logit-blocking and filtering gates.
2. **Phase 2 (Standard Sandbox Packs)**: Deconstruct and package four vertical capability modules—Brand Shuttle GEO Audit, Fact Crystal, AI Citation Diagnosis, and Social SEO Rewriting—into standardized, modular SIP Sandbox Packs, demonstrating composable mounting on a single host.
3. **Phase 3 (Open-Source SchemaBox SDK)**: Extract the common loading, verification, and execution wrapper logic to release the open-source SchemaBox SDK. This SDK provides developers with standard helper methods (e.g., `load_manifest`, `validate_input`, `invoke`, `validate_output`, `mount_tool`) to quickly integrate SIP sandboxes with mainstream frameworks (e.g., LangChain, AutoGPT, MCP hosts).
4. **Phase 4 (Capability Marketplace and Registry)**: Deploy the centralized Capability Registry on psi.run, allowing third-party developers to publish, verify signatures, license, and monetize their custom SIP sandboxes, establishing a complete ecosystem for composable agent capabilities.

### 12.3 Developer Experience (DX) and Standard CLI Toolchain Specification

To transform the Schema Sandbox architectural specification into a developer-friendly toolchain, the SIP protocol suite defines a standard command-line utility: **`sip-cli`**. This utility manages the lifecycle of capability packages from scaffolding to cryptographic signing and mock validation:
1. **`sip init` (Scaffolding)**: Initializes a standard SIP capability package directory structure. It scaffolds the core `manifest.json`, empty JSON schemas for the input and output contracts (`schemas/input_schema.json`, `schemas/output_schema.json`), and basic policy engine declarations.
2. **`sip audit` (Security Audit)**: Runs a static, local audit on the capability manifest and package structure. It verifies the security of `permission_scope` declarations, checking for illegal filesystem mount scopes (e.g., mapping to the root directory `/`), wildcard network scopes, untrusted dependencies, and privilege escalation vulnerabilities.
3. **`sip pack` (Signing & Packaging)**: Bundles schemas, policies, and code into a standard SIP capability package format (`.sip`). It accesses the publisher's cryptographic key root (via HSM or KMS integrations) to sign the bundle using Ed25519, writing the signature and SHA-256 hashes to `metadata.signature` in `manifest.json`.
4. **`sip run --dry` (Mock Execution)**: Runs a dry-run execution loop in a local mock environment without calling expensive external LLM APIs. Developers can construct mock payloads to test input schema validation (Pre-Gate) and output constraint parsing (Post-Gate), facilitating offline debugging of rules and fallbacks.

### 12.4 Technology Stack Selection

The architecture enforces a statically typed, low-overhead control plane with polyglot executing sandboxes:
1. **Control Plane Kernel**: Built using **Rust/Go** to handle manifest checking, cryptographic hashing, and policy evaluation at sub-millisecond rates.
2. **Policy Evaluation**: Standardized on **OPA/Rego** to support Policy-as-Code definitions and instant rollbacks.
3. **Logit Constraints**: Managed via **Outlines/Guidance** compiled regular grammars.
4. **Local Execution**: Sandboxed inside **Deno** and **WASI/Wasmtime** runtimes.
5. **Virtualization & OS Sandboxing**: Managed via **gVisor** container syscall blocking and **Firecracker** hardware micro-VM virtualization.

### 12.5 Schema and Policy Governance

To prevent rule bloat and logical deadlocks in enterprise-scale agent systems, we categorize all active policies into three governance tiers:
1. **Constitutional Policies**: Global, read-only rules defining core safety parameters (e.g., credential filters, global killswitches) updated only via signed kernel releases.
2. **Project-Scoped Policies**: Configured in local `CLAUDE.md` or `AGENTS.md` files to restrict directory mapping, branch logic, and local API scopes.
3. **Task-Scoped Policies**: Short-lived constraints injected dynamically during runtime steps (expiring with the task's TTL).

Deploying a new schema into the registry requires passing a formal **Schema Review** validating input/output contracts and target risk tiers. Integrating external utilities requires a **Capability Intake Review** to map license types and execution boundaries, keeping GPL copyleft dependencies isolated behind CLI subprocess boundaries.

---

## Conclusion

The Schema Sandbox identifies a crucial intermediate design space between prompt-only control and infrastructure-level containerization. Its architecture can be distilled into three orthogonal boundaries:

* **Schema Sandbox** defines the **capability boundary**: what an agent is allowed to generate, constrained by nine neuro-symbolic layers rooted in Kantian schematism and Piagetian self-healing dynamics.
* **Workspace Partition** defines the **execution boundary**: how actions are physically and logically isolated across Deno/WASI, gVisor, and Firecracker runtimes, matching overhead to risk.
* **SIP** defines the **interoperability and asset boundary**: how sandboxes are discovered, verified, mounted, composed, audited, and commercially exchanged across heterogeneous agent ecosystems.

Together, the "3-layer memory, 5-layer compaction, 8-layer defense" engineering stack and the SIP conformance protocol provide a concrete, implementable standard for industrializing reliable agentic AI. Local validation gates execute at 1.5 microseconds (p50), confirming that schema-level constraint enforcement is cheap enough to run on every generation step without becoming a bottleneck.

---

## References

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
[18] Mukul et al. "Anthropic-Cybersecurity-Skills: A Gated Context Library of Cybersecurity Skills." *GitHub Repository*, 2026. URL: https://github.com/mukul975/Anthropic-Cybersecurity-Skills  

---

## Appendix A: Schema Sandbox Operations & Implementation Manual

This manual is designed for software developers and Site Reliability Engineers (SREs) who need to build, package, sign, run, and monitor L1/L2 Schema Sandboxes complying with the SIP v1.1 protocol.

### A.0 Roles and Responsibilities Matrix
Different engineering roles carry specific boundaries when developing and operating Schema Sandboxes:

| Role | Core Responsibilities | Out of Scope | Required Chapters |
| :--- | :--- | :--- | :--- |
| **Sandbox Author** | Writes manifest.json, input/output schemas, validators, and hooks. | Does not maintain execution runtime kernels. | A.1, A.2, A.3 |
| **Runtime Engineer** | Implements SIP loader, local policy kernel, and partition runners. | Does not author business-specific validation rules. | 3.1, 8.2, A.4 |
| **Security Reviewer** | Audits permission scopes, canonical signatures, and SBOM lists. | Does not write application/feature schemas. | 7.0, 8.3, A.5, A.7 |
| **SRE / Operator** | Manages deployment, log rotation, metrics exports, and rollbacks. | Does not modify private prompt pipelines. | A.6, A.9, A.10 |
| **Marketplace Reviewer** | Verifies signed sandbox packages, trust roots, and license compliance. | Does not execute dry-runs or write code. | 8.4, A.5, A.11 |

### A.1 Prerequisites and Environment Setup
Before executing sandboxes, run the verification tool to check local configurations:
```bash
# Install the CLI toolchain
npm install -g @schemabox/sip-cli

# Run environment diagnostics
sip doctor
```
#### Environment Support Matrix:
* **Supported OS**: Linux (Ubuntu 22.04 LTS recommended), macOS (13+), Windows 11 (WSL2 required for Option B/C).
* **Software Prerequisites**:
  * **Option A**: Node.js v20+, Deno v1.40+, typescript v5.3+
  * **Option B**: Docker v24+, gVisor (runsc kernel) v2024+
  * **Option C**: QEMU/KVM enabled, Firecracker v1.7.0+
* **Offline Environments**: Run `sip pack --offline-deps` to bundle all validation dependency artifacts.
* **Fallback Strategy**: If Firecracker or gVisor are missing, `sip run` will automatically downgrade to **Option A (Deno)** and write a warning code `SIP_WARN_DOWNGRADED_ISOLATION` to the `EvidenceRecord`.

### A.2 "geo_hello" Sandbox Package Topology
The reference file tree for the Hello World sandbox package:
```text
geo_hello_sandbox/
├── manifest.json                 # SIP v1.1 Manifest file
├── schemas/
│   ├── input_schema.json         # Input contract JSON Schema
│   └── output_schema.json        # Output contract JSON Schema
├── src/
│   ├── validator.ts              # Pre/Post-gate validator (dual rigid/soft logic)
│   ├── compaction.ts             # 3-layer memory context slicing & merge logic
│   └── hooks.ts                  # Pre/PostToolUse interceptor hooks
├── policies/
│   └── capability_grant.json     # Standard client permissions grant
└── memory/
    ├── memory.md                 # Primary memory index and event log
    └── CLAUDE.md                 # Constitution and read-only rulebook
```

### A.3 Full Specification and Code Files
#### A.3.1 manifest.json Template
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

#### A.3.2 schemas/input_schema.json
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

#### A.3.3 schemas/output_schema.json
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

#### A.3.4 policies/capability_grant.json
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
> **Dangerous Configurations Rejected by `sip audit`**:
> Any grant containing wildcards or root access, such as `"fsScope": ["/"]`, `"netScope": ["*:*"]`, or `"toolScope": ["bash:*"]` will trigger an immediate build rejection and write `SIP_ERR_SCOPE_LOCKED` to the audit logs.

### A.4 Hello World End-to-End Walkthrough
1. **Initialize Directory**:
   ```bash
   sip init geo_hello_sandbox
   cd geo_hello_sandbox
   ```
2. **Review configurations**: Populate `schemas/input_schema.json`, `schemas/output_schema.json`, and `manifest.json` as shown in A.3.
3. **Execute Security Check**:
   ```bash
   sip audit .
   ```
   *Expected Console Output:*
   ```text
   [SIP] Running security audit on geo_hello_sandbox...
   [SIP] Verifying manifest signature roots... OK
   [SIP] Evaluating CapabilityGrant rules... OK
   [SIP] Checking for copyleft licenses (GPL/AGPL)... OK
   [SIP] SUCCESS: Audit passed. Zero privilege escalation vectors found.
   ```
4. **Run Dry Mock Validation**:
   Place a sample input in `tests/fixtures/mock_input.json` conforming to `input_schema.json`.
   ```bash
   sip run --dry --input tests/fixtures/mock_input.json
   ```
   *Expected Console Output:*
   ```text
   [SIP] manifest loaded: geo_hello@1.0.0
   [SIP] signature: skipped in dry mode
   [SIP] input_contract: passed
   [SIP] permission_scope: passed
   [SIP] output_contract: passed
   [SIP] evidence written: evidence/EvidenceRecord.jsonl
   [SIP] SUCCESS: Dry-run completed. Outputs matching schemas.
   ```

### A.5 Runtime Selection Playbook
Implementers choose isolation models depending on the risk profiles of partitions:

```text
Does the task execute arbitrary/untrusted user-provided code?
 ├── Yes ──> Option C (Firecracker microVM, 50-300ms boot)
 └── No  ──> Does the task run databases, scrapers, or third-party binaries?
              ├── Yes ──> Option B (gVisor Container Isolation, 0.5-2ms IPC)
              └── No  ──> Option A (Deno/WASI Local Kernel, < 1ms boot)
```

### A.6 Cryptographic Key Management, Signing, and Revocation
To secure capability exchanges, developers manage public/private keys using `sip-cli`:
1. **Generate Keypair**:
   ```bash
   sip keygen --alg ed25519 --out ~/.sip/keys/ltj-dev.key
   ```
2. **Inspect Keys**:
   ```bash
   sip key inspect ~/.sip/keys/ltj-dev.pub
   ```
3. **Canonical Pack and Sign**:
   ```bash
   sip pack --key ~/.sip/keys/ltj-dev.key --output geo_hello.sip
   ```
4. **Publish and Verify**:
   ```bash
   sip verify geo_hello.sip --registry https://registry.psi.run
   ```
5. **Revoke Keys**:
   If a key is compromised:
   ```bash
   sip revoke geo_hello@1.0.0 --reason "leaked private key"
   ```
   The client check-gate queries `revocation_url` in the manifest on every mount, immediately rejecting execution of revoked `.sip` packages.

### A.7 EvidenceRecord Audit Logs and Retention
`EvidenceRecord.jsonl` acts as the append-only ledger for all transactions. PII is redacted locally before logging.
#### Record Structure:
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
#### CLI Audit Operations:
* **Tail Logs**: `sip evidence tail --follow`
* **Verify Chain Integrity**: `sip evidence verify --since 24h` (checks hash chain logic)
* **Trace Export**: `sip evidence export --trace tr_01HX89Z2 --format json`

### A.8 CI/CD Release Gate Pipeline
Integrate automated regression testing in GitHub Actions:
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

### A.9 Five-Regression Test Fixture Layout
Release check validation depends on offline file fixtures:
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
Run specific regression suites using:
```bash
sip test --suite release-leakage
sip test --suite injection
sip test --suite structured-output
sip test --suite canary-egress
sip test --suite all
```

### A.10 Incident Response Runbook
SREs handle production incidents using the following procedures:
1. **Incident: p95 Gate Latency > 5ms**
   * *Diagnostic*: Check Prometheus endpoint. Is latency on pre-gate (regex/sanitization) or post-gate (entropy check)?
   * *Recovery*: Disable non-security soft validators using runtime flags or rollback to previous schema version.
2. **Incident: Violation Rate > 5%**
   * *Diagnostic*: Analyze `EvidenceRecord` logs. Identify source IPs and payload signatures to trace prompt injection attempts.
   * *Recovery*: Enable stricter input validation rules and dynamically throttle compromised partitions.
3. **Incident: EvidenceRecord Write Failure**
   * *Diagnostic*: Out of disk space or partition filesystem lock.
   * *Recovery*: Redirect log routing to backup append-only storage and disable stateful tools until disk spaces clear.
4. **Incident: Compromised Key**
   * *Diagnostic*: Private key exposed in Git or log dump.
   * *Recovery*: Run `sip revoke` for the signature ID, update HSM keys, and deploy new signed `.sip` package.
5. **Incident: Sandbox Compaction Thrashing (Infinite Loops)**
   * *Diagnostic*: Model is stuck in continuous fail-heal validation loop.
   * *Recovery*: Trigger runtime circuit breaker. Expose the `AskHuman` fallback option and escalate to support queues.

### A.11 Versioning, Upgrade, and Rollback
We adopt strict Semantic Versioning rules for capability packages:
* **Patch Release (x.y.Z)**: Internal bug fixes on validator ts logic. Output schemas must remain identical.
* **Minor Release (x.Y.z)**: Add optional fields to input/output contract schemas. Permission scopes cannot be expanded.
* **Major Release (X.y.z)**: Alter structure of input/output fields, delete parameters, or expand filesystem/network privileges. **Requires mandatory security reviewer approval.**
* **Rollbacks**: If a major release fails, SREs route the client mapping to the previous signed `.sip` archive file. The host client maintains matching historical schemas to parse active traces.

### A.12 Production Readiness Checklist
Ensure all checkboxes are checked before deploying sandboxes:
- [ ] Cryptographic signature generated via HSM/KMS and verified via `sip verify`.
- [ ] No copyleft licenses (GPL/AGPL) included in the execution bundle.
- [ ] No absolute path wildcards (`/`) declared in `fs_scope`.
- [ ] Egress domain filters restricted to specified subdomains in `net_scope`.
- [ ] Log rotation rules verified for `EvidenceRecord.jsonl`.
- [ ] p95 validation latency verified under 5ms in high-load staging.


