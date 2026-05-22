# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

> **Shared instructions live in [`AGENTS.md`](./AGENTS.md).**
> This file supersedes the original CLAUDE.md and consolidates project operations, architecture, and developer workflows for Claude Code.

## Project Overview

**ZeroClaw** is a Rust-first autonomous agent runtime — a single binary that connects LLM providers (Anthropic, OpenAI, Ollama, ~20+ others) to 30+ messaging channels (Discord, Telegram, Matrix, email, voice, webhooks) with tools (shell, browser, HTTP, hardware, MCP). Everything runs locally on the user's machine with their keys in their workspace.

- **Language**: Rust 2024 edition (MSRV: 1.87)
- **License**: MIT OR Apache-2.0
- **Workspace version**: 0.8.0-beta-1
- **Package name**: `zeroclawlabs` (publish = false during microkernel workspace transition)
- **Official repo**: https://github.com/zeroclaw-labs/zeroclaw

## Critical Rules (from AGENTS.md)

### ABSOLUTE RULE — SINGLE SOURCE OF TRUTH
**No piece of state lives in two places.** If a fact already exists in the codebase, you reference it — never copy it into a new field, struct, config block, or cache. Before adding any new struct field, state the source of truth. If it's a duplicate, resolve from canonical source at use-time via closures, `&Config` parameters, or getter traits.

### Pre-edit Ritual
Before writing any new field:
1. State the source of truth
2. If "created here" → OK to write the field
3. If "duplicate of `<path>`" → DO NOT write it; resolve at use-time

### Forbidden Patterns
- Cached `Vec<String>` of authorized users inside channel handles while config TOML is the source
- `ConfigSnapshot` structs that clone live `Config` fields already reachable through `Arc<RwLock<Config>>`
- Re-emitting API keys into runtime struct fields when the runtime already has the typed alias config

### Allowed Patterns
- Resolver closures (`Arc<dyn Fn() -> T + Send + Sync>`) over `Arc<RwLock<Config>>`
- `&Config` / `&AgentConfig` parameters through call sites
- On-demand materialized views (cached per-call, not stored)
- Derive macros emitting multiple surfaces from one input table

## Commands

### Quick Development
```bash
cargo build                  # Debug build
cargo test --locked           # All tests
cargo fmt --all -- --check    # Format check
cargo clippy --all-targets -- -D warnings  # Lint
```

### Full Pre-PR Validation
```bash
./dev/ci.sh all              # Full CI parity in Docker (lint, test, build, security, docker-smoke)
./dev/ci.sh lint             # rustfmt + clippy in container
./dev/ci.sh test             # cargo test in container
./dev/ci.sh build            # Release build smoke check
./dev/ci.sh security         # cargo audit + cargo deny
./dev/ci.sh lint-strict      # Strict clippy warnings gate
```

### Justfile Recipes
```bash
just ci                      # fmt-check + lint + test (local CI)
just fmt-all                 # cargo fmt + taplo format
just lint                    # cargo clippy
just test                    # cargo test --locked
just test-lib                # cargo test --lib (unit tests only, faster)
just build                   # cargo build --release --locked
just audit                   # cargo audit
just deny                    # cargo deny check
just doc                     # cargo doc --no-deps --open
```

### Running the Binary
```bash
cargo run -- <args>          # Run zeroclaw CLI
just dev <args>              # Same as above (just wrapper)
```

### Test Levels
```bash
cargo test --locked                              # All tests (unit + component + integration + system + architecture)
cargo test --lib                                 # Library unit tests only
cargo test --test component --locked             # Component tests
cargo test --test integration --locked           # Integration tests
cargo test --test system --locked                # System tests
cargo test --test live -- --ignored              # Live tests (requires credentials)
cargo test --test architecture                   # Architecture invariant gates
```

### Benchmarks
```bash
cargo bench --bench agent_benchmarks
```

### Documentation
```bash
just docs              # mdbook serve (local docs dev server)
just docs-build        # mdbook build (full site to docs/book/book/)
just docs-refs         # Regenerate reference docs (CLI, config, rustdoc)
just docs-sync         # Sync .po files with English source
just docs-translate-force  # Force-retranslate for quality pass
```

### Security & Dependencies
```bash
cargo deny check       # License + source compliance
cargo audit            # Vulnerability scan
./scripts/ci/deny_check.sh  # Additional deny checks
```

## Pre-Push Hook

Install with: `git config core.hooksPath .githooks`

The pre-push hook runs `rust_quality_gate.sh` (fmt + clippy) + `cargo test` automatically. Opt-in env vars:
- `ZEROCLAW_STRICT_LINT=1` — strict lint pass on full repo
- `ZEROCLAW_DOCS_LINT=1` — markdown gate on changed lines
- `ZEROCLAW_DOCS_LINKS=1` — link check on added links only
- `ZEROCLAW_STRICT_DELTA_LINT=1` — strict delta lint gate

Skip with `git push --no-verify`.

## Repository Structure

