---
phase: 05-agent-layer
plan: 05-02
status: complete
completed: 2026-08-21
---

# Summary: 05-02 Orchestration patterns

## What was built
- `learn/agents/orchestration.md` (482 lines) — producer/checker bounded loops, fan-out, adversarial checking, with a PlantUML diagram and per-pattern participant tables.

## Acceptance criteria
| AC | Result | Evidence |
|----|--------|----------|
| AC-1 Construction not usage | ✅ | Patterns tied to named agents and real workflow line refs |
| AC-2 Traceable | ✅ | 30 citations; one shorthand path normalised during unify |
| AC-3 Strict build | ✅ | Exits 0 |

## Notes
One citation was written as `steps/per-plan-worktree-gate.md` (relative) while the same page used full
paths elsewhere. The file is real — `gsd-core/workflows/execute-phase/steps/per-plan-worktree-gate.md` —
and the citation was normalised. Worth noting the verification pass initially read as a fabrication; it was
not, and checking before editing avoided removing correct content.

## Post-unify correction (2026-08-21)

Verification reported after unify. Three corrections:

1. **Opening claim "no workflow talks to them one at a time"** — false. Six workflows
   dispatch exactly one agent once (`audit-milestone`, `secure-phase`, `ui-review`,
   `code-review`, `validate-phase`, `explore`). Verified: each has exactly one
   `subagent_type=` occurrence. Opening rewritten.
2. **Cost-governor row wrong for 2 of 3 cited files.** `plan-checker-loop.md:63` bounds at
   **2**, not 3, and contains zero stall-detector references; `code-review-fix.md` has
   `MAX_ITERATIONS=3` but no stall detector. Table now states the bound per site.
3. **A fourth loop site was missing** — `plan-review-convergence.md:31` (`MAX_CYCLES=3`,
   user-settable via `--max-cycles`, with its own stall check at :382). Added, and the
   "two independent literals" passage now notes the family carries three different bounds.

Note: §2's capability-gate claim was checked and is **correct** —
`capabilities/research/capability.json:45-46` has exactly `when: workflow.research` and
`onError: skip`. The verifier's objection applied only to the table row, not §2.
