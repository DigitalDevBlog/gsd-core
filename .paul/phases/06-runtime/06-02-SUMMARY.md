---
phase: 06-runtime
plan: 06-02
status: complete
completed: 2026-08-21
---

# Summary: 06-02 The hook system

## What was built
- `learn/runtime/hooks.md` (420 lines) — 26 hooks grouped by lifecycle event and purpose, with a PlantUML placement diagram and the why-a-prompt-is-insufficient argument.

## Acceptance criteria
| AC | Result | Evidence |
|----|--------|----------|
| AC-1 Construction not usage | ✅ | Each guard framed by what it prevents that a prompt cannot |
| AC-2 Traceable | ✅ | 38 citations; `hooks/dist/` correctly described as gitignored build output (`.gitignore:18` verified) |
| AC-3 Strict build | ✅ | Exits 0 |

## Notes
Corrected count (26 hook scripts, not 29) carried correctly — `hooks.json` and `managed-hooks-registry.cjs`
are excluded as non-hooks.

## Post-unify correction (2026-08-21)

Verification reported after unify. Two corrections:

1. **"A self-imposed 3-second budget. Every stdin-consuming hook opens identically"** —
   false. Only 7 of 22 `.js` hooks use 3000 ms; others run to 10000
   (`gsd-context-monitor.js:36`, `gsd-windsurf-pre-write.js:66`), 5000
   (`gsd-read-injection-scanner.js:216`) and 8000 (`gsd-config-reload.js:27`). Reframed:
   the *shape* is universal, the value is not.
2. **"commented identically in three files"** — the identical Kimi-stderr string appears in
   four locations across three files, and a fourth file (`gsd-workflow-guard.js:118`) makes
   the same point in *different* wording, so "identically" was wrong for it. Corrected.
