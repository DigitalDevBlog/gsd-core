---
phase: 09-build-your-own
plan: 09-01
status: complete
completed: 2026-08-22
---

# Summary: 09-01 Transferable skills + reuse/adapt/drop

## What was built
- `learn/build-your-own/index.md` (561 lines) — the transferable design skills and a reuse / adapt / drop verdict per GSD Core choice, justified by team scale.

## Acceptance criteria
| AC | Result | Evidence |
|----|--------|----------|
| AC-1 Construction not usage | ✅ | Every verdict is a decision with a stated scale threshold, not an 'it depends' |
| AC-2 Traceable | ⚠️ → ✅ after corrections | Three claim defects fixed, two of them direct cross-page contradictions |
| AC-3 Strict build | ✅ | Exits 0 |

## Corrections applied after verification
1. **"linted by 22 bespoke rules"** — contradicted `learn/runtime/index.md:362`, which
   correctly says `eslint-rules/` holds 22 rules but the `src/**/*.cts` block promotes
   **nine** to `error`. Verified: 9. Corrected.
2. **"swept by the 40 \`lint-*\` scripts"** — 40 exist, but only **28** run in `lint:ci`
   (reproduced from `package.json`). Corrected.
3. **"\`requires:\` exists purely so lint-skill-deps can prove closures"** — false, and
   contradicted by this same page 150 lines later. `src/install-profiles.cts:121`
   (`parseRequires`) computes the install closure. Reworded to the property that actually
   makes it safe: both consumers are build/install-time, not runtime.
