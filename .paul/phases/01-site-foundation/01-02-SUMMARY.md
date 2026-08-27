---
phase: 01-site-foundation
plan: 02
status: complete
completed: 2026-08-21
---

# Summary: 01-02 Diagram tooling (Kroki)

## What was built
- `requirements.txt` — `mkdocs-kroki-plugin==0.9.0` pinned, with the rationale in a comment.
- `mkdocs.yml` — `plugins:` block (`search` + `kroki` with `FencePrefix: kroki-` and an
  env-overridable `ServerURL`), plus `not_in_nav` for the authoring reference.
- `learn/diagrams.md` — authoring reference with a working PlantUML demo, the Excalidraw
  embed convention, copyable fence syntax, and self-hosting instructions.

## Acceptance criteria

| AC | Result | Evidence |
|----|--------|----------|
| AC-1 Kroki fences render | ✅ | Built page contains `<img alt="Kroki" src="https://kroki.io/plantuml/svg/eNpVj0EK...">` — encoded diagram, not raw fence |
| AC-2 Search survives plugins block | ✅ | `learn-site/search/search_index.json` present (8.5K) |
| AC-3 Reference stays out of nav | ✅ | Strict build exits 0 with no "not in nav" warning; page absent from navigation |

## Discovery during apply
Kroki 0.9.0 does **not** POST to a server at build time. It deflate+base64-encodes the
diagram source client-side and emits an `<img>` pointing at `kroki.io`. Consequences:

- Builds are fast and work offline (0.20s, no network round-trip). Good.
- **The reader's browser fetches every diagram from kroki.io at page-load.** That is an
  availability and privacy dependency the site inherits on any public deploy.

The plan's stated constraint ("a strict build requires network access") was wrong — the
dependency is at read time, not build time. Corrected here.

## Issues logged
- **ISSUE-01 (deferred):** Diagrams load from the public kroki.io at reader page-load.
  Before any public deploy, either self-host Kroki or inline SVG at build time.
  Effort: M. Revisit: before the first public deploy, not before.

## Files changed
requirements.txt, mkdocs.yml, learn/diagrams.md
