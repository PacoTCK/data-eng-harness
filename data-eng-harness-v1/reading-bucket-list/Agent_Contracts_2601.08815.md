# Agent Contracts: A Formal Framework for Resource-Bounded Autonomous AI Systems

**Qing Ye¹, Jing Tan²**
¹ Independent Researcher — yeqi519@gmail.com
² Independent Researcher — jtan@live.de

arXiv:2601.08815v3 [cs.MA] · 25 Mar 2026
*Accepted for oral presentation at COINE 2026 (16th International Workshop on Coordination, Organizations, Institutions, Norms and Ethics for Governance of Multi-Agent Systems), co-located with AAMAS 2026, Paphos, Cyprus.*

License: CC BY 4.0
Fuente: https://arxiv.org/abs/2601.08815

---

## Abstract

The Contract Net Protocol (1980) introduced coordination through contracts in multi-agent systems. Modern agent protocols standardize connectivity and interoperability, yet none provide formal resource governance—normative mechanisms to bound *how much* agents may consume or *how long* they may operate. We introduce *Agent Contracts*, a formal framework that extends the contract metaphor from task allocation to resource-bounded execution. An Agent Contract **C = (I, O, S, R, T, Φ, Ψ)** unifies input/output specifications, multi-dimensional resource constraints, temporal boundaries, and success criteria into a coherent governance mechanism with explicit lifecycle semantics. For multi-agent coordination, we establish conservation laws ensuring delegated budgets respect parent constraints, enabling hierarchical coordination through contract delegation. Empirical validation across four experiments demonstrates 90% token reduction with 525× lower variance in iterative workflows, zero conservation violations in multi-agent delegation, and measurable quality-resource tradeoffs through contract modes. Agent Contracts provide formal foundations for predictable, auditable, and resource-bounded autonomous AI deployment.

---

## 1. Introduction

In late 2025, an engineering team deployed a multi-agent research system with four specialized agents. Two agents fell into a recursive clarification loop, running undetected for eleven days. When the invoice arrived, the team discovered a $47,000 API bill. The system had no stop conditions, no budget limits, and no real-time cost monitoring. This incident encapsulates a fundamental problem—we have built AI agents capable of autonomous action but lack formal mechanisms to bound their behavior.

Such failures reflect systemic gaps, not implementation bugs. Gartner predicts that over 40% of agentic AI projects will be canceled by 2027 due to escalating costs or inadequate risk controls, even as agentic AI is projected to appear in 33% of enterprise software by 2028. A recent MIT Sloan study finds that 35% of organizations already deploy agentic AI, with leaders citing the tension between *supervision and autonomy* as a core challenge requiring centralized governance infrastructure. Agents are becoming capable of sustained autonomous operation spanning hours or days, yet the protocols governing them address *connectivity* and *interoperability* but not *resource governance*—how much an agent may consume or how long it may operate.

Agent Contracts address this gap by extending the contract metaphor from task allocation to resource governance. Where the Contract Net Protocol asks "who should do this task?", Agent Contracts ask "within what bounds may this task be performed?" We draw on contract theory from economics, coordination theory from distributed systems, and resource-bounded computation from real-time systems to form a formal framework for resource-bounded autonomous AI.

This paper makes two contributions. First, we define an Agent Contract as a formal tuple **C = (I, O, S, R, T, Φ, Ψ)** that unifies input/output specifications, resource constraints, temporal boundaries, and success criteria into a coherent governance mechanism. Second, we establish conservation laws for multi-agent systems that ensure budget discipline across delegation hierarchies, enabling composable coordination patterns where contracting itself becomes an agent capability. We validate these contributions across four experiments demonstrating 90% token reduction in iterative workflows, zero conservation violations in multi-agent delegation, and measurable quality-resource tradeoffs. Together, these contributions provide formal foundations for explicit resource governance in autonomous AI systems.

---

## 2. Theoretical Foundations

Agent Contracts draw on three theoretical traditions: contract theory from economics, multi-agent coordination from computer science, and resource-bounded computation from real-time systems.

**Contract Theory.** Bolton and Dewatripont (2005) formalize how agreements are constructed under asymmetric information. Three concepts apply to agent governance: *moral hazard* (hidden actions; in LLM agents, unpredictable resource consumption), *incomplete contracts* (separating success criteria from execution strategies), and *mechanism design* (specifications that elicit desired behavior). Recent work identifies principal-agent dynamics in LLM systems but provides conceptual rather than operational frameworks.

