# Prior Art Map — GSD Core's Own `docs/`

> Research input for "Understanding GSD Core" (the `learn/` site).
> Scope: what the repo's existing documentation already covers, where it disagrees with
> actual source, and how a "how it is built / how to rebuild it" site should differentiate.
> **Source wins on every conflict below.** Verified against the checkout on 2026-08-21.

---

## 0. The corpus is much bigger than the five files under review

`docs/` is **400 markdown files** (406 total) organised as a full Diátaxis tree, not a
handful of design notes:

| Tier | Count | Notes |
|---|---|---|
| `docs/tutorials/` | 5 | first project, brownfield onboarding, build/install a capability |
| `docs/how-to/` | 58 | task-shaped, operator-facing |
| `docs/reference/` | 14 | `plan-md.md`, `state-md.md`, `planning-artifacts.md`, `capability-manifest.md`, `workflow-fragments.md`, … |
| `docs/explanation/` | 10 | the 3 under review + security-model, capability-trust-model, capability-overlay-model, embeddable-orchestration-system, live-dom-uat, claude-orchestration, interface-versioning |
| `docs/adr/` | 84 | numbered ADRs, indexed by `scripts/gen-adr-index.cjs` |
| top-level | ~30 | ARCHITECTURE, INVENTORY, COMMANDS, AGENTS, CLI-TOOLS, CONFIGURATION, FEATURES, USER-GUIDE, TESTING-STANDARDS, VERSIONING, … |
| translations | 4 README trees | pt-BR, ja-JP, zh-CN, ko-KR |

**Implication for the new site:** "the repo has no docs" is false, and "the repo has no
*design* docs" is also false — the 84 ADRs are the single richest seam of build rationale
in the tree. The gap is neither coverage nor rationale. It is **assembly**: nothing in
`docs/` walks a reader from zero to a working replica of the pattern. See §4.

---

## 1. What each reviewed document already covers

### `docs/ARCHITECTURE.md` (990 lines)

Self-described as "system architecture for contributors and advanced users." Sections:

- **System Overview** — the 5-layer ASCII stack: USER → COMMAND LAYER (`commands/gsd/*.md`)
  → WORKFLOW LAYER (`gsd-core/workflows/*.md`) → AGENTS (fresh context) → CLI TOOLS LAYER
  (`gsd-tools.cjs`) → FILE SYSTEM (`.planning/`).
- **Design Principles** — 5 named: fresh context per agent, thin orchestrators, file-based
  state, *absent = enabled* config semantics, defense in depth.
