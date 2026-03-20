# Hoon & Urbit Application Reference

LLM-consumable reference for writing and understanding Hoon backend code on Urbit. Derived from production codebases (Tlon groups/channels and ~paldev's suite).

## Files

| File | What it covers | When to read it |
|---|---|---|
| [fundamentals.md](fundamentals.md) | Subject model, faces, type narrowing, formatting, irregular syntax | Before reading or writing any Hoon |
| [syntax.md](syntax.md) | Full rune reference, stdlib, types, JSON, imports, scries | When you need to look up a specific rune or stdlib gate |
| [architecture.md](architecture.md) | Agent lifecycle, ACUR model, marks, versioning, wrapper libraries, generators, threads | When building or modifying a Gall agent |
| [patterns.md](patterns.md) | Composition idioms, error handling, common pitfalls | When writing idiomatic Hoon and avoiding mistakes |

## How to Use

For LLM context, include the files relevant to the task:
- **Reading existing Hoon code**: `fundamentals.md` + `syntax.md`
- **Writing a new agent**: all four files
- **Modifying an existing agent**: `architecture.md` + `patterns.md`
- **Quick syntax lookup**: `syntax.md` alone

## Source Codebases

- **homestead/desk** — Tlon's groups/channels/chat backend (production, ~20 agents, versioned marks and types, wrapper libraries). Used primarily for architectural patterns.
- **suite** — ~paldev's application collection (pals, gossip, rudder, rumors, million, face, etc.). Used primarily for syntactic style and idiom.
