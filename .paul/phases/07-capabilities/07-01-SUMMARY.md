---
phase: 07-capabilities
plan: 07-01
status: complete
completed: 2026-08-21
---

# Summary: 07-01 Capability anatomy + lifecycle

## What was built
- `learn/capabilities/index.md` (672 lines) — capability.json shape, the load → activate → consent → trust → lock lifecycle grounded per module, and a PlantUML lifecycle diagram.

## Acceptance criteria
| AC | Result | Evidence |
|----|--------|----------|
| AC-1 Construction not usage | ✅ | Treats the capability system as a plugin trust problem, not a feature list |
| AC-2 Traceable | ⚠️ → ✅ after corrections | See below |
| AC-3 Strict build | ✅ | Exits 0 |

## Corrections applied after verification
1. **The headline design lesson was false.** The page claimed the env-var denylist exists
   in two copies with *different* membership, and concluded "here the divergence is the
   design." Reproduced both sets: `DENIED_LANE_ENV_KEYS`
   (`gsd-core/bin/lib/capability-validator.cjs:930-937`) and `EXECUTION_PRIMITIVE_ENV`
   (`src/capability-trust.cts:1389-1394`) each hold **20 names with an empty set
   difference in both directions** — identical. Rewritten around the real justification,
   which is *layering* (`src/capability-trust.cts:1667-1677`: one copy refuses, the other
   warns on manifests the refusing layer would reject), plus the honest note that nothing
   tests the two lists stay in sync.
2. **DUR-6 misattributed to `promoteStagingToFinal`.** That function is
   `src/capability-lifecycle.cts:547-577` and contains no such block; DUR-6 at
   `:1649-1661` lives in `reconcileCapabilities`' rollback branch. Corrected.
3. **`review.models.<slug>`** is no longer a central dynamic pattern — the validator's own
   comment names the family in the past tense. Flagged for the example to be swapped.
4. **"the only place in the repo I found it modelled"** — false; `src/loop-resolver.cts:468-474`
   models the same prompt-injection threat class. Another superlative that did not hold.