**Multi-Agent Coordination.** Classical MAS research established foundations for agent coordination. The Contract Net Protocol demonstrated that explicit contracting could coordinate distributed systems. Coordination theory identifies fundamental problems (managing shared resources, producer-consumer relationships, and simultaneity constraints), each corresponding to challenges in multi-agent LLM systems. Research on normative multi-agent systems formalizes how norms govern agent behavior through obligations, prohibitions, and permissions. Closely related, social commitments endow agents with normative accountability: a committed agent that violates its obligation can be sanctioned. Agent Contracts operationalize the normative perspective for LLM systems: resource constraints function as prohibitions (agents *must not* exceed budgets), success criteria as obligations (agents *must* achieve quality thresholds), and the contract lifecycle provides regimented enforcement where violations trigger automatic termination. However, unlike classical committed agents, LLM agents are typically ephemeral—instantiated per task and discarded—making agent-level sanctions inapplicable. We address this asymmetry through structural enforcement (Section 7.2).

A key insight from MAS theory is that coordination mechanisms must respect *conservation laws*—resources allocated to subtasks cannot exceed parent resources. This principle requires explicit enforcement in LLM systems where token consumption is stochastic and observable only after the fact.

**Resource-Bounded Computation.** Simon's theory of bounded rationality established that agents with limited cognitive resources must *satisfice* rather than maximize. Agent Contracts operationalize satisficing by defining acceptable quality thresholds within resource budgets.

The algorithmic foundation comes from contract algorithms, which specify computation budgets *before* activation. Unlike anytime algorithms that can be interrupted arbitrarily, contract algorithms enable strategic resource allocation. An Agent Contract transforms an LLM agent into a contract algorithm where bounds **R** are known in advance. Real-time systems theory contributes the distinction between hard constraints (violation causes termination) and soft constraints (permitting graceful degradation).

---

## 3. Related Work

### 3.1 Agent Architectures and Coordination Protocols

The development of LLM-based agents has accelerated rapidly since 2022. ReAct introduced the paradigm of synergizing reasoning and acting. Chain-of-Thought prompting established that computational depth correlates with output quality, motivating explicit resource governance. Toolformer showed that language models can learn to use external tools, expanding the resource consumption profile of agents. Fully autonomous systems like AutoGPT and Generative Agents revealed both the potential and governance challenges of unbounded execution.

Coordination protocols have evolved to address different concerns. The Contract Net Protocol established task allocation through bidding; MCP standardizes tool connectivity between models and external resources; A2A enables discovery and interoperability across heterogeneous agent systems. The recent formation of the Agentic AI Foundation under the Linux Foundation—with contributions including MCP, OpenAI's AGENTS.md, and Block's goose—signals industry consensus on connectivity and documentation standards. However, resource governance remains outside this scope; none of these initiatives formalize how much agents may consume.

### 3.2 Budget-Aware Reasoning and Resource Management

A growing body of work addresses resource efficiency in LLM reasoning. The TALE framework introduces token-budget-aware reasoning, achieving 68% reduction in token usage with less than 5% accuracy degradation. Critically, the authors identify "token elasticity": LLMs often exceed specified budgets when constraints are tight, demonstrating that prompting alone is insufficient for strict enforcement. BudgetThinker addresses this through control tokens injected during inference, coupled with reinforcement learning to achieve precise budget adherence. SelfBudgeter enables models to predict required token budgets based on task complexity. Liu et al. (2025) demonstrate that simply granting larger tool-call budgets fails to improve agent performance; their Budget-Aware Tool Selection (BATS) framework shows that explicit budget awareness enables effective scaling.

Snell et al. (2024) provide a budget-aware evaluation framework, demonstrating that when compute is equalized, sophisticated reasoning strategies often do not outperform simpler baselines; much apparent improvement comes from using more resources rather than using them more intelligently. This finding underscores the importance of explicit resource accounting.

Infrastructure-level resource management has matured separately. LLM serving systems optimize throughput and memory efficiency at the inference layer. LLMOps platforms provide budget tracking, alerting, and rate limiting at the organizational level. However, these operate below or above the application layer; neither provides formal contracts that govern individual agent behavior within multi-agent workflows.

### 3.3 Agent Safety and Formal Verification

Foundational AI safety work identifies problems including reward hacking and safe exploration. Recent work calls for responsible LLM-empowered multi-agent systems, recognizing that uncertainties compound across agent interactions.

