---
phase: 08-architecture
plan: 08-01
status: complete
completed: 2026-08-22
---

# Summary: 08-01 Architecture synthesis

## What was built
- `learn/architecture/index.md` (558 lines) — the strata assembled into one system with a
  PlantUML diagram, an end-to-end command trace, design principles, and an honest seams
  section.

## Acceptance criteria
| AC | Result | Evidence |
|----|--------|----------|
| AC-1 Construction not usage | ✅ | Traces one real command through routing → skill → agents → runtime → hooks |
| AC-2 Traceable | ⚠️ → ✅ after corrections | 4 line-anchor drifts + 6 claim defects fixed |
| AC-3 Strict build | ✅ | Exits 0 |

## Corrections applied after verification
1. **Cross-page contradiction on hook counts.** This page attached "26" to the `hooks/*.js`
   glob; 26 is the `.js` + `.sh` total (22 + 4). Two sibling pages had it right. Fixed in
   prose and in the diagram.
2. **"Dual registration is the exception, not the norm"** — inverted. 9 of the 10 scripts
   `hooks/hooks.json` references also appear in `src/runtime-hooks-surface.cts`, which is
   exactly what `learn/runtime/hooks.md` reports independently.
3. **"It does not parse the model's `Agent()` arguments"** — false. The guard reads
   `data.tool_input` (`:421`), `subagent_type` (`:422`) and the isolation kwarg (`:465`).
   The real point is that it does not *trust* them; reworded.
4. **"the point vocabulary cannot drift"** — false. `src/loop-resolver.cts:70-82` keeps a
   hardcoded fallback copy of all twelve points, and two more copies exist elsewhere.
5. **"They appear only in `tests/fixtures/install-tree/*.json`"** — false; the narrower
   true claim (absent from all prompt markdown) substituted.
6. **Four line anchors off by 2-4** (`:644`→`:647`, `:967`→`:971`, `:1170`→`:1172`,
   `:98`→`:97`) — flagged; surrounding arguments survive.

## The page's best finding
`execute:pre` is a canonical loop point in `loop-host-contract.cjs:51`, in the validator's
accepted set, ships as an empty registry bucket, and has a passing test asserting a step
there is accepted — but `grep -l '"execute:pre"' capabilities/*/capability.json` returns 0.
A fully-supported extension point nothing uses. Independently re-verified.
