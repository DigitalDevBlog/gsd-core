---
phase: 01-site-foundation
plan: 03
status: complete
completed: 2026-08-21
---

# Summary: 01-03 Versioning with mike

## What was built
- `requirements.txt` — `mike` added (2.2.0 installed).
- `mkdocs.yml` — `extra.version.provider: mike`, with a comment recording that the
  dropdown only renders in a mike-served build.
- `learn/diagrams.md` — "Publishing versions" section with exact deploy commands and an
  explicit warning that nothing has been deployed.

## Acceptance criteria

| AC | Result | Evidence |
|----|--------|----------|
| AC-1 Version provider configured | ⚠️ Partial | Config accepted (`extra.version = {'provider': 'mike'}`) and `mkdocs build --strict` exits 0. But the second clause — "theme emits version-switcher markup" — is **not** met: `grep md-version learn-site/index.html` returns nothing |
| AC-2 mike installed and usable | ✅ | `mike 2.2.0` |
| AC-3 No deploy performed | ✅ | `git branch --list gh-pages` → 0 branches |

## Plan vs actual
AC-1 was written wrong. Material renders the version switcher only when a mike-deployed
`versions.json` is reachable at runtime; a plain `mkdocs build` never emits that markup
regardless of configuration. The testable observable is "config accepted + clean build",
which holds. The switcher itself cannot be verified without a deploy — and a deploy was
deliberately excluded from this plan's scope.

Corrected understanding, carried forward: **versioning is configured, not proven.** It
will be proven the first time the owner runs `mike deploy`.

## Decisions made during apply
| Decision | Rationale |
|----------|-----------|
| Configure mike but perform no deploy | `mike deploy` writes a `gh-pages` branch into the live `@opengsd/gsd-core` package repo (currently on `next`), and `--push` publishes it. Both are the repo owner's call, not an autonomous plan's |

## Issues logged
- **ISSUE-02 (deferred):** Version switcher unverified until a first `mike deploy`.
  Effort: S. Revisit: when the owner decides to publish.

## Files changed
requirements.txt, mkdocs.yml, learn/diagrams.md
