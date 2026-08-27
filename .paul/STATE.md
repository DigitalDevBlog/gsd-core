# Project State

## Project Reference

See: .paul/PROJECT.md (updated 2026-08-21)

**Core value:** Understand in detail how a framework like GSD Core is actually built — deeply enough to reconstruct something similar yourself.
**Current focus:** v0.1 Initial Release — ✅ COMPLETE (9/9 phases)

## Current Position

Milestone: v0.1 Initial Release
Phase: 9 of 9 (Build Your Own) — ✅ Complete
Plan: 09-02 complete — all 19 plans done
Status: ✅ v0.1 milestone COMPLETE — 18 pages, strict build clean
Last activity: 2026-08-22 00:10 — v0.1 complete: 9 phases, 19 plans, 18 pages, 8,462 lines

Progress:
- Milestone: [██████████] 100%
- Phase 9: [██████████] 100%

## Loop Position

Current loop state:
```
PLAN ──▶ APPLY ──▶ UNIFY
  ✓        ✓        ✓     [v0.1 COMPLETE — ready for next milestone PLAN]
```

## Accumulated Context

### Decisions

| Decision | Phase | Impact |
|----------|-------|--------|
| Content = how it is built, not how it is used | Init | Shapes every content section; no usage manuals |
| Build-your-own is the spine, not the final chapter | Init | Each section must answer "how would I rebuild this?" |
| MkDocs Material + mike + Kroki (mirrors "Understanding PAUL") | Init | Fixes the doc toolchain |
| Runtime gets its own section | Init | Largest structural difference from PAUL — 194 .cts modules, 26 hooks, MCP server |
| Subject surface is large (71 skills, 35 agents, 46 capabilities, 194 runtime modules) | Init | Phase sizing during `/paul:plan` must account for the scale |
| All 19 plans for v0.1 written up front | Phase 2 | Content phases execute via workflow without re-planning each time |
| `validation.nav.omitted_files: warn` added | 02-02 | `--strict` alone let an orphaned page pass; build gate now load-bearing |
| Workflow agents must not edit mkdocs.yml | 03/04 | Concurrent agents race on the nav block; orchestrator wires nav after |
| Counts corrected: 194 .cts, 26 hooks, 71 commands (6 ns-* + 65 concrete) | 02-02 | Earlier figures were wrong; verified against source |
| Do NOT unify a phase before its verification stage reports | 03/04 | Phases 3-4 were unified early and shipped 7 false claims; all corrected post-hoc |
| Path resolution != claim verification | 03/04 | A citation can resolve while the sentence around it is false; both must be checked |
| Superlatives are the dominant failure mode | 05/06/07 | "the only/always/every/zero/total" produced most false claims; verify stage now hunts them explicitly |
| Verify the verifiers | 03/04, 06-03 | Two verifier objections were themselves wrong; one of my own "fixes" introduced an error and was reverted |

### Deferred Issues

| Issue | Origin | Effort | Revisit |
|-------|--------|--------|---------|
| ISSUE-01: diagrams load from public kroki.io at reader page-load; self-host or inline SVG at build time | 01-02 | M | Before first public deploy |
| ISSUE-02: version switcher unverified until a first \`mike deploy\` | 01-03 | S | When owner decides to publish |

### Blockers/Concerns
None yet.

## Session Continuity

Last session: 2026-08-21 21:47
Stopped at: v0.1 milestone complete — all 9 phases unified
Next action: Resolve ISSUE-01 (self-host Kroki) before any public deploy, or start a v0.2 milestone
Resume file: .paul/ROADMAP.md

---
*STATE.md — Updated after every significant action*
