# Architecture

## Overview

Distributed agent orchestration framework with plugin architecture, skill system, and multi-model routing. Rust daemon handles orchestration and state; Go CLI/API gateway handles client interaction.

## Components

### phenotype-daemon (`phenotype-daemon/`)
Background orchestration service with event-sourced state (SQLite). Modules:
- `agent/` — Agent lifecycle, state machine
- `skill/` — Skill registry, routing, MCP-compatible tool adapters
- `model/` — Multi-model inference routing (Claude, GPT-4, Gemini, Ollama)
- `workflow/` — Multi-agent job scheduling and dependency resolution

### pheno-cli (`pheno-cli/`)
Command-line interface with command handlers (`commands/`), config loading (`config/`). Entry: `main.rs`.

### phenotype-agent-core (`phenotype-agent-core/`)
Core types and traits shared across the workspace: Agent trait, Skill contracts, Model interfaces, Error types.

### agentapi (`agentapi/`)
gRPC API definitions (Protocol Buffers) for daemon-CLI communication.

### CLIProxyAPI
CLI proxy/gateway for interactive client access.

## Data Flow

`pheno-cli` command -> gRPC call to `phenotype-daemon` (Unix socket) -> skill registry + model router -> tool provider plugins (MCP, Composio, E2B) -> event-sourced state in SQLite.

## Key Files

- `phenotype-daemon/` — Rust daemon (entry: `main.rs`)
- `phenotype-agent-core/` — Core traits and types
- `pheno-cli/` — Go CLI binary
- `agentapi/proto/` — Protocol Buffer schemas
- `Cargo.toml` — workspace manifest