Formal verification approaches have begun addressing agentic AI specifically. Zhang et al. (2025) propose a modeling framework with 17 properties for host agents and 14 for task lifecycles, expressed in temporal logic. This work is complementary to Agent Contracts. Formal verification addresses *whether* systems satisfy properties; Agent Contracts specify *what* resource constraints systems must satisfy.

### 3.4 Multi-Agent Coordination Frameworks

LLM-based multi-agent systems have proliferated with frameworks addressing different coordination paradigms. MetaGPT integrates human workflow patterns into multi-agent collaboration, using standardized operating procedures to structure agent interactions. AutoGen frames coordination as asynchronous conversation among specialized agents, with each agent capable of responding, reflecting, or invoking tools based on message content. LangGraph provides graph-based orchestration with explicit state management and checkpointing, enabling durable execution of complex workflows. CrewAI takes a role-based approach where agents are assigned organizational roles (researcher, developer, etc.) with corresponding capabilities.

Recent surveys characterize the landscape systematically. Tran et al. (2025) analyze collaboration mechanisms across five dimensions: actors involved, interaction types (cooperative, competitive, or coopetitive), organizational structures, coordination strategies, and communication protocols. Additional surveys examine architectural patterns and how message-passing architectures affect coordination effectiveness. These taxonomies reveal sophisticated pattern vocabulary but consistently note that resource governance remains underdeveloped. Research on self-resource allocation demonstrates that LLMs can serve as effective resource allocators, with planner-based approaches outperforming real-time orchestration for concurrent task management. However, this work studies allocation *capability* rather than providing a formal governance *framework*.

To clarify this governance gap, Table 1 summarizes resource governance features across eight major agent frameworks. All provide operational controls (iteration limits, timeouts, and rate limiting), reflecting engineering best practices for preventing runaway execution. However, none provide the formal governance layer that Agent Contracts introduce: cost budgets, temporal deadlines, success criteria, or conservation laws for multi-agent delegation.

**Table 1.** Governance features across agent frameworks. Y = native, P = partial, – = none. *Italic*: operational controls. **Bold**: governance features unique to Agent Contracts. LG = LangGraph, AG = AutoGen, Crew = CrewAI, OAI = OpenAI Agents SDK, ADK = Google ADK, BR = Amazon Bedrock, LI = LlamaIndex, smol = smolagents. *AutoGen is being merged with Semantic Kernel into Microsoft Agent Framework.*

| Feature | LG | AG | Crew | OAI | ADK | BR | LI | smol |
|---|---|---|---|---|---|---|---|---|
| *Max iterations* | Y | Y | Y | Y | Y | Y | Y | Y |
| *Timeout* | Y | Y | Y | Y | Y | Y | Y | Y |
| *Rate limiting* | Y | P | Y | P | P | Y | P | Y |
| *Token limits* | P | Y | Y | Y | Y | Y | Y | Y |
| *Observability* | Y | Y | Y | Y | Y | Y | Y | Y |
| *Guardrails* | P | – | P | Y | Y | Y | P | – |
| **Agent Contract (cost budgets, deadlines, success criteria, conservation laws)** | – | – | – | – | – | – | – | – |

The table reveals a consistent pattern: existing frameworks provide operational controls but lack formal governance mechanisms. Recent practitioner perspectives frame "agent engineering" as iterative refinement for reliability, but do not address resource governance. The following section presents Agent Contracts as a framework that fills this gap.

---

## 4. The Agent Contract Framework

### 4.1 Contract Definition

An Agent Contract **C** is defined as a seven-tuple:

> **C = (I, O, S, R, T, Φ, Ψ)**

The components capture the complete specification for bounded agent execution. The input specification **I** defines the schema and constraints for acceptable inputs. The output specification **O** defines the schema and quality criteria for deliverables. The skill set **S** enumerates the capabilities (tools, functions, and knowledge domains) available to the agent. Resource constraints **R** specify a multi-dimensional budget governing consumption. Temporal constraints **T** establish time-related boundaries and duration limits. Success criteria **Φ** define measurable conditions for contract fulfillment. Finally, termination conditions **Ψ** specify events that end the contract regardless of fulfillment.

This formulation synthesizes contract theory in economics, where contracts align incentives and define obligations between parties, with real-time systems theory, where correctness depends on meeting explicit timing constraints. The contract serves as both specification (defining what the agent should do) and governance mechanism (constraining how the agent may operate).

An important distinction separates **I** from **R**. The input specification **I** defines *what* the agent receives: task content, context, and parameters. The resource constraints **R** define *how much* the agent may consume while processing: token budgets, API call limits, time bounds. An agent may receive a small input but consume many resources through complex reasoning, or receive a large input but consume few resources through simple transformation.

