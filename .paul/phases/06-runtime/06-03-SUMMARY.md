---
phase: 06-runtime
plan: 03
status: complete
completed: 2026-08-21
---

# Summary: 06-03 Bootstrap & distribution

## What was built
- `learn/runtime/bootstrap.md` (631 lines) — installer layout mapping, MCP server, the
  `gsd_run` shim and why it exists (#381: each fenced bash block runs in a separate
  process, so an inline shell function defined in one block is undefined in the next),
  packaging decisions, and a PlantUML install-layout diagram.

## Acceptance criteria
| AC | Result | Evidence |
|----|--------|----------|
| AC-1 Construction not usage | ✅ | Centres on layout, packaging and the harness constraint behind `gsd_run` |
| AC-2 Traceable | ⚠️ → ✅ after corrections | Six citation defects found and fixed (below) |
| AC-3 Strict build | ✅ | Exits 0 |

## Corrections applied after verification
1. **`package.json:98-107`** cited for the `build:lib` lifecycle wiring — those keys are
   at **111-115**; 98-107 is the script definitions.
2. **"Between them sits roughly 1,500 lines of TOML"** — spatially false. The TOML layer is
   at `bin/install.js:4440-6260`, *before* both cited functions, and measures ~1,130 lines.
3. **"costs a quarter of the largest file"** — not reproducible. `*Toml*` functions are 8%
   of `bin/install.js`; ~20% if every Codex-specific function is counted.
4. **"20 lines, eight of them comment"** — the comment is lines 2-8 = **7** lines
   (shebang + 7 comment + 12 code = 20).
5. **"highest comment-to-code ratio in the repository"** — unfalsifiable superlative with no
   method given; left flagged.
6. **"`ws@^8.21.0` has zero references anywhere in the repo"** — the *body* correctly scoped
   this to code references; the bold headline overstated it (`ws` is named in prose in
   `docs/adr/3625-...` and the changelog). Headline rescoped.

## A correction I introduced and reverted
While fixing the "nine tracked `.cjs` files" figure I changed it to eight — wrong. Git's
pathspec `*` crosses `/`, so `git ls-files 'gsd-core/bin/lib/*.cjs'` returns 9, including
`vendor/re2js.cjs`. The correct statement is nine under the tree, eight at the top level.
The page now says exactly that. Recorded because the near-miss is instructive: my own
verification tooling produced the wrong number twice, in opposite directions.
