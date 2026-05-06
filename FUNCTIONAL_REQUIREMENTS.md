# Functional Requirements — PhenoAgent

**Document ID**: FR-PHENOAGENT-001
**Version**: 2.0.0
**Status**: Active
**Traces to**: PRD.md epics E1–E8
**ID format**: FR-PHENOAGENT-{NNN}

---

## 1. Overview

PhenoAgent is Phenotype's autonomous agent framework. This document captures
detailed functional requirements for all agent subsystems including runtime,
skill registration, planning, memory, multi-model routing, observability, and
security. Each requirement is traceable to a PRD epic and includes acceptance
criteria that define done-ness.

---

## 2. User Stories

### US-AGENT-001: Developer Creates a Custom Agent
**As a** developer building on the Phenotype platform,
**I want** to instantiate a typed agent with a name, model, and skill set,
**so that** I can execute autonomous workflows without boilerplate infrastructure code.

**Acceptance criteria**:
- `pheno-cli agent create --name my-agent --model claude-opus-4-1 --skills search,code` creates a persistent agent.
- Agent config is validated (non-empty name, known model ID, existing skill IDs).
- Agent is persisted to SQLite and appears in `pheno-cli agent list`.

### US-AGENT-002: Agent Executes a Multi-Step Task
**As a** platform operator,
**I want** the agent to decompose a user goal into ordered steps, execute each step with the appropriate tool, and return a coherent response,
**so that** complex tasks are handled end-to-end without manual intervention.

**Acceptance criteria**:
- Given a goal string, the agent produces a plan with at least one step.
- Each step is executed sequentially with observable state transitions (pending, running, done, failed).
- The agent retries transient failures up to 3 times before surfacing an error.
- A final response is returned within 60 seconds for goals that have a resolvable answer.

### US-AGENT-003: Developer Registers a Custom Skill
**As a** developer extending PhenoAgent,
**I want** to register a skill that implements the `Skill` trait with metadata, input schema, and versioning,
**so that** the skill is discoverable and callable by agents at runtime.

**Acceptance criteria**:
- A skill struct implementing `Skill` registers via `SkillRegistry::register(skill)`.
- Skill metadata (name, version, description) is stored and queryable.
- Skill input is validated against the declared JSON Schema before invocation.
- Version conflicts (same name, different version) are rejected with a distinct error.

### US-AGENT-004: Operator Audits Agent Decision History
**As a** security officer,
**I want** to inspect the full decision tree and tool call log for any agent run,
**so that** I can understand why an agent took a specific action and prove compliance.

**Acceptance criteria**:
- Every agent step is written to the event log with timestamp, step number, selected tool, tool input, and tool output.
- Decision traces include LLM reasoning text when available.
- Logs are queryable by agent ID and time range via `pheno-cli agent traces`.
- Traces are immutable once written (append-only event log).

### US-AGENT-005: Developer Constrains Agent Behavior with Policy Guards
**As a** platform admin,
**I want** to attach policy guard conditions to an agent that prevent or warn on specific actions,
**so that** I can enforce organizational safety rules without modifying agent logic.

**Acceptance criteria**:
- A `PolicyGuard` struct defines pre-conditions and post-conditions for tool calls.
- Guards are evaluated before every tool invocation; `GuardResult::Allow` passes execution, `GuardResult::Deny` aborts with a structured error.
- Policy evaluation itself is logged with the matched rule ID.
- Guards can be composed (AND/OR/NOT combinators) and attached at agent creation time.

### US-AGENT-006: Platform Routes Across Multiple LLM Providers
**As a** infrastructure engineer,
**I want** the agent to route requests to Claude, GPT-4, Gemini, or Ollama based on skill, latency, or cost preferences,
**so that** I can optimize for quality, speed, or cost per task type.

**Acceptance criteria**:
- `ModelRouter` selects a provider based on a configurable routing policy (e.g., skill-based, fallback chain).
- Each provider implements the `ModelProvider` trait with a uniform `complete(prompt, params) -> Completion` interface.
- If the primary provider returns a 429 or 503, the router falls back to the next provider in the chain.
- Routing decisions are logged with the selected provider and reason.

### US-AGENT-007: Agent Resumes from Checkpoint After Crash
**As a** operator running long-running agents,
**I want** agent state to be checkpointed after each step and resumable on daemon restart,
**so that** agents recover gracefully from infrastructure failures.