### 4.2 Contract Components

**Input and Output Specifications.** The input specification **I = (σ_I, V_I, P_I)** comprises the input schema σ_I, validation rules V_I, and preprocessing transformations P_I. For example, a code review agent might specify σ_I as `{repository: string, pr_id: integer}`, V_I as `pr_id > 0`, and P_I as `fetch_diff(pr_id)`.

The output specification **O = (σ_O, Q_min, F_O)** comprises the output schema σ_O, minimum acceptable quality threshold Q_min, and formatting requirements F_O. For the code review agent, σ_O might be `{summary: string, issues: list, approval: boolean}`, Q_min = 0.8 (requiring 80% of issues correctly identified), and F_O as markdown format with severity labels.

**Skills and Capabilities.** The skill set **S = {s₁, s₂, ..., sₘ}** where sᵢ ∈ S_available enumerates what the agent *can* do. Each skill sᵢ may have associated costs c(sᵢ) and success probabilities p(sᵢ). Skills encompass tool invocations (web search, code execution, API calls), knowledge domains (legal, medical, technical), and cognitive capabilities (reasoning, planning, summarization). Recent industry standards for agent skills employ *progressive disclosure*—loading metadata (~50 tokens) initially and full specifications (~500+ tokens) on-demand—reflecting resource-aware design even in capability definitions. The contract restricts the agent to skills in **S**; for example, an agent contracted for data analysis cannot invoke payment APIs even if technically accessible.

**Resource Constraints.** The resource constraint **R = {r₁, r₂, ..., rₙ}** defines a multi-dimensional budget. Common resource dimensions include:

| Resource | Symbol | Unit | Example |
|---|---|---|---|
| LLM Tokens | r_tok | tokens | 100,000 |
| API Calls | r_api | calls | 50 |
| Iterations | r_iter | rounds | 10 |
| Web Searches | r_web | queries | 10 |
| Compute Time | r_cpu | seconds | 300 |
| External Cost | r_cost | USD | 5.00 |

Each resource rᵢ has a budget bᵢ and consumption function cᵢ(t). Constraint satisfaction requires: for all i, cᵢ(t) ≤ bᵢ.

**Temporal Constraints.** The temporal constraint **T = (t_start, τ)** comprises the contract activation timestamp t_start and the contract duration (time-to-live) τ. The constraint requires t_current − t_start ≤ τ. While users often think in terms of deadlines ("complete by 5pm"), duration is the operational primitive—a deadline is simply t_deadline = t_start + τ.

**Success Criteria and Termination.** Success criteria **Φ = {(φ₁, w₁), (φ₂, w₂), ..., (φₖ, wₖ)}** pair measurable conditions φᵢ with weights wᵢ. The contract is fulfilled when Σ wᵢ · 𝟙[φᵢ] ≥ θ for threshold θ. Conditions may include task completion (`all_items_processed`), quality metrics (`accuracy > 0.95`), or business logic (`response_generated AND reviewed`).

The output quality threshold Q_min (in specification O) is a *structural requirement* on output format and minimum acceptability. Success criteria Φ are *fulfillment conditions* that may include quality checks (e.g., φ₁ = Q ≥ Q_min) along with other conditions. An agent might produce output meeting Q_min but fail Φ if other criteria are unmet.

Termination conditions **Ψ = {ψ₁ ∨ ψ₂ ∨ ... ∨ ψₗ}** define when the contract ends regardless of success. Common termination conditions include resource exhaustion (∃ rᵢ: cᵢ ≥ bᵢ), duration expiration (t − t_start > τ), explicit cancellation (external signal), and unrecoverable errors (critical failure states). The existential quantifier ensures that exceeding *any* resource budget causes termination, not just aggregate overruns.

### 4.3 Contract Lifecycle

Contracts progress through distinct states following the transition pattern **DRAFTED → ACTIVE → {FULFILLED, VIOLATED, EXPIRED, TERMINATED}**. A contract begins in the DRAFTED state, where its parameters are specified but execution has not begun. Activation transitions the contract to ACTIVE, at which point resources are reserved and monitoring begins. From ACTIVE, the contract reaches exactly one of four terminal states.

The terminal state FULFILLED indicates that success criteria Φ were satisfied within all resource and temporal constraints. VIOLATED indicates that some constraint in R or T was breached before success criteria were met. EXPIRED indicates that the duration τ was exceeded. TERMINATED indicates external cancellation, regardless of progress toward success criteria.

