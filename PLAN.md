# PhenoAgent - Project Plan

**Document ID**: PLAN-PHENOAGENT-001
**Version**: 2.0.0
**Created**: 2026-04-05
**Updated**: 2026-05-06
**Status**: In Progress
**Project Owner**: Phenotype Agent Team
**Review Cycle**: Monthly

---

## 1. Project Overview & Objectives

### 1.1 Vision Statement

PhenoAgent is Phenotype's autonomous agent framework - providing the infrastructure for building, deploying, and managing AI agents that can perform complex tasks, make decisions, and interact with Phenotype services, tools, and data to accomplish complex goals autonomously.

### 1.2 Mission Statement

To enable the development of reliable, observable, and secure AI agents that extend human capabilities and automate complex workflows within the Phenotype ecosystem. PhenoAgent provides the opinionated, production-ready substrate that bridges the gap between LLM capability and reliable autonomous execution.

### 1.3 Core Objectives

| Objective ID | Description | Success Criteria | Priority |
|---|---|---|---|
| OBJ-001 | Agent runtime | Tokio async runtime with lifecycle hooks and checkpointing | P0 |
| OBJ-002 | Tool system | Skill registry with MCP tool exposure and schema validation | P0 |
| OBJ-003 | Memory management | Working memory + long-term memory with semantic retrieval | P0 |
| OBJ-004 | Planning | LLM-driven goal decomposition with DAG plan representation | P1 |
| OBJ-005 | LLM integration | Multi-model routing with skill-based and fallback policies | P0 |
| OBJ-006 | Observability | Structured logs, Prometheus metrics, OTel spans | P1 |
| OBJ-007 | Safety | Policy guards, permission scopes, approval workflow | P0 |
| OBJ-008 | Multi-agent | Agent communication, role assignment, task delegation | P2 |
| OBJ-009 | Deployment | Containerized daemon with Docker and systemd unit | P1 |
| OBJ-010 | Testing | Unit, integration, and E2E test suite with coverage gate | P1 |

---

## 2. Architecture

### 2.1 Component Map

```
PhenoAgent/
├── phenotype-daemon/          # Tokio async runtime, agent orchestration
│   ├── src/
│   │   ├── agent/           # Agent lifecycle, state machine, hooks
│   │   ├── skill/           # Skill registry, routing, invocation
│   │   ├── model/            # Model router, provider traits
│   │   ├── memory/           # Working memory, long-term store
│   │   ├── planning/        # Planner trait, LLM-based implementation
│   │   ├── policy/          # PolicyGuard, scope enforcement
│   │   ├── observability/    # Logging, metrics, tracing
│   │   └── workflow/        # Multi-agent coordination primitives
│   └── Cargo.toml
├── phenotype-agent-core/      # Core traits, types, error definitions
│   ├── src/
│   │   ├── agent.rs         # Agent trait, AgentConfig, RunContext
│   │   ├── skill.rs         # Skill trait, SkillRegistry, SkillContext
│   │   ├── model.rs         # ModelProvider trait, ModelError
│   │   ├── memory.rs        # MemoryStore trait, MemoryEntry
│   │   ├── plan.rs          # Plan, Step, PlanGraph types
│   │   ├── policy.rs        # PolicyGuard, GuardResult, PermissionScope
│   │   └── error.rs         # Unified error taxonomy
│   └── Cargo.toml
├── pheno-cli/                 # CLI interface
│   ├── src/
│   │   ├── commands/         # agent, skill, model, approval subcommands
│   │   ├── config/           # Config loading, validation
│   │   └── main.rs
│   └── Cargo.toml
├── phenotype-skills/           # Built-in skill implementations
│   ├── src/
│   │   ├── builtin/          # Search, code analysis, file I/O skills
│   │   └── mcp/              # MCP adapter for external tools
│   └── Cargo.toml
├── agentapi/                  # gRPC API definitions (protobuf)
│   └── proto/agent.proto
└── CLIProxyAPI/               # HTTP gateway (Go; proxy to gRPC)
```

### 2.2 Agent Execution Flow

