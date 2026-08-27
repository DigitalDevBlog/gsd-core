# Understanding GSD Core

This site reverse-engineers **GSD Core** — a meta-prompting, context-engineering and
spec-driven development system for AI coding agents — and explains *how it is built*.

It is not a usage manual. The repo's own `docs/` covers operating GSD Core. This site
answers a different question:

!!! quote "Core value"
    Understand in detail how a framework like GSD Core is actually built — deeply enough
    to reconstruct something similar yourself.

Every section is written from the reconstruction angle: what the piece is, how it is
constructed, what design decision it encodes, and what you would do differently if you
were building your own.

## What GSD Core is made of

The subject is large. Measured against this repo at v1.11.0:

| Stratum | Surface |
|---------|---------|
| Skills | 71 skills, generated from 71 `commands/gsd/*.md` — 6 `ns-*` namespace routers plus 65 concrete commands |
| Agents | 35 subagent definitions with scoped tool access |
| Runtime | 194 `.cts` modules, 26 hooks, an MCP server, an installer |
| Capabilities | 46 capability packs targeting Claude, Cursor, Codex, Gemini, Kilo, Kimi, Windsurf, Copilot and more |

The runtime is the part that most distinguishes GSD Core from a pure prompt framework —
and it gets its own section.

## The map

| Section | What it covers |
|---------|----------------|
| [The Loop](loop/index.md) | The spec-driven cycle and why it is shaped that way |
| [Files & State](files/index.md) | How `.planning/` carries state between stages |
| [Skills](skills/index.md) | Skill anatomy and how `/gsd` routes into 71 of them |
| [Agents](agents/index.md) | Subagent contracts and orchestration patterns |
| [Runtime](runtime/index.md) | Where the prompt layer stops and code begins |
| [Capabilities](capabilities/index.md) | One framework, many harnesses |
| [Architecture](architecture/index.md) | How the strata compose into one system |
| [Build Your Own](build-your-own/index.md) | Transferable skills and a minimal blueprint |