- **Component Architecture** — per-surface descriptions of commands, workflows, agents,
  references, templates, hooks; the two-stage namespace-router skill layout (#2792);
  the workflow **byte** size-budget ratchet (XL 90 000 / LARGE 54 000 / DEFAULT 38 000)
  with the honest "extraction only helps if the extracted content is loaded lazily" caveat;
  the Command Routing Hub result taxonomy; capability command dispatch (first-party +
  installed overlay); the Research Module L2-hybrid seam; the `CONTEXT.md` predicate
  fact-store; workflow fragmentization (`gsd:section` markers); the section manifest.
- **Agent Model** — orchestrator→agent pseudocode, spawn-category table, wave execution,
  adaptive context enrichment at `context_window >= 500000`, parallel commit safety,
  the STATE.md write path (`FIELD_CLASSIFICATION` / `applyStatePreservation`, ADR-3408).
- **Data Flow** — new-project flow, phase-execution flow with its gates (research gate,
  package-legitimacy gate, requirements-coverage gate, decision-coverage gate), and a
  context-propagation matrix (which artifact feeds which agent).
- **File System Layout** — install tree per runtime, and the full `.planning/` tree.
- **Installer Architecture** — 9 numbered responsibilities of `bin/install.js`, the
  `install-fs-adapter.cjs` injectable-fs seam, migration module, platform handling.
- **Hook System** — event wiring diagram, context-monitor thresholds (35 % / 25 %,
  5-tool-use debounce), safety properties, package-legitimacy gate layers, security hooks.
- **Runtime Abstraction** — the 15-row Runtime Install Contract Matrix (global root /
  local root / invocation surface / agent surface / config+hooks), upstream contract
  source snapshot dates, and the 5 abstraction points.

**Verdict:** the most mechanism-dense existing document. Strong on *seams and contracts*;
its counts and CLI invocation syntax have drifted (§2). Written as a reference for someone
already inside the codebase — no narrative, no ordering, no "why this and not that."

### `docs/INVENTORY.md` (689 lines)

The authoritative, CI-guarded roster. Explicitly declares itself the tiebreaker:
*"When the filesystem diverges from `docs/ARCHITECTURE.md` counts … this file is the
source of truth."* Six families, each a table anchored by a drift test:

| Family | INVENTORY rows | Actual on disk | Guard |
|---|---|---|---|
| Agents | 35 | 35 (`agents/*.md`) | `tests/inventory-manifest-sync.test.cjs` |
| Commands | 71 | 71 (`commands/gsd/*.md`) | `tests/command-count-sync.test.cjs` |
| Workflows | 89 (incl. sub-files) | 88 top-level `.md` + sub-dirs | inventory-manifest-sync |
| References | — | 104 (`gsd-core/references/*.md`) | inventory-manifest-sync |
| CLI modules | 202 | 194 `src/*.cts` → compiled `bin/lib/*.cjs` | inventory-manifest-sync |
| Hooks | 26 | 26 (`hooks/`) | inventory-manifest-sync |

Machine-readable twin: `docs/INVENTORY-MANIFEST.json`, regenerated by
`scripts/gen-inventory-manifest.cjs --write`, `--check`ed in `lint:generated-sync`.

**Verdict:** accurate and load-bearing. The new site should *cite* it for counts rather
than restate them — restating is how ARCHITECTURE.md drifted.

### `docs/explanation/the-phase-loop.md` (129 lines)

The mental model. Covers: the loop shape `Discuss → (UI design) → Plan → Execute → Verify
→ Ship`; a *why does this step exist* paragraph per step; the milestone/phase distinction;
an unusually good section on **what makes a good phase scope** (too-large vs too-small
failure modes, four criteria, concrete good/bad examples); how `.planning/` carries state
across context resets; and a closing "the loop is a rhythm, not a constraint."

**Verdict:** the best-written document in the set and the strongest *conceptual*
prior art. It is entirely about **why the shape is right** and contains almost nothing
about **how the shape is implemented**. No mention of the `gsd:loop-host` markers, the
generated Loop Host Contract, gates, or how a step is actually dispatched.

### `docs/explanation/context-engineering.md` (133 lines)

The origin story. Covers: context rot as a property of transformer attention (four named
symptoms); fresh-context subagents as a *structural* rather than palliative fix; the
worked `/gsd-plan-phase` example (load payload → researcher → planner → plan-checker);
spec-driven development + meta-prompting as the two disciplines that keep fresh context
from producing vague output; `.planning/` and `STATE.md` as the durable substrate;
**lifecycle hooks and context headroom** (PreCompact/Stop/SubagentStop, Qwen parity,
`FileChanged` config hot-reload, `effort:` frontmatter, the removed `context: fork`);
and two honest trade-off sections (overhead, latency, ceremony; hook maintenance surface,
headroom-as-heuristic, subagent isolation).

**Verdict:** the clearest statement of the *problem* GSD exists to solve. Its runtime
claims are the most drifted of the three (§2).

### `docs/explanation/multi-agent-orchestration.md` (234 lines)

Covers: the problem restated; the orchestrator→agent pseudocode (duplicated verbatim from
ARCHITECTURE.md); an 8-row agent-category table (a subset of INVENTORY's 35); wave-based
parallel execution with the dependency diagram; parallel commit safety; adaptive context
enrichment at 500 K; the connection back to context engineering (context isolation +
context hygiene); and five trade-offs — coordination overhead, opacity during execution,
context-stitching cost, model-cost amplification, and the `dynamic_routing` escalation.

**Verdict:** substantial overlap with ARCHITECTURE.md §Agent Model — the same pseudocode,
the same wave diagram, the same enrichment paragraph, the same commit-safety mechanisms.
Its distinct contribution is the trade-offs section and the "opacity during execution"
observation. It also inherits ARCHITECTURE.md's drift verbatim, including the inverted
`--no-verify` claim.

---

## 2. Where existing docs disagree with source

Ordered by consequence, not by size. **Tier A** would make a reader write something that
does not run or hold a wrong model of the mechanism. **Tier B** is sanctioned staleness
in a curated subset — INVENTORY.md already declares itself the tiebreaker for these.

### Tier A — mechanism-level, load-bearing

**A1. `--no-verify` on executor commits: #2924 inverted the default; the documented
behaviour is now opt-in.**

Both `ARCHITECTURE.md` §Parallel Commit Safety and
`explanation/multi-agent-orchestration.md` §Parallel commit safety state:

> "`--no-verify` commits — Parallel agents skip pre-commit hooks … The orchestrator runs
> `git hook run pre-commit` once after each wave completes."

Source says the opposite. `gsd-core/workflows/execute-phase.md:717` (parallel/worktree
executor prompt):

```
Run `git commit` normally — hooks run by default. Do NOT pass `--no-verify`
unless the orchestrator surfaces `workflow.worktree_skip_hooks=true` in this
prompt; silent bypass violates project CLAUDE.md guidance (#2924).
```

`:796` (sequential mode): *"Use normal git commits (with hooks). Do NOT use --no-verify."*
And `:855` — the post-wave hook run is now **conditional on the opt-out**:

```
5. Post-wave hook validation (parallel mode only): Hooks run on every executor commit
   by default (#2924); this post-wave run only fires when
   workflow.worktree_skip_hooks=true opted out of per-commit hooks
```

The `--no-verify` path still exists, but it is now **opt-in** behind
`workflow.worktree_skip_hooks=true`, and the per-wave `git hook run pre-commit` fires
**only on that opt-out path**. Both docs describe the pre-#2924 behaviour as current.
This is the single most misleading claim in the reviewed set: it is repeated identically
in two documents, presented as a design rationale ("cargo lock fights in Rust projects"),
and a reader rebuilding the pattern would copy an inverted safety default.

**A2. The CLI surface documentation predates the `query` namespace refactor — wholesale.**

Three separate documented invocations, zero hits for any of them in shipped source:

| Documented (ARCHITECTURE.md, multi-agent-orchestration.md) | Actual in `gsd-core/workflows/` | Evidence |
|---|---|---|
| `gsd-tools.cjs init <workflow> <phase>` | `gsd_run query init.<workflow>` | 16× `query init.phase-op`, 7× `query init.plan-phase`, 3× `query init.execute-phase`; **0** hits for `gsd-tools.cjs init ` anywhere in `gsd-core/`, `agents/`, `commands/` |
| `gsd-tools.cjs resolve-model <agent-name>` | `gsd_run query resolve-model <agent> --raw` / `--pick model` / `--pick tier` | `validate-phase.md:31`, `ui-review.md:31`, `secure-phase.md:31`, `code-review-fix.md:28-29`, `explore.md:83-84`, … |
| `gsd-tools.cjs state update / state patch / state advance-plan` | `gsd_run query state.update`, `query state.patch`, `query state.advance-plan` | `state.load` ×7, `state.record-session` ×6, `state.update` ×4, `state.add-roadmap-evolution` ×3, `state.patch` ×2, plus `state.milestone-switch`, `state.begin-phase`, `state.planned-phase`, `state.advance-plan`, `state.add-decision`, `state.add-blocker`, `state.record-metric`, `state.get`, `state.json` |

Two things changed and neither is documented in the reviewed set:

1. **The verb namespace.** Everything moved under a `query` verb with a dotted
   `family.operation` key: `query roadmap.get-phase` (15×), `query roadmap.analyze` (10×),
   `query verification.status` (9×), `query worktree.base-check` (7×), `query phase.complete`
   (7×), `query git.base-branch` (5×), `query frontmatter.set` (5×) …
2. **The entrypoint.** Workflows never call `node gsd-tools.cjs`. They call **`gsd_run`**, a
   POSIX `sh` shim at `gsd-core/bin/gsd_run` that resolves its own real path through
   symlinks and `exec node "$dir/gsd-tools.cjs" "$@"`. Its header documents *why* it exists:

   > "Claude Code (and similar runtimes) runs each fenced bash block in a separate process,
   > so an inline `gsd_run()` function defined in an earlier block is undefined in later
   > ones (issue #381)."

   It reaches `PATH` via the npm `bin` field and a per-file preamble `CLAUDE_ENV_FILE`
   export. **This is a genuinely transferable design lesson** — "your agent's shell blocks
   are not one shell" — and it appears in *no* reviewed document.

**A3. `effort:` frontmatter is deliberately NOT emitted into skills.**

`explanation/context-engineering.md` §"Effort signals for heavy and light skills":

> "GSD uses `effort:` frontmatter to signal the token budget appropriate for each skill.
> Heavy orchestrator **skills** (`plan-phase`, `execute-phase`, `autonomous`) declare
> `effort: max`; quick-status **skills** (`progress`, `stats`) declare `effort: low`."

Source: **zero** of the 71 `skills/*/SKILL.md` files carry an `effort:` key. It exists only
in `commands/gsd/*.md` (3× `effort: max`, 3× `effort: low`). The omission is deliberate —
`src/runtime-artifact-conversion.cts:468`:

> "(#3151: `effort:` is no longer emitted into skill frontmatter — a static effort value
> changes `output_config.effort` on invocation and invalidates the caller's prompt
> [cache] … the mechanism. The separate agent-effort surface is tracked by #3160.)"

Restated at `:513`. The claim is true of the *command* surface and false of the *skill*
surface, and the doc never distinguishes them — which matters precisely because
§3 shows skills are generated from commands.

Minor rider: the `effort: low` set is `next.md`, `progress.md`, `stats.md`. The doc names
only `progress` and `stats`.

**A4 — RECLASSIFIED, see §4.2 Move 5.** The "narrated loop vs contracted loop" distinction
is *not* a doc error — a reader following `the-phase-loop.md` writes nothing wrong. It has
been moved out of the conflicts list and into the differentiation moves, where it belongs.
The evidence is retained below for reference.

<details>

**The canonical loop has five machine-declared steps, and UI is not one of them.**

`the-phase-loop.md` presents the loop as `Discuss → (UI design) → Plan → Execute → Verify
→ Ship`, with UI design as an optional step *inside* the loop.

The generated **Loop Host Contract** (`gsd-core/bin/lib/loop-host-contract.cjs`, ADR-894 §3)
declares exactly **12 points across 5 steps**:

```
discuss:pre  discuss:post
plan:pre     plan:post
execute:pre  execute:wave:pre  execute:wave:post  execute:post
verify:pre   verify:post
ship:pre     ship:post
```

It is generated by `scripts/gen-loop-host-contract.cjs` from `<!-- gsd:loop-host -->`
markers that appear in exactly five workflow files: `discuss-phase.md`, `plan-phase.md`,
`execute-phase.md`, `verify-work.md`, `ship.md`. **`ui-phase.md` carries no marker.**

This is not a contradiction so much as a missing distinction the new site should draw:
there is a *narrated* loop (what a user experiences, 6 steps) and a *contracted* loop
(what capabilities can hook into, 5 steps / 12 points, machine-generated, CI-guarded).
`ui-phase` is a side workflow that produces an artifact the loop consumes, not a loop
stage. No existing document names the Loop Host Contract at all — including
ARCHITECTURE.md, which mentions `loop-host-contract.cjs` only as a one-line row in its
CLI-module table.

</details>

**A5. ARCHITECTURE.md's CLI-module table presents build artifacts as if hand-authored.**

The §CLI Tools table lists ~40 `gsd-core/bin/lib/*.cjs` modules with per-module
responsibilities. Two problems:

- **The roster is ~5× short.** INVENTORY.md lists **202** CLI modules.
- **Almost none of those files are in the repository.** `gsd-core/bin/lib/` contains only
  **8** committed `.cjs` files (`capability-command-router`, `capability-registry`,
  `capability-validator`, `legacy-cleanup`, `loop-host-contract`, `package-identity`,
  `profile-pipeline-command-router`, `stale-bake-guard`) plus `vendor/`. The rest are
  **compiled from 194 `src/*.cts` sources** by `npm run build:lib` (ADR-457) and are
  explicitly gitignored (`.gitignore:70-89+`).

ARCHITECTURE.md does say "generated to `gsd-core/bin/lib/*.cjs` per ADR-457" in three
scattered prose paragraphs, but its own module table — the part a reader treats as the map
of the runtime — gives no hint that the directory it describes is a build output.

### Tier B — counts, already sanctioned as stale

INVENTORY.md's Maintenance section pre-authorises these ("Broad docs may render narrative
or curated subsets"). Report them; do not rank them with Tier A.

| Claim | ARCHITECTURE.md | Actual |
|---|---|---|
| Agent count | "**Total agents: 33**" | **35** `agents/*.md` (INVENTORY agrees: 35) |
| Hook roster | 12-row table | **26** hooks (INVENTORY agrees: 26) — the table omits the entire Cursor family (6), the Windsurf pair, `gsd-write-guard`, `gsd-worktree-path-guard`, `gsd-agent-isolation-guard`, `gsd-config-reload`, `gsd-update-banner`, `gsd-graphify-update.sh` |
| CLI modules | ~40-row table | **202** (INVENTORY) / 194 `src/*.cts` |
| Workflows | "88 of the 89 shipped workflows" | 88 top-level `.md` + sub-directories; INVENTORY 89 rows incl. sub-files — consistent |
| Skill layout | "≈67 concrete skills", "~61 concrete" | 71 `skills/*/` dirs, 71 `commands/gsd/*.md`, 6 of which are `ns-*` routers → 65 concrete |
| Capabilities | not counted in ARCHITECTURE | **46** `capabilities/*/` |

### Tier C — internal cross-references worth knowing about

- `the-phase-loop.md` links `../reference/planning-artifacts.md`, `../reference/state-md.md`,
  `../how-to/discuss-a-phase.md`; ARCHITECTURE.md links `reference/workflow-fragments.md`.
  **All of these resolve** — `docs/reference/` (14 files) and `docs/how-to/` (58 files)
  both exist. No dangling-tier problem.
- `multi-agent-orchestration.md` duplicates ARCHITECTURE.md's orchestrator pseudocode,
  wave diagram, enrichment paragraph and commit-safety list nearly verbatim. When
  ARCHITECTURE drifted (A1, A2), the duplicate drifted with it. **Duplication is the
  drift vector** — a lesson the new site should both state and obey.

---

## 3. The thing no existing document describes: the generation pipeline

This is the highest-value finding. Every reviewed document describes GSD's surfaces as
though a human wrote each one. In fact **most shipped surfaces are generated from a
smaller set of authored sources, and the generation is guarded by a ring of `--check`
linters in CI.**

### 3.1 Authored → generated

| Authored source | Generator | Emitted surface |
|---|---|---|
| `commands/gsd/*.md` (71) | `scripts/gen-plugin-skills.cjs --write` | `skills/gsd-<stem>/SKILL.md` (71) |
| `src/*.cts` (194) | `npm run build:lib` (ADR-457) | `gsd-core/bin/lib/*.cjs` (gitignored) |
| `<!-- gsd:loop-host -->` markers in 5 workflows | `scripts/gen-loop-host-contract.cjs` | `bin/lib/loop-host-contract.cjs` (12 points) |
| `capabilities/*/` co-located declarations (46) | `scripts/gen-capability-registry.cjs` | `bin/lib/capability-registry.cjs` |
| `<!-- gsd:section -->` markers in workflows | `scripts/gen-section-manifest.cjs` | `gsd-core/workflows/section-manifest.json` |
| backtick `CLASS.subkey=value` predicates in root `CONTEXT.md` | `scripts/gen-context-index.cjs` | `docs/CONTEXT-INDEX.json` |
| the filesystem itself | `scripts/gen-inventory-manifest.cjs` | `docs/INVENTORY-MANIFEST.json` → `docs/INVENTORY.md` |
| ADR files | `scripts/gen-adr-index.cjs` | ADR index |
| capability declarations | `scripts/gen-capability-matrix.cjs` | capability matrix |
| health rules | `scripts/gen-health-docs.cjs` | health docs |

`package.json:98` — the whole chain in order:

```
"build": "generate:identity && build:lib && gen:section-manifest && gen:context-index
          && gen:plugin-skills && gen:loop-host-contract && gen:capability-registry
          && build:hooks"
```

### 3.2 The drift-guard ring

`lint:generated-sync` (`package.json:130`) runs **15 `--check` passes** — capability
registry, loop host contract, capability matrix, manifest versions, inventory manifest,
package identity, plugin skills, registry, ADR index, glossary refs, compiled-artifact
sync, context index, section manifest, health docs. It is wired into `lint:ci`, so into
CI. `lint:ci` additionally runs **~30 bespoke `lint-*-drift.cjs` scripts** — plan-count
drift, milestone-window drift, phase-enumeration drift, planning-prompt drift,
unreachable-guard drift, completion-ratio drift, state-field drift, state-write-path
drift, completion-predicate drift, table-schema drift, contract drift, and more.

**This is the actual architecture of GSD Core as an engineering artifact**: author one
surface, generate the rest, and make CI fail the moment a generated artifact stops
matching its source. It is why INVENTORY.md is accurate and ARCHITECTURE.md is not —
INVENTORY has a generator and a `--check`; ARCHITECTURE is hand-maintained prose.

### 3.3 Why `skills/` being generated matters for the reader

`commands/gsd/plan-phase.md` and `skills/gsd-plan-phase/SKILL.md` are near-identical, but:

| | command | generated skill |
|---|---|---|
| `name:` | `gsd:plan-phase` (colon) | `gsd-plan-phase` (hyphen) |
| `effort:` | `max` | **absent by design** (#3151) |
| `requires:` | `[discuss-phase, phase, review, update]` | absent |
| body | identical | `/gsd:review` → `/gsd-review` rewritten |

The converter is `convertClaudeCommandToClaudeSkill` in
`src/runtime-artifact-conversion.cts:452` — **the same function the installer uses** to
materialise skills for each of the 16 runtimes. So `gen-plugin-skills.cjs` is not a
parallel code path; it is the installer's own converter run at build time to produce a
plugin-shaped npm payload. That single design choice — *one converter, used at build time
and install time* — is exactly the kind of decision the new site exists to explain and
that no existing doc mentions.

### 3.4 Both dispatchers are thin; the workflow is the program

`skills/gsd-plan-phase/SKILL.md` is 4 861 bytes. Its entire `<process>` block reads:

```
Execute end-to-end.
Preserve all workflow gates (validation, research, planning, verification loop, routing).
```

The substance is `@~/.claude/gsd-core/workflows/plan-phase.md` — 1 600+ lines. Every
behavioural claim in the explanation docs is a claim about *workflow* content, and the
workflows are where the interesting engineering lives:

- `plan-phase.md:1130-1200` — the revision loop. **`max 3 iterations` is confirmed**, but
  the docs stop there. Source additionally has *stall detection* (issue count not
  decreasing between iterations), an interactive Proceed / Adjust-approach branch, a
  `stall_reentry_count` capped at 2 that persists across re-entries while
  `iteration_count` resets, and a terminal `## PLANNING INCONCLUSIVE` state.
- `execute-phase.md:144` — the enrichment threshold is described from the *other* side:
  "When `CONTEXT_WINDOW < 200000` (sub-200K models), subagent prompts are **thinned**."
  The docs describe only the ≥500 K enrichment; source has a three-band policy.
- `execute-phase.md:1510-1511` — "Orchestrator: ~10-15 % context for 200k windows, can use
  more for 1M+ windows. Subagents: fresh context each (200k-1M depending on model)."
  The command file states the budget as a first-class contract: *"Context budget: ~15 %
  orchestrator, 100 % fresh per subagent."*
- `execute-phase.md` decomposes into 11 step files under `execute-phase/steps/` —
  `codebase-drift-gate`, `executor-isolation-dispatch`, `gap-closure-artifacts`,
  `partial-wave`, `per-plan-executor-routing`, `per-plan-worktree-gate`, `post-merge-gate`,
  `regression-gate` / `regression-gate-run`, `wave-post-gate-hooks`,
  `worktree-recovery-policy` — each lazily `Read` only when its `gsd:section` `when=`
  predicate holds. That is progressive disclosure implemented as a data structure, and
  ARCHITECTURE.md describes the *mechanism* (section manifest) without ever showing the
  *result*.

---

## 4. How the new site should differentiate

`.paul/PROJECT.md` already fixes the thesis: *content = how it is built, not how it is
used*, with build-your-own as the spine. This research sharpens that from a stylistic
choice into a structural argument.

### 4.1 The one-sentence differentiator

> The repo's `docs/` documents **surfaces**. The new site documents the **pipeline that
> produces the surfaces** — and the drift guards that keep it honest.

`docs/` is a 400-file reference tree written for someone operating or contributing to GSD
Core. It answers *what exists* (INVENTORY), *how to use it* (58 how-tos), *why the shape*
(10 explanations), and *why this decision* (84 ADRs). The one question it never answers is
**how would I build this from nothing** — because that question is uninteresting to
everyone the existing docs are written for.

### 4.2 Five concrete differentiation moves

**Move 1 — Lead with the generation pipeline, not the layer cake.**
Every existing document opens with the USER → COMMAND → WORKFLOW → AGENT → CLI → FILESYSTEM
stack. Repeating it adds nothing. Open instead with the *build* axis (§3): 71 commands and
194 `.cts` sources are authored; 71 skills, ~200 `.cjs` modules, 4 registries and 2 JSON
indexes are generated; 45 `--check` passes make drift a CI failure. That reframes the
"Runtime gets its own section" decision in PROJECT.md from a size argument (199 modules is
a lot) into a mechanism argument (the runtime is a *compiler target*, and that is the
structural departure from a pure prompt framework).

**Move 2 — Make "authored vs generated" a visible property of every surface.**
For each stratum the site covers, state which side of the line it sits on and what guards
it. A reader rebuilding the pattern needs to know that `skills/` is a build artifact far
more than they need its 71-row roster. This is also the site's own drift defence: never
restate a count INVENTORY.md owns — cite it and link.

**Move 3 — Descend to the workflow layer, where the docs stop.**
The explanation docs describe orchestration; the *orchestrator* is the 1 600-line
`plan-phase.md`, and nobody has written about it. The richest untouched material:
- the revision loop as a real state machine (`iteration_count`, `prev_issue_count`,
  `stall_reentry_count`, PLANNING INCONCLUSIVE) — not "max 3x"
- the gate vocabulary as a first-class construct (research gate, package-legitimacy gate,
  requirements-coverage gate, blocking vs non-blocking decision-coverage gates, codebase-
  drift gate, schema-drift gate, regression gate, post-merge gate) and the design question
  behind each: *what does this gate fail-open or fail-closed on, and why?*
- progressive disclosure as data (`gsd:section` markers → section manifest → lazy `Read`),
  including ARCHITECTURE's own honest caveat that `@`-import behind a conditional still
  loads eagerly (#720) — a genuinely transferable trap
- the byte-budget ratchet as a *quality* control rather than a cost control, and why bytes
  beat lines as the unit

**Move 4 — Name the narrated loop and the contracted loop as two different things.**
(Formerly filed as conflict A4; it is a differentiation opportunity, not a doc error.)
`the-phase-loop.md` narrates 6 steps with UI design inside the loop. The machine-generated
**Loop Host Contract** declares 5 steps / 12 points and has no `ui:` point — `ui-phase.md`
carries no `<!-- gsd:loop-host -->` marker. Both are correct about different things: one
describes what a user experiences, the other enumerates where a capability may attach.
The distinction is invisible from inside either document, it is exactly the kind of thing
a reconstruction guide must get right (you cannot rebuild an extension point you do not
know exists), and **no document in `docs/` names the Loop Host Contract at all** —
ARCHITECTURE.md mentions `loop-host-contract.cjs` only as a one-line row in a table.

**Move 5 — Ship a corrections section, and treat it as a feature.**
Publish the Tier A findings (§2) as a short "where the upstream docs have drifted" page
with source line citations. It is the sharpest possible demonstration of the site's
central success metric (*every explanation traceable to real GSD Core source*), it gives
the site a reason to exist that no amount of re-narration would, and it produces upstream
value — the `--no-verify` inversion (A1) and the `effort:`-on-skills claim (A3) are both
worth an issue against `open-gsd/gsd-core`.

### 4.3 What to deliberately NOT re-do

- **Do not restate rosters.** INVENTORY.md is generated and CI-guarded. Link it.
- **Do not re-argue context rot.** `context-engineering.md` does it well in 133 lines.
  Cite it, then go to the mechanism it does not cover.
- **Do not re-derive phase-scoping heuristics.** `the-phase-loop.md`'s "what makes a good
  phase scope" is the best prose in the corpus and is pure usage guidance — explicitly out
  of scope per PROJECT.md.
- **Do not duplicate.** ARCHITECTURE.md and multi-agent-orchestration.md share four
  passages verbatim, and every one of them drifted together. If a fact belongs in two
  pages, it belongs in one page and a link.

### 4.4 Mapping the moves onto the existing `learn/` skeleton

Phase 1 shipped 8 section stubs plus `index.md` and `diagrams.md`, with `mkdocs.yml` nav
already wired. Three of the five moves land cleanly on existing stubs; two do not have a
home yet.

| Move | Lands in | Action needed |
|---|---|---|
| 1 — lead with the generation pipeline | **`learn/runtime/`** and `learn/index.md` | Runtime's stub bullets are "the `src/*.cts` module map" and "the code-versus-prompt boundary" — both *static* framings. Add the build axis: authored→generated table, the `build` chain, the `lint:generated-sync` ring. Also promote the pipeline into `learn/index.md`'s framing, replacing the layer-cake opener. |
| 2 — authored vs generated as a visible property | **every section** | Add a standing convention: each section states which side of the line its surface sits on and what guards it. Most urgent in `learn/skills/` (skills are 100 % generated). |
| 3 — descend to the workflow layer | **no home — gap** | The skeleton has `loop/`, `skills/`, `agents/`, `runtime/` but **no workflow section**. Workflows (88 files, 104 references, the 1 600-line `plan-phase.md`) are where the orchestration actually lives, and they currently fall between the `skills/` stub (dispatchers) and the `agents/` stub (subagents). Either add a `learn/workflows/` section or explicitly scope it into `learn/loop/`. |
| 4 — narrated vs contracted loop | **`learn/loop/`** | The stub's own framing is `new-project → plan-phase → execute-phase → verify → milestone` — a *command* sequence, not the contracted `discuss/plan/execute/verify/ship`. Fix the stub, then make the two-loop distinction a section. |
| 5 — corrections page | **no home — gap** | Needs a new top-level nav entry (e.g. `learn/corrections/` or an appendix under `build-your-own/`). Nothing in the current 8 sections accommodates it. |

**Additionally: `learn/index.md` has already inherited the drift this site exists to
correct.** Its "What GSD Core is made of" table copies numbers from the upstream docs
rather than from the filesystem:

| `learn/index.md` says | Actual | Source |
|---|---|---|
| "~199 `.cts` modules" | **194** | `src/*.cts` |
| "29 hooks" | **26** | `hooks/` (INVENTORY.md agrees: 26) |
| "71 skill definitions behind a single `/gsd` command" | 71 commands → 71 generated skills, of which **6 are `ns-*` namespace routers and 65 concrete**; there is **no single `/gsd` command** | `commands/gsd/`, `commands/gsd/ns-*.md` |
| "35 subagent definitions" | **35** ✓ | `agents/*.md` |
| "46 capability packs" | **46** ✓ | `capabilities/*/` |
| "v1.11.0" | **1.11.0** ✓ | `package.json` |

`learn/skills/index.md` repeats the error ("how one command routes into 71 of them",
"The `/gsd` router and how dispatch works"). The real mechanism is the two-stage namespace
routing of #2792 — six `ns-*` routers above ~65 concrete skills, nested only on runtimes
with non-recursive skill loaders and flat everywhere else. Fixing these two stubs should
be the first content task, and it is a live demonstration of Move 5's premise.

### 4.5 The transferable lesson, stated once

If the site has a single thesis a reader carries away, it should be the one the repo
demonstrates but never states:

> **Author one surface. Generate the rest. Make CI fail when a generated artifact stops
> matching its source.**

GSD Core proves both directions of it in the same tree. `INVENTORY.md` has a generator and
a `--check` and is accurate to the file. `ARCHITECTURE.md` is hand-maintained prose and is
wrong about its agent count, its hook roster, its module roster, its CLI syntax, and its
commit-hook default. That contrast is the most instructive thing in the repository, and it
is invisible from inside either document.

---

## 5. Evidence index

| Claim | File:line |
|---|---|
| skills generated from commands | `scripts/gen-plugin-skills.cjs:1-20`; `package.json:104` |
| converter shared with installer | `src/runtime-artifact-conversion.cts:452` |
| `effort:` deliberately not emitted | `src/runtime-artifact-conversion.cts:468`, `:513` |
| `--no-verify` default flipped (#2924) | `gsd-core/workflows/execute-phase.md:717`, `:796`, `:855-866` |
| `gsd_run` shim rationale (#381) | `gsd-core/bin/gsd_run:1-20` |
| `query init.*` usage | `gsd-core/workflows/` — 16× `init.phase-op`, 7× `init.plan-phase`, 3× `init.execute-phase` |
| `query resolve-model` usage | `workflows/validate-phase.md:31`, `ui-review.md:31`, `secure-phase.md:31`, `code-review-fix.md:28` |
| `query state.*` usage | `workflows/` — `state.load` ×7, `state.update` ×4, `state.patch` ×2, +11 more verbs |
| Loop Host Contract 12 points / 5 steps | `gsd-core/bin/lib/loop-host-contract.cjs:1-10` |
| `gsd:loop-host` markers | `workflows/{discuss-phase,plan-phase,execute-phase,verify-work,ship}.md:1` |
| revision loop state machine | `gsd-core/workflows/plan-phase.md:1130-1200` |
| sub-200K thinning band | `gsd-core/workflows/execute-phase.md:144` |
| orchestrator context budget | `gsd-core/workflows/execute-phase.md:1510-1511`; `commands/gsd/execute-phase.md` objective |
| `bin/lib` is a build output | `.gitignore:70-89+`; 8 committed `.cjs` vs 194 `src/*.cts` |
| build chain order | `package.json:98` |
| 15-check generated-sync ring | `package.json:130` |
| INVENTORY self-declared authoritative | `docs/INVENTORY.md` §Maintenance |
| agent count 35 vs 33 | `agents/*.md` = 35; `docs/ARCHITECTURE.md:215` = "Total agents: 33" |
| hook count 26 vs 12 | `hooks/` = 26; `docs/ARCHITECTURE.md:287-301` = 12 rows |
| CLI modules 202 vs ~40 | `docs/INVENTORY.md` = 202 rows; `docs/ARCHITECTURE.md:422-463` ≈ 40 rows |

---
*Written by the prior-art mapping pass, 2026-08-21. Source-verified against the `next` branch at `dacae927`.*
