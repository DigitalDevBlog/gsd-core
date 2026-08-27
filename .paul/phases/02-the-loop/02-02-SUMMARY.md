---
phase: 02-the-loop
plan: 02
status: complete
completed: 2026-08-21
---

# Summary: 02-02 Stage internals

## What was built
- `learn/loop/stages.md` (470 lines) — per-stage deep dive: where a stage spec lives,
  the dispatcher as routing header, the shared grammar of a workflow spec, what is
  distinctive per stage, how state moves, what makes a markdown file executable, and a
  reconstruction guide.
- `mkdocs.yml` — nav entry added; nav validation tightened.

## Acceptance criteria

| AC | Result | Evidence |
|----|--------|----------|
| AC-1 Stage explained as built artifact | ✅ | Sections 1-7 cover spec location, grammar, executability and reconstruction |
| AC-2 Claims cite real source | ✅ | All path citations resolve after the prefix fix (`commands/gsd/*.md` is a valid 71-file glob) |
| AC-3 Strict build with page in nav | ⚠️ Failed on first attempt, then fixed | See below |

## The defect worth recording
The writing agent created `learn/loop/stages.md` but **never added it to `mkdocs.yml`
nav** — and `mkdocs build --strict` still exited 0. MkDocs 1.6 logs "pages exist in the
docs directory but are not included in nav" at **INFO**, and `--strict` only promotes
WARNING and above. So an unreachable page passed the plan's own build gate.

Fixes applied:
1. Added the nav entry.
2. Added to `mkdocs.yml`:
   ```yaml
   validation:
     nav:
       omitted_files: warn
     links:
       absolute_links: warn
   ```
   Omitted-from-nav is now a WARNING, which `--strict` turns into a build failure.

**Carry-forward:** every remaining content phase writes new pages. Without this change,
each one could have shipped an orphaned page invisibly. The build gate in plans 03-01
through 09-02 is now actually load-bearing.

## Issues logged
- **ISSUE-03 (resolved during this plan):** `--strict` alone does not catch orphaned
  pages. Resolved via explicit `validation:` config.

## Files changed
learn/loop/stages.md, mkdocs.yml