**Acceptance criteria**:
- After each completed step, the agent serializes its state to SQLite (plan, step index, memory, tool history).
- On startup, the daemon detects incomplete runs and resumes from the latest checkpoint.
- The operator can force-restart via `pheno-cli agent reset --name my-agent`.
- Resumed agents continue from the next pending step, not from scratch.

---

## 3. Agent Runtime

**FR-PHENOAGENT-001**: The system SHALL provide an async runtime for executing autonomous agent workflows with task scheduling and state management.
Traces to: PRD E1.1 (FR-RUN-001)

Acceptance criteria:
- Tokio-based async runtime in `phenotype-daemon` handles all agent work.
- Task scheduling uses a work-stealing queue with configurable parallelism.
- State transitions are explicit and typed (`Idle`, `Running`, `Paused`, `Completed`, `Failed`).

**FR-PHENOAGENT-002**: The system SHALL support agent lifecycle hooks: `init`, `before_step`, `after_step`, `on_error`, `shutdown`.
Traces to: PRD E1.2

Acceptance criteria:
- Each hook is an async fn with access to the agent context and step metadata.
- Hooks run sequentially; a hook panic does not cascade to the next hook.
- `init` runs once before the first step; `shutdown` runs once after the final step or on error.

**FR-PHENOAGENT-003**: The system SHALL persist agent state to durable storage and support resumption from checkpoints.
Traces to: PRD E1.3 (FR-RUN-001 acceptance criteria)

Acceptance criteria:
- State is serialized as JSON in a SQLite `agent_checkpoints` table.
- The checkpoint key is `(agent_id, run_id)`.
- Resumption reads the latest checkpoint and reconstructs the agent context in memory.

---

## 4. Skill & Tool Registration

**FR-PHENOAGENT-004**: The system SHALL expose a `Skill` trait for registering reusable capabilities with metadata, versioning, and input schema validation.
Traces to: PRD E2.1 (FR-TOOL-001)

Acceptance criteria:
- `Skill` requires `name: SkillName`, `version: Version`, `description: String`, `input_schema: JsonSchema`, and `invoke(input: Value) -> Result<Value, SkillError>`.
- `SkillRegistry` maintains an in-memory index keyed by `(name, version)` with O(1) lookup.
- Schema validation uses a JSON Schema validator; mismatched input types return `SkillError::SchemaMismatch`.
- `SkillRegistry::list()` returns all registered skills with metadata.

**FR-PHENOAGENT-005**: The system SHALL support MCP resource exposure for skills to enable Claude/LLM tool invocation.
Traces to: PRD E2.2

Acceptance criteria:
- Skills are exposed as MCP resources via the `mcp` feature flag in `phenotype-skills`.
- MCP resource URIs follow the pattern `skill://{name}/{version}/{resource}`.
- Tool schema is derived from the skill's `input_schema` and exposed as an MCP tool definition.

**FR-PHENOAGENT-006**: The system SHALL support skill dependency injection so that one skill can call another.
Traces to: PRD E2.3

Acceptance criteria:
- `SkillContext` provides access to the `SkillRegistry` and agent memory.
- Skills receive `SkillContext` at invocation time; no global state is required.
- Circular skill dependencies are detected at registration time and rejected.

---

## 5. Decision & Reasoning

**FR-PHENOAGENT-007**: The system SHALL log agent decision trees and reasoning traces for auditability and debugging.
Traces to: PRD E3.1 (FR-OBS-001)

Acceptance criteria:
- Every LLM call records: model, prompt tokens, completion tokens, reasoning text, tool call selection, and latency.
- Decision events are written to the `decision_events` table in SQLite.
- Traces are queryable by `run_id` and return an ordered list of decision nodes.
- Reasoning text is retained verbatim (not truncated) for full auditability.

**FR-PHENOAGENT-008**: The system SHALL support policy-driven agent behavior constraints via `PolicyGuard`.
Traces to: PRD E3.2 (FR-SEC-001)

Acceptance criteria:
- `PolicyGuard` evaluates a predicate `(ToolCall, AgentContext) -> GuardResult` before tool execution.
- `GuardResult` is `Allow` or `Deny { reason: String, rule_id: RuleId }`.
- Guards are evaluated in registration order; first `Deny` short-circuits execution.
- Guard evaluation is itself logged to the audit trail.

