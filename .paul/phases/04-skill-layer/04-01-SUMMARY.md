---
phase: 04-skill-layer
plan: 04-01
status: complete
completed: 2026-08-21
---

# Summary: 04-01 Skill anatomy + routing

## What was built
- `learn/skills/index.md` (470 lines) — SKILL.md frontmatter contract, command→skill generation, dispatch, and what makes markdown model-executable.

## Acceptance criteria
| AC | Result | Evidence |
|----|--------|----------|
| AC-1 Construction not usage | ✅ | Centres on the generation pipeline and the converter-drift defect |
| AC-2 Traceable | ✅ | 38 citations; `bin/install.js:64` require verified verbatim |
| AC-3 Strict build | ✅ | Exits 0 after nav wiring |

## Notes
Documents a real duplication defect: `bin/install.js` both requires the compiled converter AND redefines
it with a byte-identical doc comment. Cited `.cjs` paths are gitignored build outputs — the page says so
explicitly and names the `.cts` sources, so the citations are deliberate, not broken.

## Post-unify correction (2026-08-21)

Verification reported after unify. Four corrections, all independently re-verified:

1. **`spike.md` was the wrong example.** Its only `gsd:` occurrence is the frontmatter
   `name:` key, which `extractBody()` strips before reference extraction — so it never
   exercises the self-reference guard at all. Replaced with `graphify` (3 body
   self-references). Six commands genuinely exercise it: explore, fast, graphify,
   progress, quick, workstreams.
2. **`agent` frontmatter semantics were inferred, not evidenced.** Zero of 71 commands
   declare it; no ADR or comment documents what it binds to. Table now says so.
3. **`context` field** — real converter capability, but no command declares it. Noted.
4. **"most diffs are 7–12 changed lines"** — false. Median is 6; 46 of 71 pairs fall in
   4–6. The 23-line `complete-milestone` diff is an outlier, correctly identified.
