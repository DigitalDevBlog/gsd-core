# Roadmap: Understanding GSD Core

## Overview
A versioned MkDocs Material site that reverse-engineers GSD Core's internals — the
spec-driven loop, the state model, the skill and agent layers, the runtime, the
multi-harness capability system — written so a reader could reconstruct a comparable
framework themselves. The journey runs from standing up the doc site, through
documenting each stratum of GSD Core, to a build-your-own synthesis.

## Current Milestone
**v0.1 Initial Release** (v0.1.0)
Status: ✅ Complete
Phases: 9 of 9 complete

## Phases

**Phase Numbering:**
- Integer phases (1, 2, 3): Planned milestone work
- Decimal phases (2.1, 2.2): Urgent insertions (marked with [INSERTED])

Phases execute in numeric order.

| Phase | Name | Plans | Status | Completed |
|-------|------|-------|--------|-----------|
| 1 | Site Foundation | 3 | ✅ Complete | 2026-08-21 |
| 2 | The GSD Loop | 2 | ✅ Complete | 2026-08-21 |
| 3 | Files & State Model | 2 | ✅ Complete | 2026-08-21 |
| 4 | The Skill Layer | 2 | ✅ Complete | 2026-08-21 |
| 5 | The Agent Layer | 2 | ✅ Complete | 2026-08-21 |
| 6 | The Runtime | 3 | ✅ Complete | 2026-08-21 |
| 7 | Capabilities & Multi-Harness | 2 | ✅ Complete | 2026-08-21 |
| 8 | Architecture Synthesis | 1 | ✅ Complete | 2026-08-22 |
| 9 | Build Your Own | 2 | ✅ Complete | 2026-08-22 |

## Phase Details

### Phase 1: Site Foundation

**Goal:** A running MkDocs Material site with a nav skeleton for all eight content areas,
Kroki diagram rendering wired in, and `mike`-based version switching.
**Depends on:** Nothing (first phase)
**Research:** Likely (docs_dir relocation, Kroki plugin config, mike versioning)

**Scope:**
- MkDocs Material scaffold (`mkdocs.yml`, `requirements.txt`, `learn/`)
- Navigation skeleton covering all eight deliverables
- Diagram-as-code via `mkdocs-kroki-plugin==0.9.0` (PlantUML / Excalidraw)
- Versioning with `mike`
- Local strict build verified

**Plans:**
- [x] 01-01: MkDocs Material scaffold + nav skeleton + local strict build
- [x] 01-02: Diagram tooling (Kroki / PlantUML / Excalidraw)
- [x] 01-03: Versioning with mike

### Phase 2: The GSD Loop

**Goal:** The spec-driven cycle explained as a construction: new-project → plan-phase →
execute-phase → verify → milestone, why each stage exists, and how the stages hand off.
**Depends on:** Phase 1 (site to publish into)
**Research:** Unlikely (source is in-repo)

**Scope:**
- Loop overview + diagram
- Per-stage deep dives grounded in `skills/gsd-*` source

**Plans:**
- [x] 02-01: Loop overview page + loop diagram
- [x] 02-02: Per-stage deep dives (plan / execute / verify)

**Status: ✅ Complete (2026-08-21) — 2/2 plans**

### Phase 3: Files & State Model

**Goal:** How `.planning/` carries state between stages — ROADMAP, PLAN, RESEARCH,
VERIFICATION artifacts and the ledger — and the design rules that keep them coherent.
**Depends on:** Phase 2 (the loop the artifacts serve)
**Research:** Unlikely

**Scope:**
- Artifact catalogue + lifecycle
- Cross-stage data-flow diagram
- Ledger mechanics

**Plans:**
- [x] 03-01: The .planning/ artifact set + lifecycle
- [x] 03-02: Ledger + cross-stage data flow (with diagram)

**Status: ✅ Complete (2026-08-21) — 2/2 plans**

### Phase 4: The Skill Layer

**Goal:** Anatomy of a GSD skill, how the namespaced `commands/gsd/` surface routes into 71 of them,
and the design rules that make a skill file executable by a model.
**Depends on:** Phase 2
**Research:** Unlikely

**Scope:**
- Skill file anatomy (frontmatter, body, composition)
- The `/gsd` router and dispatch
- Skill catalogue grouped by loop position

**Plans:**
- [x] 04-01: Skill anatomy + the /gsd router
- [x] 04-02: Skill catalogue by role and loop position

**Status: ✅ Complete (2026-08-21) — 2/2 plans**

### Phase 5: The Agent Layer

**Goal:** How the 35 subagent definitions work — their contracts, tool scoping, spawn
patterns, and the orchestration built on top of them.
**Depends on:** Phase 4 (skills spawn agents)
**Research:** Unlikely

**Scope:**
- Agent definition anatomy + least-privilege tool scoping
- Spawn and orchestration patterns (fan-out, verification agents)

**Plans:**
- [x] 05-01: Agent anatomy + contracts
- [x] 05-02: Orchestration patterns

**Status: ✅ Complete (2026-08-21) — 2/2 plans**

### Phase 6: The Runtime

**Goal:** The part PAUL has no equivalent of — 194 `.cts` modules, 26 hooks, the MCP
server, and the installer: what runs as code, why, and where the prompt layer stops.
**Depends on:** Phases 4-5 (the layers the runtime serves)
**Research:** Likely (largest and least-documented surface)

**Scope:**
- `src/*.cts` module map and the code/prompt boundary
- The hook system and its lifecycle events
- The MCP server and the installer/bootstrap

**Plans:**
- [x] 06-01: src/*.cts module map + the code-vs-prompt boundary
- [x] 06-02: The hook system
- [x] 06-03: MCP server + installer/bootstrap

**Status: ✅ Complete (2026-08-21) — 3/3 plans**

### Phase 7: Capabilities & Multi-Harness

**Goal:** How one framework targets 46 capabilities across Claude, Cursor, Codex, Gemini,
Kilo, Kimi, Windsurf and Copilot — the adapter design and what it costs.
**Depends on:** Phase 6 (capability loading is runtime code)
**Research:** Likely

**Scope:**
- Capability anatomy, lifecycle, trust/consent model
- Harness adapters (declarative vs imperative)

**Plans:**
- [x] 07-01: Capability anatomy + lifecycle
- [x] 07-02: Harness adapters and the portability strategy

**Status: ✅ Complete (2026-08-21) — 2/2 plans**

### Phase 8: Architecture Synthesis

**Goal:** All strata assembled into one picture — how prompt layer, state layer, runtime
and capabilities compose, with an end-to-end walkthrough.
**Depends on:** Phases 2-7
**Research:** Unlikely (synthesis of prior phases)

**Scope:**
- System diagram + end-to-end trace of one real command
- Design principles the architecture embodies

**Plans:**
- [x] 08-01: Architecture synthesis + end-to-end walkthrough

**Status: ✅ Complete (2026-08-22) — 1/1 plans**

### Phase 9: Build Your Own

**Goal:** The payoff — transferable design skills and a minimal blueprint for building a
comparable framework, with a reuse/adapt/drop guide against GSD Core's choices.
**Depends on:** Phase 8
**Research:** Unlikely

**Scope:**
- Transferable skills + reuse/adapt/drop guide
- Minimal framework blueprint with a grow-from-here path

**Plans:**
- [x] 09-01: Transferable skills + reuse/adapt/drop guide
- [x] 09-02: Minimal blueprint + grow-from-here path

**Status: ✅ Complete (2026-08-22) — 2/2 plans**

---
*Roadmap created: 2026-08-21*
*Last updated: 2026-08-21 — 9 phases defined for v0.1*