**FR-PHENOAGENT-009**: The system SHALL support configurable step-level budgets (max steps, max compute time) to prevent runaway loops.
Traces to: PRD E3.3

Acceptance criteria:
- `AgentConfig` exposes `max_steps: u32` (default 50) and `max_compute_seconds: u64` (default 300).
- The runtime enforces these budgets by tracking step count and wall-clock time.
- When a budget is exceeded, the agent enters `Failed` state with reason `BudgetExceeded`.

---

## 6. Planning Engine

**FR-PHENOAGENT-010**: The system SHALL decompose user goals into ordered execution steps using the LLM as the planner.
Traces to: PRD E4.1 (FR-PLAN-001)

Acceptance criteria:
- Given a `goal: String`, `Planner::plan(goal, context) -> Plan` produces a `Plan { steps: Vec<Step> }`.
- Each `Step` has: `description: String`, `tool: SkillName`, `input: Value`, `dependencies: Vec<StepId>`.
- Dependencies form a directed acyclic graph; circular dependencies are rejected at planning time.
- Re-planning on step failure is triggered by `on_error` hook; the LLM receives the failure context and produces a revised plan.

**FR-PHENOAGENT-011**: The system SHALL provide plan visualization via a structured `PlanGraph` type.
Traces to: PRD E4.2 (FR-PLAN-001 acceptance criteria)

Acceptance criteria:
- `PlanGraph` exposes `nodes()` (steps) and `edges()` (dependencies) for serialization.
- `PlanGraph::to_dot()` emits a GraphViz DOT representation for debugging.
- The CLI exposes `pheno-cli plan visualize --run-id X` to render the plan as text.

---

## 7. Memory System

**FR-PHENOAGENT-012**: The system SHALL provide working memory (agent scratchpad) accessible during a single run.
Traces to: PRD E5.1 (FR-MEM-001)

Acceptance criteria:
- `WorkingMemory` is a typed key-value store backed by an in-memory `HashMap<String, Value>`.
- Keys are dot-separated paths (e.g., `code.files[0].name`) for ergonomic access.
- Memory is checkpointed along with agent state and is fully serializable.

**FR-PHENOAGENT-013**: The system SHALL support long-term memory with a semantic retrieval interface.
Traces to: PRD E5.2 (FR-MEM-001 acceptance criteria)

Acceptance criteria:
- `LongTermMemory` is backed by a SQLite table with an embedding column (blob).
- `MemoryStore::retrieve(query: &str, top_k: usize) -> Vec<MemoryEntry>` returns the k most relevant entries by cosine similarity on embeddings.
- Embedding generation is delegated to the configured `ModelProvider` (text-embedding model).
- If no embedding model is configured, retrieval returns an empty vector with a warning log.

---

## 8. Multi-Model Routing

**FR-PHENOAGENT-014**: The system SHALL route LLM requests across providers using a configurable routing policy.
Traces to: PRD E6.1 (FR-RUN-001 acceptance criteria — multi-model inference)

Acceptance criteria:
- `ModelRouter` implements `ModelProvider` and delegates to one of its configured providers.
- Built-in policies: `SkillBased` (route by skill tag), `FallbackChain` (retry on error), `LatencyBudget` (switch after N ms).
- Policies are composable; `RouterConfig` accepts an ordered list of policy evaluators.
- Routing decisions emit a `RoutingDecision` event to the trace log.

**FR-PHENOAGENT-015**: The system SHALL expose a uniform `ModelProvider` trait covering chat completions, embeddings, and tool results.
Traces to: PRD E6.2

Acceptance criteria:
- `ModelProvider::complete(CompletionRequest) -> Result<Completion, ModelError>` is the primary interface.
- `ModelProvider::embed(EmbedRequest) -> Result<EmbedResponse, ModelError>` is optional; providers that don't support embeddings return `ModelError::Unsupported`.
- `ModelError` variants: `ApiError { status, body }`, `RateLimited`, `Timeout`, `Unsupported`.

---

## 9. Observability

**FR-PHENOAGENT-016**: The system SHALL emit structured logs with correlation IDs for every agent operation.
Traces to: PRD E7.1 (FR-OBS-001)

Acceptance criteria:
- Log format is JSON with fields: `timestamp`, `level`, `agent_id`, `run_id`, `step`, `message`, `span_id`.
- A `trace_id` is propagated from the CLI/API request through all async tasks.
- Log output is configurable: stdout (default), file, or syslog.

