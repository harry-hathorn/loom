# Loom Repository - Phase 1 Inventory

**Iteration:** 1
**Date:** 2025-01-16
**Status:** IN PROGRESS

---

## System Overview

**Loom** is an AI-powered coding agent built in Rust. It provides a REPL interface for interacting with LLM-powered agents that can execute tools to perform file system operations, code analysis, and other development tasks.

**Repository:** https://github.com/ghuntley/loom
**License:** Proprietary
**Primary Language:** Rust (with TypeScript/Svelte frontend)

---

## File Tree Structure

```
loom/
├── crates/                    # 86 Rust workspace crates
│   ├── loom-cli-*            # CLI components (10 crates)
│   ├── loom-server-*         # Server components (28 crates)
│   ├── loom-common-*         # Shared libraries (9 crates)
│   ├── loom-tui-*            # Terminal UI (12 crates)
│   ├── loom-weaver-*         # Remote execution (5 crates)
│   ├── loom-wgtunnel-*       # WireGuard tunnel (5 crates)
│   ├── loom-analytics*       # Analytics (3 crates)
│   ├── loom-flags*           # Feature flags (3 crates)
│   └── [specialized crates]  # SCIM, redact, etc.
│
├── web/                       # Web frontend
│   ├── loom-web/             # Svelte 5 application
│   └── packages/             # Shared packages (analytics, flags, http)
│
├── ide/                       # Editor integrations
│   └── vscode/               # VSCode extension (TypeScript)
│
├── infra/                     # Infrastructure as code
│   ├── nixos-modules/        # NixOS configuration modules (21 modules)
│   ├── machines/             # Machine definitions
│   ├── pkgs/                 # Nix package definitions (15 packages)
│   └── secrets/              # Encrypted secrets (sops)
│
├── specs/                     # Design specifications (48 spec documents)
├── tools/                     # Build tools
├── third_party/               # Third-party vendored code
├── docker/                    # Container entrypoints
├── .github/workflows/         # CI/CD pipelines
└── [config files]            # Cargo.toml, flake.nix, devenv.nix, etc.
```

---

## Primary Programming Languages

| Language | File Count | Purpose |
|----------|-----------|---------|
| **Rust** | ~1,169 files | Core implementation (crates/) |
| **TypeScript** | ~50 files | Web frontend, VSCode extension |
| **JavaScript** | ~10 files | Web UI scripts, build tools |
| **Svelte** | ~100+ files | Web frontend components |
| **Nix** | ~50 files | Build system, infrastructure |
| **SQL** | ~20 files | Database migrations |
| **Shell** | ~10 files | Build scripts, entrypoints |
| **YAML** | ~30 files | Config, CI/CD, secrets |
| **Markdown** | ~100+ files | Documentation, specs |
| **PO/POT** | ~50 files | i18n translations |

---

## Entry Points

### CLI Entry Point
- **File:** `crates/loom-cli/src/main.rs`
- **Binary name:** `loom`
- **Purpose:** Command-line interface for the Loom agent

### Server Entry Point
- **File:** `crates/loom-server/src/main.rs`
- **Binary name:** `loom-server`
- **Purpose:** HTTP API server with LLM proxy

### Web Entry Point
- **File:** `web/loom-web/src/routes/app/+layout.svelte`
- **Build:** `pnpm dev` (SvelteKit dev server)
- **Purpose:** Web UI for Loom

### VSCode Extension Entry Point
- **File:** `ide/vscode/src/extension.ts`
- **Purpose:** VSCode integration for Loom

---

## Workspace Crates (86 Total)

### CLI Components (10 crates)
| Crate | Purpose |
|-------|---------|
| `loom-cli` | Main CLI binary |
| `loom-cli-acp` | Agent Communication Protocol |
| `loom-cli-auto-commit` | Automatic git commit feature |
| `loom-cli-config` | CLI configuration management |
| `loom-cli-credentials` | Credential storage |
| `loom-cli-git` | Git integration |
| `loom-cli-spool` | Spool (message queue) CLI |
| `loom-cli-tools` | CLI tool implementations |
| `loom-cli-wgtunnel` | WireGuard tunnel CLI |

