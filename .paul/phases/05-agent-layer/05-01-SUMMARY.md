---
phase: 05-agent-layer
plan: 05-01
status: complete
completed: 2026-08-21
---

# Summary: 05-01 Agent anatomy + contracts

## What was built
- `learn/agents/index.md` (472 lines) — frontmatter contract, least-privilege tool scoping, and the caller contract, grounded in `gsd-core/references/agent-contracts.md`.

## Acceptance criteria
| AC | Result | Evidence |
|----|--------|----------|
| AC-1 Construction not usage | ✅ | Centres on what a definition must specify to be safe to spawn |
| AC-2 Traceable | ✅ | 30 citations; `bin/lib` references correctly disclosed as generated + gitignored |
| AC-3 Strict build | ✅ | Exits 0 after nav wiring |

## Notes
The ground-truth block fed into the agent prompts worked: this page cites `gsd-core/bin/lib/frontmatter.cjs`
and explicitly labels it "a **generated, gitignored**" artifact — the disclosure the repo's own
ARCHITECTURE.md omits.

## Post-unify correction (2026-08-21)

Verification reported after unify. Four corrections, each independently re-verified:

1. **`<inputs>` described as a "common section"** — it appears in 3 of 35 agent files,
   among the rarest tags. The sentence attached counts to its neighbours but not to
   itself, which hid the problem. Replaced with the genuinely common tags:
   `<success_criteria>` (30), `<execution_flow>` (19), `<structured_returns>` (14),
   `<project_context>` (13).
2. **"names six concrete calls" followed by seven** — count word corrected.
3. **"four deliberate scope boundaries"** — the cited test lists five.
4. **"the two heaviest agents are both artifact-based"** — the two named agents are
   classified `sentinel-match`, and "heaviest" was undefined. Reworded to
   "largest-by-file-size".