**FR-PHENOAGENT-017**: The system SHALL expose Prometheus metrics for agent throughput, latency, and error rates.
Traces to: PRD E7.2

Acceptance criteria:
- Metrics: `phenoagent_steps_total{agent, status}`, `phenoagent_step_duration_seconds{agent, tool}`, `phenoagent_active_agents{gauge}`.
- Metrics endpoint is `/metrics` on the daemon HTTP port (default 8080).
- A `metrics.rs` module in `phenotype-daemon` defines and registers all metrics with `metrics` crate.

**FR-PHENOAGENT-018**: The system SHALL support distributed tracing via OpenTelemetry spans.
Traces to: PRD E7.3

Acceptance criteria:
- `tracing` spans wrap each agent step, tool call, and LLM invocation.
- Spans are exported via the configured `OTEL_EXPORTER` (console, OTLP, Jaeger).
- Span context is propagated over gRPC via `W3C TraceContext` headers.

---

## 10. Security

**FR-PHENOAGENT-019**: The system SHALL enforce permission scopes on tool calls based on agent identity.
Traces to: PRD E8.1 (FR-SEC-001)

Acceptance criteria:
- `PermissionScope` is a set of allowed `SkillName` values; agents are instantiated with a scope.
- The runtime checks the agent's scope before invoking any skill.
- Skills outside the agent's scope return `SkillError::PermissionDenied`.
- Scope configuration is stored in the agent config file, not in code.

**FR-PHENOAGENT-020**: The system SHALL support an approval workflow for high-risk tool calls.
Traces to: PRD E8.2 (FR-SEC-001 acceptance criteria)

Acceptance criteria:
- Tool calls tagged `high_risk: true` in the skill definition are paused pending approval.
- Pending approvals are queued in the `pending_approvals` table with a UUID.
- `pheno-cli approval approve --id UUID` or `--deny --id UUID` resolves the queue entry.
- Approval timeouts are configurable; default is 5 minutes; expired approvals are treated as denied.

**FR-PHENOAGENT-021**: The system SHALL sanitize tool inputs and outputs to prevent injection attacks.
Traces to: PRD E8.3

Acceptance criteria:
- All skill inputs are validated against the skill's JSON Schema before invocation.
- Output sanitization strips control characters and enforces maximum output size (configurable, default 1 MB).
- The runtime rejects oversized outputs with `SkillError::OutputTooLarge`.

---

## 11. Non-Functional Requirements

### NFR-PHENOAGENT-001: Performance
- A single-step agent completion (no tool call) SHALL complete within 2 seconds end-to-end on the local loopback.
- A 10-step agent run SHALL complete within 60 seconds under normal conditions (no external API latency).
- The daemon SHALL support at least 50 concurrent agent runs on a machine with 8 CPU cores and 16 GB RAM.

### NFR-PHENOAGENT-002: Reliability
- Checkpoint writes SHALL be atomic (write succeeds or fails, no partial state).
- The daemon SHALL recover from a SIGKILL of the worker process without data loss for completed steps.
- No runtime panics shall escape the agent sandbox; panics are caught and surfaced as `AgentError::Internal`.

### NFR-PHENOAGENT-003: Security
- Agent state SHALL NOT be readable by other agents unless explicitly shared.
- Credentials for model providers SHALL be stored in environment variables or a secrets manager, never in plaintext config.
- All network traffic between CLI and daemon SHALL use Unix domain sockets (no TCP on loopback by default).

### NFR-PHENOAGENT-004: Compatibility
- The `pheno-cli` SHALL remain backward-compatible across minor version bumps (semver patch).
- The `phenotype-daemon` API (gRPC) version is declared in `agentapi/proto/agent.proto`; breaking changes increment the major version.
- Skill schemas follow JSON Schema draft-07.

---

## 12. Trace & Test Guidance

All tests MUST reference a Functional Requirement:

```rust
// Traces to: FR-PHENOAGENT-NNN
#[test]
fn test_agent_lifecycle() { ... }
```

Test categories:
- **Unit**: test individual components in isolation with mocked dependencies.
- **Integration**: test the agent runtime with an in-process skill registry and SQLite.
- **E2E**: test `pheno-cli agent run` against a running daemon with a real model (mocked HTTP).

---

**Document Control**
- Version: 2.0.0
- Status: Active
- Last Updated: 2026-05-06