```
pheno-cli agent run --name my-agent --goal "analyze code"
  │
  └─► phenotype-daemon (Unix socket /var/run/phenotype/agent.sock)
       │
       ├─ 1. PolicyGuard.check(scope) ──── deny/allow
       │
       ├─ 2. Planner.plan(goal, context) ──── Plan { steps: [...] }
       │
       ├─ 3. For each step in dependency order:
       │     ├─ 3a. ModelRouter.complete(prompt) ──── ToolCall
       │     ├─ 3b. PolicyGuard.check(tool) ──── deny/allow
       │     ├─ 3c. SkillRegistry.invoke(tool, input) ──── Value
       │     └─ 3d. Memory.append(step_result)
       │
       └─ 4. Respond with completion + decision trace
```

---

## 3. Milestones & Deliverables

### Phase 1: Foundation (Complete — v0.1–0.3)

**M1.1 — Core Agent Runtime**
- Tokio async runtime scaffold in `phenotype-daemon`
- `Agent` trait with `run(goal) -> Result<Response, AgentError>`
- Lifecycle hooks: init, before_step, after_step, on_error, shutdown
- Basic state machine: Idle → Running → Completed | Failed
- SQLite checkpointing after each step

**M1.2 — Skill Registry**
- `Skill` trait with name, version, description, input_schema, invoke()
- `SkillRegistry` with in-memory index and O(1) lookup
- JSON Schema validation before invocation
- Version conflict detection at registration time

**M1.3 — pheno-cli**
- `agent create`, `agent list`, `agent run`, `agent reset` commands
- `skill list`, `skill invoke` commands
- Unix socket IPC to daemon

---

### Phase 2: Intelligence (Current — v0.4–0.6)

**M2.1 — Planning Engine**
- `Planner` trait with LLM-based implementation
- `PlanGraph` with step dependencies and DOT export
- Re-planning on step failure with failure context injection
- Circular dependency detection at planning time
- **Deliverable**: `planner.rs` in `phenotype-agent-core`, integration test in `phenotype-daemon`

**M2.2 — Multi-Model Routing**
- `ModelProvider` trait with `complete()` and `embed()` methods
- Provider implementations: Anthropic (Claude), OpenAI (GPT-4), Google (Gemini), Ollama
- `ModelRouter` with `SkillBased`, `FallbackChain`, `LatencyBudget` policies
- Routing decision logging to trace event stream
- **Deliverable**: `model/` module, router integration test with mocked HTTP

**M2.3 — Policy Guards & Permissions**
- `PolicyGuard` trait with pre/post-condition evaluation
- `PermissionScope` enforcement at runtime
- `GuardResult::Allow` / `GuardResult::Deny` with structured reason
- Guard composition (AND/OR/NOT combinators)
- **Deliverable**: `policy.rs`, guard integration test

---

### Phase 3: Memory & Knowledge (v0.7–0.9)

**M3.1 — Working Memory**
- `WorkingMemory` key-value store with dot-path access
- Checkpoint integration (serialize/deserialize with agent state)
- Memory access from within skills via `SkillContext`

**M3.2 — Long-Term Memory**
- SQLite-backed `LongTermMemory` with embedding column
- `MemoryStore::retrieve(query, top_k)` by cosine similarity
- Embedding generation via configured `ModelProvider`
- Fallback to keyword search if no embedding model is configured

**M3.3 — Episodic Memory**
- `MemoryStore::store_episode(run_id, events)` for full run replay
- `MemoryStore::get_episode(run_id)` for replay/debugging
- Episode pruning policy (keep last N runs, configurable)

---

### Phase 4: Observability (v1.0)

**M4.1 — Structured Logging**
- JSON log format with agent_id, run_id, step, span_id
- trace_id propagation from CLI request through all async tasks
- Configurable output: stdout, file, syslog
- Log level tuning per module via RUST_LOG

**M4.2 — Prometheus Metrics**
- `phenoagent_steps_total{agent, status}` counter
- `phenoagent_step_duration_seconds{agent, tool}` histogram
- `phenoagent_active_agents{gauge}` gauge
- `/metrics` endpoint on daemon HTTP port (default 8080)