```
src/                          # Main package (CLI entrypoint, lib, commands)
  main.rs                     # CLI entrypoint and command routing
  lib.rs                      # Module re-exports and CLI command enums
  commands/                   # CLI subcommand implementations
  cli_input.rs                # CLI input parsing

crates/                       # Workspace sub-crates
  zeroclaw-api/               # Public trait definitions (Provider, Channel, Tool, Memory, etc.)
  zeroclaw-runtime/           # Agent loop, security, SOP, cron, skills, onboarding, TUI
  zeroclaw-config/            # TOML schema, config loading/merging, secrets
  zeroclaw-providers/         # LLM clients (Anthropic, OpenAI, Ollama, etc.) + router
  zeroclaw-channels/          # 30+ messaging integrations + orchestrator
  zeroclaw-tools/             # Tool implementations (shell, file, memory, browser)
  zeroclaw-memory/            # Memory backends (markdown, sqlite, embeddings, vector merge)
  zeroclaw-gateway/           # HTTP/WebSocket gateway + web dashboard (separate binary)
  zeroclaw-infra/             # Shared infrastructure (debounce, session, watchdog)
  zeroclaw-log/               # Unified logging (record! macro, JSONL, broadcast)
  zeroclaw-macros/            # Configurable derive macro
  zeroclaw-tui/               # Terminal UI onboarding wizard
  zeroclaw-plugins/           # WASM plugin system
  zeroclaw-hardware/          # Hardware abstraction (USB, GPIO, I2C, SPI)
  zeroclaw-tool-call-parser/  # Tool call syntax parsing
  aardvark-sys/               # Aardvark I2C/SPI/GPIO USB adapter bindings
  robot-kit/                  # Robot kit hardware support
```

## Architecture

ZeroClaw is a **microkernel-style layered workspace**. Key flows:

1. **Request lifecycle**: User message → Channel → Runtime → Provider (LLM) → Tool (if needed) → Security check → Memory → Reply → Channel → User
2. **Extension points**: All defined as traits in `zeroclaw-api`:
   - `Provider` — new LLM endpoints
   - `Channel` — new messaging platforms
   - `Tool` — new agent capabilities
   - `Memory` — new memory backends
   - `Observer` — new observability targets
   - `RuntimeAdapter` — runtime customization
   - `Peripheral` — hardware boards

3. **Feature-flagged modules**: Most subsystems are opt-in via Cargo features. The `agent-runtime` meta-feature pulls in channels, tools, and all default channels. See Cargo.toml `[features]` for the full taxonomy.

4. **Binary targets**:
   - `zeroclaw` — main CLI + agent runtime binary
   - `zeroclaw-acp-bridge` — ACP (Agent Client Protocol) bridge (requires `acp-bridge` feature)

## Stability Tiers

| Tier | Meaning |
|------|---------|
| **Stable** | Covered by breaking-change policy |
| **Beta** | Breaking changes permitted in MINOR with changelog notes |
| **Experimental** | No stability guarantee |

Key crate tiers (from AGENTS.md § Stability Tiers):
- `zeroclaw-api` — Experimental (formal milestone: v1.0.0)
- `zeroclaw-config`, `zeroclaw-log`, `zeroclaw-providers`, `zeroclaw-memory`, `zeroclaw-infra`, `zeroclaw-tool-call-parser`, `zeroclaw-macros` — Beta
- `zeroclaw-channels`, `zeroclaw-tools`, `zeroclaw-runtime`, `zeroclaw-gateway`, `zeroclaw-tui`, `zeroclaw-plugins`, `zeroclaw-hardware` — Experimental

Tiers are promoted, never demoted.

## Risk Tiers

- **Low**: docs/chore/tests-only changes
- **Medium**: most `crates/*/src/**` behavior changes without boundary/security impact
- **High**: `zeroclaw-runtime/src/security/`, `zeroclaw-gateway/src/`, `zeroclaw-tools/src/`, `.github/workflows/`, access-control boundaries

When uncertain, classify as higher risk.

## Code Style

- **rustfmt**: 100-char max width, 4-space tabs, `use_field_init_shorthand`, `use_try_shorthand`, sorted imports/modules
- **taplo**: TOML files use 2-space indent, sorted arrays in `[dependencies]`/`[features]`
- **Clippy**: Disallows `tracing::*`, `log::*`, `dbg!`, `anyhow::anyhow!` in production code — all log emissions go through `::zeroclaw_log::record!`
- **No `unwrap()`/`expect()`** in production paths; propagate errors or document the invariant
- **No dead code** with underscore prefixes — delete, wire in, or track a follow-up issue

## Localization

- All user-facing output uses `fl!()` / Fluent strings — never bare string literals
- Log messages, `tracing::` spans, and panics stay in English
- Wiki and internal dev docs are English only
- Supported locales: en, fr, ja, es, zh-CN (see `locales.toml`)

## Skills (AI Coding Assistant Tools)

Located in `.claude/skills/`:
- `github-pr-review-session` — PR review co-pilot (taxonomy:/red, yellow, green)
- `changelog-generation` — generates `CHANGELOG-next.md` between stable tags
- `github-issue-triage` — issue triage, labeling, stale policy
- `github-issue` — interactive GitHub issue filing
- `github-pr` — open/update PRs with validation
- `skill-creator` — create/test/evaluate new skills
- `squash-merge` — squash-merge with preserved history
- `zeroclaw` — operational guide for ZeroClaw agent instances

## Contributing Workflow

1. Work from a non-`master` branch (`feat/*` or `fix/*`)
2. One concern per PR (no mixed feature+refactor+infra)
3. Complete every section in `.github/pull_request_template.md`
4. Include validation evidence (literal command output, not "CI will check")
5. Squash-merge with conventional commits is the merge style
6. Never commit secrets, PII, or real identity information

## Security

- Report security vulnerabilities to `security@zeroclaw.dev` (not public GitHub issues)
- Pre-commit hook runs `gitleaks protect --staged --redact`
- `cargo deny` enforces: no unmaintained deps, no yanked crates, no unknown registries/git sources
- Approved licenses: MIT, Apache-2.0, BSD-2/3, ISC, Unicode, MPL-2.0, 0BSD, BSL-1.0, CC0-1.0, CDLA-Permissive-2.0, Zlib, OpenSSL