Transitions between states are governed by formal guard conditions:

| From | To | Guard Condition |
|---|---|---|
| DRAFTED | ACTIVE | activate() ∧ resources_available() |
| ACTIVE | FULFILLED | Σ wᵢ · 𝟙[φᵢ] ≥ θ |
| ACTIVE | VIOLATED | ∃ rᵢ: cᵢ ≥ bᵢ |
| ACTIVE | EXPIRED | t − t_start > τ |
| ACTIVE | TERMINATED | cancel_signal() |

The lifecycle model ensures clear accountability. Every contract reaches exactly one terminal state, enabling unambiguous resource release and audit logging.

---

## 5. Resource Tracking and Monitoring

The preceding chapter defined what contracts *are*: their structure, components, and lifecycle. This chapter addresses how resources are *tracked* during execution. While the framework tracks multiple resource types (tokens, API calls, tool invocations, compute time, cost), we focus here on two foundational aspects: how token budgets decompose into measurable categories, and how runtime monitoring provides visibility into constraint utilization.

### 5.1 Token Budget Decomposition

Modern LLMs distinguish input, reasoning, and output tokens. We model this as **R_tok = (r_in, r_r, r_out)**. Since input tokens r_in are largely determined by task context, the *controllable budget* is **B_ctrl = B_tok − r_in**, representing tokens available for reasoning and output. This decomposition enables fine-grained monitoring across categories, supporting both real-time adaptation and post-hoc analysis.

### 5.2 Runtime Monitoring

During execution, a monitoring system tracks both resource consumption and temporal progress in real-time. The monitor function **Monitor: (C, t) → (c⃗, u⃗, τ_util)** takes a contract C and current time t and returns three values: the resource consumption vector c⃗, the resource utilization vector u⃗ = c⃗ / b⃗ (computed element-wise), and the duration utilization τ_util = (t − t_start) / τ, ranging from 0 to 1. The agent can query these values at any time to adapt its strategy as constraints tighten. Note that utilization is monotonically non-decreasing since resource consumption is cumulative.

A useful aggregate metric captures the most-constrained resource:

> utilization(t) = max( (t_current − t_start) / τ , maxᵢ [cᵢ(t) / bᵢ] )

This single value summarizes how close the agent is to any constraint boundary, enabling simple threshold-based policies (e.g., "warn when utilization exceeds 80%") without requiring sophisticated optimization.

**Communicating Budget to Agents.** A current approach is *budget-aware prompting*: injecting remaining budget into system prompts or providing dynamic status updates during execution. This enables agents to self-regulate—producing concise outputs when budget is tight or taking exploratory actions when resources are ample. As the field matures, native solutions may emerge (e.g., models with built-in resource awareness), but prompt-based communication provides a practical mechanism with current infrastructure.

---

## 6. Multi-Agent Coordination Under Contracts

The preceding sections establish contracts for individual agents. However, complex tasks often require multiple agents working together: a researcher gathering data, an analyzer identifying patterns, a reporter synthesizing findings. This extension from single-agent to multi-agent governance raises new questions: How should a parent budget be divided among child agents? What happens when one agent exceeds its allocation while others remain under budget? How can the system guarantee that aggregate consumption respects the original constraint?

These questions have theoretical grounding in coordination theory. Malone and Crowston (1994) identified managing shared resources as a fundamental coordination problem. In LLM-based multi-agent systems, the shared resource is typically the token budget (and associated cost), but the same principles apply to API call limits, compute time, and other constrained resources. The key insight is that contracts provide a natural unit of delegation. When an orchestrator creates subcontracts for workers, the contract specification ensures that each worker operates within defined bounds, and the aggregate of those bounds respects the parent constraint.

### 6.1 Conservation Laws and Budget Allocation

When multiple agents collaborate, contracts must govern how resources flow between them. The fundamental constraint is conservation: total consumption cannot exceed the system budget:

> Σ (j ∈ agents) c_j^(r) ≤ B^(r), for all r ∈ R

This invariant holds regardless of execution pattern—sequential, parallel, hierarchical, or competitive.

**Initial Allocation.** Before execution, the total budget B must be divided among agents. Three strategies apply depending on available information:

- **Proportional:** b_j = ω_j / Σ ω_k · (B − B_reserve), where ω_j reflects estimated task complexity
- **Equal:** b_j = (B − B_reserve) / n, when complexity is unknown
- **Negotiated:** Agents request budgets; a coordinator allocates based on requests with caps to prevent over-claiming

