---
phase: 03-files-and-state
plan: 03-01
status: complete
completed: 2026-08-21
---

# Summary: 03-01 Artifact set + lifecycle

## What was built
- `learn/files/index.md` (735 lines) — the artifact catalogue grouped by role, lifecycle table, and the design rationale for the split.

## Acceptance criteria
| AC | Result | Evidence |
|----|--------|----------|
| AC-1 Artifact set mapped | ✅ | Catalogue grouped by role, not a flat list of 46 |
| AC-2 Lifecycle not catalogue | ✅ | created-when / read-by / updated-by present |
| AC-3 Design rationale explicit | ✅ | Finite-context pressure argued directly |
| AC-4 Traceable + building | ✅ | 85 citations; strict build exits 0 after nav wiring |

## Notes
The page's strongest finding is a genuine drift defect in the framework: `gsd-core/templates/README.md:75`
tells maintainers to edit `CANONICAL_EXACT` in `gsd-core/bin/lib/artifacts.cjs` — a file that no longer
exists. `src/artifacts.cts:2` records why (ADR-457 collapsed it to a TypeScript source of truth). Verified
verbatim against both files during unify.

## Post-unify correction (2026-08-21)

This plan was unified BEFORE the workflow's verification stage reported. That was
premature — citations resolved, but three *claims* did not hold. Corrected after
independent re-verification against source:

1. **"`isCanonicalPlanningFile` has no other consumer"** — false. Four consumers exist,
   including `scripts/lint-planning-artifact-writer-drift.cjs`, which runs in the
   `lint:ci` chain (`package.json:122`). The dependent conclusion that `config.json` is
   an inert registry row was therefore also wrong. Only `milestone.lock` is genuinely
   inert. Page rewritten to the narrower, correct defect.
2. **"`STATE.md` is the only artifact whose write path is a code module"** — false, and
   contradicted by the page's own tables. `src/roadmap.cts:1081,1311` writes `ROADMAP.md`
   via `platformWriteSync`. Reframed around what is actually distinctive: the
   prose-by-agent / frontmatter-by-code split within one file.
3. **Manifest leaf count 77 → 78.** The verifier also challenged the "44 leaf keys"
   figure; an independent count confirmed 44 is correct and the verifier was wrong.
   Only the manifest number was changed.

**AC-2 (traceable) is downgraded from ✅ to ⚠️ at time of original unify, ✅ after these
corrections.** Path resolution is not claim verification — the two were conflated.