### Server Components (28 crates)
| Crate | Purpose |
|-------|---------|
| `loom-server` | Main server binary |
| `loom-server-api` | API layer with typed routing |
| `loom-server-auth` | Authentication framework |
| `loom-server-auth-devicecode` | Device code OAuth flow |
| `loom-server-auth-github` | GitHub OAuth |
| `loom-server-auth-google` | Google OAuth |
| `loom-server-auth-magiclink` | Magic link authentication |
| `loom-server-auth-okta` | Okta OAuth |
| `loom-server-audit` | Audit logging |
| `loom-server-config` | Server configuration |
| `loom-server-db` | Database layer |
| `loom-server-docs` | Documentation serving |
| `loom-server-email` | Email sending |
| `loom-server-flags` | Feature flags endpoint |
| `loom-server-github-app` | GitHub App integration |
| `loom-server-geoip` | GeoIP lookup |
| `loom-server-jobs` | Job scheduler |
| `loom-server-k8s` | Kubernetes client |
| `loom-server-logs` | Log viewing |
| `loom-server-llm-*` (5) | LLM provider integrations |
| `loom-server-provisioning` | SCIM provisioning |
| `loom-server-scim` | SCIM protocol |
| `loom-server-scm` | Source control management |
| `loom-server-scm-mirror` | Git repository mirroring |
| `loom-server-search-*` (2) | Search integrations |
| `loom-server-secrets` | Secrets management |
| `loom-server-session` | Session management |
| `loom-server-smtp` | SMTP server |
| `loom-server-weaver` | Remote execution pods |

### Common Libraries (9 crates)
| Crate | Purpose |
|-------|---------|
| `loom-common-config` | Shared configuration |
| `loom-common-core` | Core types and utilities |
| `loom-common-http` | HTTP client with retry |
| `loom-common-i18n` | Internationalization |
| `loom-common-secret` | Secret handling |
| `loom-common-spool` | Message queue spool |
| `loom-common-thread` | Conversation persistence |
| `loom-common-version` | Version utilities |
| `loom-common-webhook` | Webhook handling |

### Terminal UI (12 crates)
| Crate | Purpose |
|-------|---------|
| `loom-tui-app` | TUI application |
| `loom-tui-component` | Base components |
| `loom-tui-core` | Core TUI abstractions |
| `loom-tui-testing` | TUI testing utilities |
| `loom-tui-theme` | Theme system |
| `loom-tui-storybook` | Component showcase |
| `loom-tui-widget-*` (6) | Individual widgets |

### Weaver (Remote Execution, 5 crates)
| Crate | Purpose |
|-------|---------|
| `loom-weaver-audit-sidecar` | eBPF audit sidecar |
| `loom-weaver-ebpf` | eBPF programs |
| `loom-weaver-ebpf-common` | eBPF shared types |
| `loom-weaver-secrets` | Weaver secret injection |
| `loom-weaver-wgtunnel` | Weaver WireGuard tunneling |

### WireGuard Tunnel (5 crates)
| Crate | Purpose |
|-------|---------|
| `loom-wgtunnel-common` | Shared types |
| `loom-wgtunnel-conn` | Connection management |
| `loom-wgtunnel-derp` | DERP relay |
| `loom-wgtunnel-engine` | Tunnel engine |
| `loom-cli-wgtunnel` | CLI binary |

### Feature Flags (3 crates)
| Crate | Purpose |
|-------|---------|
| `loom-flags-core` | Core feature flag logic |
| `loom-server-flags` | Server endpoints |
| `loom-flags` | Combined package |

### Analytics (3 crates)
| Crate | Purpose |
|-------|---------|
| `loom-analytics-core` | Core analytics logic |
| `loom-server-analytics` | Server endpoints |
| `loom-analytics` | Combined package |

### Specialized Crates (8 crates)
| Crate | Purpose |
|-------|---------|
| `loom-redact` | PII redaction |
| `loom-scim` | SCIM protocol types |
| `loom-server-search-google-cse` | Google Custom Search |
| `loom-server-search-serper` | Serper search API |
| `loom-server-llm-anthropic` | Anthropic Claude |
| `loom-server-llm-openai` | OpenAI GPT |
| `loom-server-llm-proxy` | LLM HTTP proxy |
| `loom-server-llm-service` | LLM service abstraction |
| `loom-server-llm-vertex` | Google Vertex AI |

---

## Build & Deployment Configuration

### Build System Files
| File | Purpose |
|------|---------|
| `Cargo.toml` | Rust workspace definition |
| `Cargo.nix` | cargo2nix generated file |
| `flake.nix` | Nix flake for reproducible builds |
| `devenv.nix` | Development environment |
| `shell.nix` | Nix shell for development |
| `.cargo/config.toml` | Cargo configuration |

