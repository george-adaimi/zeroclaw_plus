# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with this repository.

> **Shared instructions live in [`AGENTS.md`](./AGENTS.md).** Read it first — it contains the single-source-of-truth rule, risk tiers, workflow rules, anti-patterns, skills, and localization requirements.

## Project Overview

ZeroClaw is a Rust-first autonomous agent runtime (Rust 2024, MSRV 1.87). A single binary that connects to LLM providers (~20+), reaches users through 30+ messaging channels, and acts via tools (shell, browser, HTTP, hardware, MCP). Everything runs locally with user-held keys.

## Quick Commands

```bash
# Full CI gate (recommended before PRs)
./dev/ci.sh all

# Format + lint + test (local, no Docker)
cargo fmt --all -- --check
cargo clippy --all-targets -- -D warnings
cargo test --locked

# Justfile shortcuts
just ci        # fmt-check + lint + test
just build     # release build
just dev -- <args>  # cargo run

# Docs
just docs-build   # build full docs site
just docs         # serve docs locally
just docs-refs    # regenerate CLI/config/rustdoc references

# Docker CI (isolated, reproducible)
./dev/ci.sh lint       # rustfmt + clippy in container
./dev/ci.sh test       # cargo test in container
./dev/ci.sh security   # cargo audit + cargo deny
./dev/ci.sh all        # full pipeline in container
```

## Testing

```bash
# All tests
cargo test --locked

# Unit tests only (faster)
cargo test --lib

# Specific test suites
cargo test --test component --locked
cargo test --test integration --locked
cargo test --test system --locked

# Run a single test
cargo test --lib some_crate::some_module::test_name
```

## Workspace Structure

```
crates/
  zeroclaw-api/        # Public traits: Provider, Channel, Tool, Memory, Observer, RuntimeAdapter, Peripheral
  zeroclaw-config/     # Schema, config loading/merging
  zeroclaw-runtime/    # Agent loop, security, cron, SOP, skills, observability
  zeroclaw-channels/   # 30+ messaging platform adapters
  zeroclaw-tools/      # Tool execution surface (shell, file, memory, browser)
  zeroclaw-providers/  # Model providers + resilient wrapper
  zeroclaw-memory/     # Memory backends (markdown, SQLite, embeddings, vector merge)
  zeroclaw-infra/      # Shared utilities (debounce, session, stall watchdog)
  zeroclaw-gateway/    # HTTP/WebSocket gateway server (separate binary)
  zeroclaw-tui/        # TUI onboarding wizard
  zeroclaw-plugins/    # WASM plugin system
  zeroclaw-hardware/   # USB discovery, peripherals, serial, GPIO
  zeroclog/log/        # Unified log surface + JSONL persistence
  zeroclaw-macros/     # Configurable derive macro
  zeroclaw-tool-call-parser/  # Tool call parsing
  aardvark-sys/        # Native bindings
  robot-kit/           # Hardware kit support
src/
  main.rs              # CLI entrypoint and command routing
  lib.rs               # Module re-exports and CLI command enum
```

## Feature Flags

The workspace uses feature flags extensively. Key flags:

- **`agent-runtime`** (default) — full agent loop, channels, tools. Without it, you get the kernel (config + providers + memory + CLI chat).
- **`gateway`** (default) — HTTP/WebSocket gateway server.
- **`tui-onboarding`** (default) — TUI onboarding wizard.
- **`acp-bridge`** (default) — Agent Client Protocol bridge.
- **`channel-*`** — individual channel adapters (discord, telegram, matrix, slack, etc.).
- **`schema-export`** (default) — export config schema to docs.

Build with custom features:
```bash
cargo build --no-default-features --features agent-runtime,channel-discord
```

## Architecture Principles

1. **Single source of truth** — No piece of state lives in two places. Resolve from canonical source at runtime. See AGENTS.md for the forcing mechanism.
2. **Trait-driven extensibility** — Extend by implementing traits (`Provider`, `Channel`, `Tool`, `Memory`, etc.) and registering in factory modules.
3. **Modular microkernel** — Core runtime is minimal; subsystems are optional crates wired via features.
4. **Localization** — All user-facing output uses `fl!()` / Fluent strings. Log/panic messages stay in English.

## Stability Tiers

| Tier | Meaning |
|------|---------|
| Stable | Covered by breaking-change policy |
| Beta | Breaking changes permitted in MINOR with changelog notes |
| Experimental | No stability guarantee |

See AGENTS.md for the full crate-to-tier mapping. Tiers are promoted, never demoted.

## Skills (invoked via slash commands)

| Skill | Trigger |
|-------|---------|
| `github-pr-review-session` | `review 1234`, `re-review 1234` |
| `changelog-generation` | `generate changelog`, `release notes` |
| `github-issue-triage` | `triage issues`, `sweep issues` |
| `github-issue` | `file issue`, `report bug`, `feature request` |
| `github-pr` | `open PR`, `update PR` |
| `skill-creator` | `create skill` |
| `squash-merge` | `squash-merge #123` |
| `zeroclaw` | `check agent status`, `manage memory` |

## Risk Tiers

- **Low**: docs/chore/tests-only
- **Medium**: most `crates/*/src/**` behavior changes without boundary/security impact
- **High**: `zeroclaw-runtime/src/security/`, `zeroclaw-gateway/src/**`, `zeroclaw-tools/src/**`, `.github/workflows/**`, access-control boundaries

When uncertain, classify higher.
