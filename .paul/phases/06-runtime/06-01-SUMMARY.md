---
phase: 06-runtime
plan: 06-01
status: complete
completed: 2026-08-21
---

# Summary: 06-01 Runtime module map

## What was built
- `learn/runtime/index.md` (513 lines) — 194 modules grouped into families with a PlantUML diagram, the ADR-457 build-at-publish pipeline, and the code-vs-prompt boundary argument.

## Acceptance criteria
| AC | Result | Evidence |
|----|--------|----------|
| AC-1 Construction not usage | ✅ | Central argument is what deserves deterministic code |
| AC-2 Traceable | ✅ | 79 citations — the densest page; build-artifact status disclosed |
| AC-3 Strict build | ✅ | Exits 0 |

## Notes
Corrected counts (194 .cts, not 199) are carried correctly. Cited `src/` subdirectories
(health-diagnostic-rules, installer-migrations, host-integration-adapters, observability) all verified to exist.

## Post-unify correction (2026-08-21)

Verification reported after unify. Four corrections, each independently re-verified:

1. **"All 34 modules with a `cmd*` entry point use `export =`. Zero use ESM named
   exports"** — false, and the "it is not a tendency, it is total" framing with it. Five
   modules use ESM named exports (`broken-windows`, `estimate-cli`, `git-base-branch`,
   `teams-status`, `worktree-base-ref`), all dispatched from `gsd-tools.cjs`. Reworded to
   "thirty of the 35 … a strong tendency, not an absolute".
2. **"`planning-inspect.cts` is the only one writing `export =` against a named object"** —
   seven do. Corrected.
3. **"Seven modules exceed 100 KB"** — eight. `src/install-engine.cts` (104,430 B) was
   missing. Corrected in both places, including the summary table at line 532.
4. **"Most of the eight tracked `bin/lib` files are generator output"** — only 3 of 8
   carry a generated marker.