**M4.3 — OpenTelemetry Tracing**
- `tracing` spans wrapping agent steps, tool calls, LLM invocations
- OTLP / console exporter configuration via `OTEL_EXPORTER` env var
- W3C TraceContext propagation over gRPC

---

### Phase 5: Safety & Security (v1.1)

**M5.1 — Approval Workflow**
- `high_risk: bool` flag on skill definitions
- `pending_approvals` table with UUID, tool, input, timestamp
- `pheno-cli approval approve --id UUID` and `--deny --id UUID`
- Configurable timeout (default 5 min); expired = denied

**M5.2 — Input/Output Sanitization**
- JSON Schema validation of all skill inputs (already in M1.2, ensure enforced)
- Output size limit enforcement (configurable, default 1 MB)
- Control character stripping on all string outputs from skills
- `SkillError::SchemaMismatch` and `SkillError::OutputTooLarge` variants

**M5.3 — Secrets Management**
- Model provider credentials loaded from environment variables
- No plaintext credentials in `config.toml` or `agent.toml`
- Secrets redacted in all log output

---

### Phase 6: Multi-Agent (v1.2)

**M6.1 — Agent Communication**
- `AgentBus` trait for inter-agent message passing
- Message types: `TaskRequest`, `TaskResponse`, `StatusUpdate`, `Cancellation`
- Redis-backed implementation for distributed deployments
- In-process implementation for single-machine testing

**M6.2 — Role Assignment & Delegation**
- Role definitions: `orchestrator`, `worker`, `reviewer`
- `TaskDelegator` that assigns sub-tasks to worker agents
- Dependency resolution across agents (blocking on remote completion)

**M6.3 — Conflict Resolution**
- `ConflictDetector` that identifies competing tool calls
- Configurable resolution strategies: first-write-wins, priority-based, human-in-the-loop
- Conflict events written to the audit trail

---

### Phase 7: Deployment & Release (v1.3)

**M7.1 — Containerization**
- `Dockerfile` for `phenotype-daemon` with multi-stage build
- `docker-compose.yml` for daemon + Redis + SQLite volumes
- Helm chart for Kubernetes deployment
- Systemd unit file for bare-metal deployment

**M7.2 — Release Process**
- Semantic versioning (MAJOR.MINOR.PATCH)
- GitHub Actions release workflow: build, test, tag, publish crate, push Docker image
- Changelog auto-generation from conventional commits
- `CHANGELOG.md` and `Cargo.toml` version bump as part of release

---

## 4. Timeline (Target)

| Phase | Target Version | Estimated Completion | Status |
|---|---|---|---|
| Phase 1: Foundation | v0.3 | 2026-04 | Complete |
| Phase 2: Intelligence | v0.6 | 2026-05 | In Progress |
| Phase 3: Memory | v0.9 | 2026-06 | Planned |
| Phase 4: Observability | v1.0 | 2026-07 | Planned |
| Phase 5: Safety | v1.1 | 2026-08 | Planned |
| Phase 6: Multi-Agent | v1.2 | 2026-09 | Planned |
| Phase 7: Deployment | v1.3 | 2026-10 | Planned |

All timelines are agent-driven wall-clock estimates. External dependencies (API integrations, CI/CD) are assumed to be non-blocking.

---

## 5. Dependencies

### 5.1 External Crates (Rust)

| Crate | Version | Purpose | Risk |
|---|---|---|---|
| tokio | 1.x | Async runtime | Low |
| tokio-rusqlite | 0.3.x | SQLite async bindings | Low |
| serde + serde_json | 1.x | Serialization | Low |
| jsonschema | 0.18.x | Skill input validation | Low |
| metrics | 0.22.x | Prometheus instrumentation | Low |
| tracing + tracing-otlp | 0.1.x | Structured logging + OTel | Medium |
| reqwest | 0.12.x | HTTP client for model providers | Low |
| anyhow + thiserror | 1.x | Error handling | Low |
| toml | 0.8.x | Config file parsing | Low |
| clap | 4.x | CLI argument parsing | Low |

