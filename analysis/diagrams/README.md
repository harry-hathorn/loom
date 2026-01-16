# Loom Architecture & Workflow Diagrams

This directory contains Mermaid diagrams visualizing the Loom codebase architecture, dependencies, and workflows.

## Module Dependency Maps

| File | Description |
|------|-------------|
| `module-map-01.mermaid` | Layer Architecture |
| `module-map-02.mermaid` | LLM Provider Dependencies |
| `module-map-03.mermaid` | Authentication Provider Dependencies |
| `module-map-04.mermaid` | Weaver (Remote Execution) Dependencies |
| `module-map-05.mermaid` | WireGuard Tunnel Dependencies |
| `module-map-06.mermaid` | TUI Widget Dependencies |
| `module-map-07.mermaid` | Feature Flags & Analytics Dependencies |
| `module-map-08.mermaid` | Complete Crate Dependency Graph |

## Workflow Diagrams

| File | Description |
|------|-------------|
| `workflow-01.mermaid` | Agent Conversation Flow |
| `workflow-02.mermaid` | Tool Execution Flow |
| `workflow-03.mermaid` | Authentication Flow (OAuth) |
| `workflow-04.mermaid` | Thread Persistence Flow |
| `workflow-05.mermaid` | Auto-Commit Flow |
| `workflow-06.mermaid` | LLM Request Flow |
| `workflow-07.mermaid` | Device Code Flow |
| `workflow-08.mermaid` | Feature Flag Evaluation Flow |
| `workflow-09.mermaid` | Weaver Provisioning Flow |
| `workflow-10.mermaid` | Analytics Event Flow |
| `workflow-11.mermaid` | Search Flow |
| `workflow-12.mermaid` | Secret Injection Flow |

## Architecture Diagrams

| File | Description |
|------|-------------|
| `architecture-01.mermaid` | Overall System Architecture |
| `architecture-02.mermaid` | Agent State Machine |
| `architecture-03.mermaid` | Database Schema Architecture |
| `architecture-04.mermaid` | LLM Provider Architecture |
| `architecture-05.mermaid` | Authentication & Authorization Flow |
| `architecture-06.mermaid` | Multi-Tenant Architecture |
| `architecture-07.mermaid` | Security Architecture |
| `architecture-08.mermaid` | Feature Flag Evaluation Architecture |
| `architecture-09.mermaid` | Analytics Processing Architecture |
| `architecture-10.mermaid` | Tool Execution Architecture |
| `architecture-11.mermaid` | Weaver Lifecycle Architecture |
| `architecture-12.mermaid` | Component Dependency Overview |

## Viewing

Each `.mermaid` file contains a single diagram that can be rendered:
- **GitHub**: Automatically renders when viewing the file
- **VS Code**: Use the "Mermaid Preview" extension
- **Online**: Paste content into [Mermaid Live Editor](https://mermaid.live)

## Documentation

For detailed documentation, see:
- `../02-dependencies.md` - Module dependency analysis
- `../04-business-logic.md` - Workflow documentation
- `../06-architecture.md` - Architecture documentation
