# Hoon & Urbit Agent Reference

Practical reference for writing and understanding Hoon backend code on Urbit, with a primary focus on Gall agents built in the "large app / Tlon-style" tradition. Derived from production codebases (Tlon groups/channels and ~paldev's suite).

## Scope

This repo mixes two kinds of material:
- **General reference**: Hoon/Gall concepts and syntax that apply broadly.
- **How we build**: production defaults for long-lived agents with multiple clients, versioned marks and paths, and compatibility requirements.

The second category is the main focus. Many recommendations here are deliberate house patterns, not universal laws of Hoon.

If the only client for an app is shipped by the same desk, client-facing backwards compatibility is often unnecessary. You may still want backwards compatibility for persisted state, inter-agent protocols, or external integrations.

## Version Assumptions

This reference is derived from contemporary Urbit application code rather than a frozen language specification. Treat it as a practical guide for current Gall-style agent work, and verify details against the target ship/kernel when working near version-sensitive edges.

## Files

| File | What it covers | When to read it |
|---|---|---|
| [fundamentals.md](fundamentals.md) | Subject model, faces, type narrowing, formatting, irregular syntax | Before reading or writing any Hoon |
| [syntax.md](syntax.md) | Practical syntax, stdlib, types, JSON, imports, scries | When you need to look up a specific rune or stdlib gate |
| [simple-agent-primer.md](simple-agent-primer.md) | Small-agent Gall primer, default-agent delegation, minimal versioning | When building a simple or first agent |
| [architecture.md](architecture.md) | Agent lifecycle, ACUR model, marks, versioning, wrapper libraries, generators, threads | When building or modifying a Gall agent |
| [patterns.md](patterns.md) | Composition idioms, error handling, common pitfalls | When writing idiomatic Hoon and avoiding mistakes |

## How to Use

For LLM context, include the files relevant to the task:
- **Reading existing Hoon code**: `fundamentals.md` + `syntax.md`
- **Writing a small agent**: `simple-agent-primer.md` + selected sections of `architecture.md`
- **Writing a production agent**: all five files
- **Maintaining a long-lived app**: `architecture.md` + `patterns.md`
- **Quick syntax lookup**: `syntax.md` alone

## Source Codebases

- **homestead/desk** — Tlon's groups/channels/chat backend (production, ~20 agents, versioned marks and types, wrapper libraries). Used primarily for architectural patterns.
- **suite** — ~paldev's application collection (pals, gossip, rudder, rumors, million, face, etc.). Used primarily for syntactic style and idiom.