A reserve buffer B_reserve (typically 10–15%) accommodates coordination overhead and unexpected costs.

**Dynamic Reallocation.** As agents complete, unused budget returns to a shared pool:

> B_available(t) = B_reserve + Σ (j ∈ completed) (b_j − c_j)

This enables *budget pooling*—efficient agents effectively subsidize those requiring more resources, improving overall throughput while maintaining total budget discipline.

### 6.2 Coordination Patterns Through a Contract Lens

Recent work has identified recurring design patterns for agentic AI systems, including routing, orchestration, parallelization, and iterative refinement.

We organize these patterns through a contract-centric lens, focusing on how resource constraints govern each pattern's behavior and where contracts provide critical safety guarantees. We focus here on two typical control flow patterns, task routing and delegation decisions, since these are where contracts add the most value. Execution within any pattern can be sequential or parallel; these are orthogonal concerns that compose naturally.

**Routing.** An input classifier directs requests to specialized handlers based on task characteristics, selecting the best-suited agent for each input and allocating the corresponding budget. Budget is reserved per potential branch; unused allocations return to the pool. This enables separation of concerns—specialized agents outperform generalists on their specific tasks.

When specialists have explicit contracts, routing becomes more principled: the router matches task requirements against specialist capabilities, resource profiles, and success criteria. Furthermore, with well-defined contracts, the router need not be limited to a fixed pool; it can dynamically instantiate or configure an agent specifically for the required contract. This blurs the line between routing and orchestration, with the contract serving as the specification for agent creation, not just agent selection.

**Orchestrator-Workers.** A central orchestrator dynamically decomposes tasks and delegates to worker agents, synthesizing results. This pattern can extend to multiple levels (hierarchical orchestration).

From a contract perspective, this pattern is particularly significant: the orchestrator drafts and issues subcontracts to workers. Each subcontract specifies the worker's task, allocated budget, and success criteria:

> orchestrator(C_parent) → {C_i = (I_i, O_i, S_i, R_i, T_i, Φ_i, Ψ_i)} for i = 1 to k

This frames contracting as a capability: the orchestrator must understand the contract framework to effectively delegate work. The conservation law (Section 6.1) constrains subcontract allocation: Σ R_i ≤ R_parent.

The implications of contracting as a capability are significant:

| Implication | Description |
|---|---|
| *Recursive delegation* | Agents spawn sub-agents that themselves have contracting capability, enabling hierarchical self-organization |
| *Bounded autonomy* | Even highly capable orchestrators remain governed by their parent contract: they can create subcontracts but cannot exceed their own constraints |
| *Dynamic team formation* | Agents form coalitions and delegate work without centralized coordination, as long as conservation laws are satisfied |
| *Dynamic agent instantiation* | Rather than selecting from a fixed pool, agents instantiate specialists on-demand; the contract becomes the specification for agent creation, not just selection |
| *Meta-governance* | Contracts can govern the creation of other contracts, enabling principled scaling of multi-agent systems |

These patterns demonstrate how contracts can govern increasingly complex multi-agent systems. However, practical enforcement faces fundamental constraints that shape what contracts can and cannot guarantee.

---

## 7. Fundamental Limitations and Practical Enforcement

### 7.1 Single-Call Enforcement Constraints

A critical limitation exists: token consumption is only known *after* an LLM call completes, not during execution. This means contracts cannot interrupt a single call mid-generation to enforce a hard token budget — enforcement must occur at call boundaries (before/after individual LLM invocations), not within them.

> ⚠️ **Nota:** El contenido a partir de este punto (secciones 7.2 en adelante: Capacidades de cumplimiento, Requisitos de infraestructura futura, Evaluación empírica completa, Conclusión y Referencias) no pudo recuperarse completo desde la versión HTML de arXiv debido a límites de extracción de la herramienta de navegación. El resumen del abstract ya cubre los resultados clave: **90% de reducción de tokens, 525× menor varianza en flujos iterativos, cero violaciones de conservación en delegación multi-agente, y tradeoffs medibles de calidad-recursos**.
>
> Para el texto completo (incluyendo Secciones 7.2–9 y todas las referencias), consulta directamente:
> - Versión HTML: https://arxiv.org/html/2601.08815
> - Versión PDF: https://arxiv.org/pdf/2601.08815v3

---

*Documento generado a partir de arXiv:2601.08815v3 — "Agent Contracts: A Formal Framework for Resource-Bounded Autonomous AI Systems" (Qing Ye, Jing Tan). Licencia CC BY 4.0.*