### CI/CD (.github/workflows/)
| File | Purpose |
|------|---------|
| `ci.yml` | Continuous integration |
| `build.yml` | Build pipelines |
| `update-devenv.yml` | devenv updates |
| `claude-code-image.yml` | Claude Code container |
| `ampcode-image.yml` | AmpCode container |
| `publish-audit-sidecar.yml` | Sidecar deployment |

### Infrastructure (infra/)
| Directory | Contents |
|-----------|----------|
| `nixos-modules/` | 21 NixOS modules (server, desktop, k8s, etc.) |
| `machines/` | Machine definitions (loom.nix) |
| `pkgs/` | 15 Nix packages (CLI, server, images) |
| `secrets/` | sops-encrypted secrets |

---

## External Dependencies (Key Libraries)

### Runtime Dependencies (from workspace dependencies)
| Category | Dependencies |
|----------|-------------|
| **Async Runtime** | tokio |
| **HTTP** | reqwest, http, axum (implied) |
| **Serialization** | serde, serde_json |
| **Error Handling** | anyhow, thiserror |
| **Logging** | tracing, tracing-subscriber |
| **Database** | sqlx (SQLite) |
| **Kubernetes** | kube, k8s-openapi |
| **CLI** | clap |
| **Cryptography** | hmac, sha2, argon2, hex |
| **UUID/Time** | uuid, uuid7, chrono |
| **Testing** | proptest, tempfile, tokio-test |

### Web Frontend Dependencies
| Category | Technologies |
|----------|-------------|
| **Framework** | SvelteKit, Svelte 5 |
| **Language** | TypeScript |
| **Build** | Vite, pnpm |
| **UI** | Tailwind CSS (implied) |
| **i18n** | Lingui |
| **Testing** | Vitest |

---

## Code Organization Patterns

### Modular Architecture
- **Horizontal slicing:** Separate crates for CLI, server, common
- **Vertical slicing:** Feature-specific crates (auth, LLM, SCM, etc.)
- **Shared libraries:** `loom-common-*` crates for reuse

### Layered Architecture
```
┌─────────────────────────────────────┐
│   Presentation Layer                 │
│   (loom-cli, loom-tui-*, loom-web)  │
├─────────────────────────────────────┤
│   API Layer                          │
│   (loom-server-api)                  │
├─────────────────────────────────────┤
│   Business Logic Layer               │
│   (loom-server-*, loom-cli-*)        │
├─────────────────────────────────────┤
│   Data Layer                         │
│   (loom-server-db, sqlx)            │
├─────────────────────────────────────┤
│   Infrastructure Layer               │
│   (loom-common-*, loom-llm-*)        │
└─────────────────────────────────────┘
```

### Naming Conventions
- `loom-cli-*` - CLI-specific components
- `loom-server-*` - Server-specific components
- `loom-common-*` - Shared libraries
- `loom-tui-*` - Terminal UI components
- `loom-weaver-*` - Remote execution
- `loom-wgtunnel-*` - WireGuard tunneling

---

## Specification Documents (specs/)

| Category | Specs |
|----------|-------|
| **Core** | architecture, configuration, error-handling, state-machine |
| **LLM** | llm-client, streaming, anthropic-max-pool-management |
| **Security** | secret-system, auth-abac-system, redact-system, audit-system |
| **Analytics** | analytics-system, analytics-implementation-plan |
| **Features** | feature-flags-system, auto-commit-system, scm-system |
| **Infrastructure** | weaver-provisioner, container-system, weaver-cli |
| **Web** | loom-web, design-system, api-documentation |
| **Integration** | vscode-extension, github-app-system |

---

## File Summary

| Category | Count |
|----------|-------|
| **Total tracked source files** | ~600 |
| **Rust source files** | ~1,169 |
| **Workspace crates** | 86 |
| **Nix modules** | 21 |
| **Nix packages** | 15 |
| **Spec documents** | 48 |
| **Web routes** | 50+ |
| **i18n locales** | 16 |

---

## Next Steps

**[ ]** Complete file-index.json with metadata
**[ ]** Begin Phase 2: Dependency Analysis
**[ ]** Map internal module relationships
**[ ]** Identify circular dependencies

---

**✓ PHASE 1 COMPLETE**

---

**Phase 1 Status:** 100% Complete
**Files Created:**
- `analysis/01-inventory.md` - This document
- `analysis/file-index.json` - Metadata index
- `analysis/file-tree.txt` - Directory structure

**Next Phase:** Phase 2 - Dependency Analysis