### 5.2 External Services

| Service | Purpose | Fallback |
|---|---|---|
| Anthropic API | Claude model | Mock provider |
| OpenAI API | GPT-4 model | Mock provider |
| Redis (optional) | Distributed agent bus | In-process bus |
| OTEL Collector (optional) | Trace export | Console exporter |

### 5.3 Internal Dependencies

| Repo | Dependency | Status |
|---|---|---|
| PhenoMCP | MCP skill adapter | In development |
| cheap-llm-mcp | Batch embedding via cheap LLM | Available |
| agentapi-plusplus | gRPC API gateway | Available |
| thegent-dispatch | Agent scheduling | In development |

---

## 6. Risks & Mitigations

| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| LLM planning produces invalid step schemas | Medium | High | Schema-guided prompting + post-generation validation |
| Circular skill dependencies in complex workflows | Low | Medium | Registration-time DAG check |
| Model API rate limits disrupting agent runs | High | Medium | Fallback chain policy + exponential backoff |
| Checkpoint corruption on unclean shutdown | Low | High | Atomic writes (write to temp, rename) + startup integrity check |
| Credential exposure in logs | Low | Critical | Secrets redaction pass + pre-commit hook scanning |
| Redis unavailability blocking multi-agent runs | Medium | Medium | In-process bus as fallback; Redis failure degrades to single-node |

---

## 7. Open Decisions

| Decision | Options | Owner | Due |
|---|---|---|---|
| Embedding model for long-term memory | Instructor (local) vs OpenAI ada-002 vs Voyage AI | TBD | Phase 3 start |
| Max step budget default | 50 (current) vs 100 | TBD | Phase 2 close |
| Agent run TTL (auto-cleanup) | 24h vs 7d vs never | TBD | Phase 5 |
| Multi-agent message serialization format | JSON vs MessagePack vs Protobuf | TBD | Phase 6 start |

---

## 8. Metrics & Success

### 8.1 Engineering Metrics

- **Test coverage**: 80% line coverage gate on `phenotype-agent-core`; 60% on `phenotype-daemon`.
- **Build time**: `cargo build --release` < 5 minutes on standard Linux runner.
- **Daemon startup**: Cold start (no cached agents) < 2 seconds.

### 8.2 Product Metrics (v1.0)

- 95% of single-step agent runs complete without error.
- Planning success rate (LLM produces valid step plan) > 90% for well-scoped goals.
- Policy guard evaluation latency < 1 ms per guard.

---

## 9. Work Breakdown Structure

### WBS-1: Planning Engine (M2.1)
1.1 Define `Planner` trait and `PlanGraph` types in `phenotype-agent-core`
1.2 Implement LLM-based `PlannerImpl` with structured output
1.3 Add dependency DAG validation
1.4 Add re-planning on failure with failure context
1.5 Write unit tests and integration tests
1.6 Add `pheno-cli plan visualize` command

### WBS-2: Multi-Model Routing (M2.2)
2.1 Define `ModelProvider` trait with `complete` and `embed` methods
2.2 Implement `AnthropicProvider`, `OpenAIProvider`, `OllamaProvider`
2.3 Implement `ModelRouter` with policy enum
2.4 Add routing decision logging
2.5 Write HTTP mock tests for each provider
2.6 Add `pheno-cli model list` and `pheno-cli model test` commands

### WBS-3: Policy Guards (M2.3)
3.1 Define `PolicyGuard` trait and `GuardResult` enum
3.2 Implement `PermissionScope` enforcement
3.3 Implement guard combinators (AND/OR/NOT)
3.4 Add guard evaluation to agent runtime
3.5 Write policy test cases including edge cases

---

## 10. Document Control

- **Version**: 2.0.0
- **Status**: In Progress
- **Last Updated**: 2026-05-06
- **Next Review**: 2026-06-05
- **Changelog**:
  - v2.0.0 (2026-05-06): Expanded from thin 75-line draft to full plan with WBS, milestones, risks, open decisions, and metrics. Added Phases 3-7.
  - v1.0.0 (2026-04-05): Initial draft structure.
