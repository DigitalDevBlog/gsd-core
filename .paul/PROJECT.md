# Understanding GSD Core

## What This Is

A versioned documentation site (MkDocs Material) that reverse-engineers the GSD Core
framework — its spec-driven loop, its skill and agent layers, its runtime, and its
multi-harness capability system. The site is written as a **build-it-yourself guide**,
not a usage manual: every section explains how a piece is constructed and why it is
designed that way, so a reader could reconstruct a comparable agentic development
framework from first principles.

## Core Value

Understand in detail how a framework like GSD Core is actually built — deeply enough to
reconstruct something similar yourself.

## Current State

| Attribute | Value |
|-----------|-------|
| Type | Other (Documentation / Learning) |
| Version | 0.0.0 |
| Status | Initializing |
| Last Updated | 2026-08-21 |

## Requirements

### Core Deliverables

1. **The GSD loop** — new-project → plan-phase → execute-phase → verify → milestone: the
   spec-driven cycle, what each stage does and why the cycle is shaped this way.
2. **Files & state model** — `.planning/`, the ROADMAP / PLAN / RESEARCH / VERIFICATION
   artifacts, and the ledger: how state is carried between stages.
3. **The skill layer** — 71 skills: anatomy of a skill, and how the namespaced
   `commands/gsd/` surface (6 `ns-*` routers + 65 concrete) dispatches into them.
4. **The agent layer** — 35 subagent definitions: their contracts, how they are spawned,
   and the orchestration patterns built on top of them.
5. **The runtime** — `src/*.cts` modules, the hook system, the MCP server, and the
   installer. GSD Core's biggest structural departure from a pure prompt framework.
6. **Capabilities & multi-harness** — how one framework targets 46 capabilities across
   Claude, Cursor, Codex, Gemini, Kilo, Kimi, Windsurf, Copilot and more.
7. **Architecture synthesis** — how the strata fit together as a single system.
8. **Build your own** — the transferable skills, design decisions, and a minimal
   blueprint for constructing a comparable framework.

### Validated (Shipped)
None yet.

### Active (In Progress)
None yet.

### Planned (Next)
Phases to be defined during `/paul:plan`.

### Out of Scope
- Changing or forking the GSD Core source itself — this project documents and learns from it.
- Usage manuals / how-to guides for operating GSD Core — the repo's own `docs/` covers that.

## Constraints

### Technical Constraints
- Doc site generator: **MkDocs Material**.
- Versioned docs with an Antora-style version switcher — via the **`mike`** plugin.
- Diagrams via **`mkdocs-kroki-plugin==0.9.0`** (PlantUML / Excalidraw).
- Documentation must stay **accurate to the actual GSD Core source** in this repo.
- Docs tooling self-contained at repo root; framework source untouched.
- Coexists with the repo's existing `.plans/` and `docs/` — `.paul/` sits alongside them.

### Business Constraints
- Timeline: open-ended (learning pace).

## Key Decisions

| Decision | Rationale | Date | Status |
|----------|-----------|------|--------|
| MkDocs Material + `mike` + Kroki, mirroring the "Understanding PAUL" project | Proven stack for a diagram-heavy, versioned reverse-documentation site | 2026-08-21 | Active |
| **Content = how it is built, not how it is used** | Core value is reconstruction skill; mechanism + rationale transfer, usage instructions do not. Applies to ALL content sections. | 2026-08-21 | Active |
| Build-your-own is the spine, not the final chapter | Every section answers "how would I rebuild this?" rather than "what does it do" | 2026-08-21 | Active |
| Runtime gets its own section | Unlike PAUL (no runtime), GSD Core ships 194 `.cts` modules, 26 hooks and an MCP server — the largest structural difference | 2026-08-21 | Active |
| `.paul/` coexists with the repo's own `.plans/` | The docs effort is planned separately from the framework being documented | 2026-08-21 | Active |

## Success Metrics

| Metric | Target | Current | Status |
|--------|--------|---------|--------|
| Reconstruction capability | From the site alone, a reader could build a GSD-like framework: the loop, skill/agent layers, runtime, and multi-harness design are understood well enough to reproduce the pattern | - | Not started |
| Source accuracy | Every explanation traceable to real GSD Core source in this repo | - | Not started |
| Versioned docs | Docs versioned per GSD Core release, switchable in the site | - | Not started |

## Tech Stack / Tools

| Layer | Technology | Notes |
|-------|------------|-------|
| Doc site | MkDocs Material | Static markdown site generator |
| Versioning | mike | Antora-style version switcher; deploys to gh-pages |
| Diagrams | mkdocs-kroki-plugin==0.9.0 | Renders PlantUML / Excalidraw fences via a Kroki server |
| Diagram server | kroki.io (KROKI_SERVER_URL overridable) | Public by default; can self-host via local Docker |
| Source of truth | GSD Core repo (this repo, v1.11.0) | Docs track the real source |

---
*Created: 2026-08-21*
