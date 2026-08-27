# Skill catalogue

Seventy-one markdown files live in `commands/gsd/`, and seventy-one directories mirror
them in `skills/`. Any framework that reaches this size faces the same question, and it
is not "what does each one do?" — a reader can open the files. It is: **what stops
seventy-one from becoming sprawl?**

GSD Core's answer is more interesting than a single taxonomy. It partitions the same 71
skills **three separate times, along three orthogonal axes**, and each partition is owned
by a different mechanism, serves a different consumer, and is enforced — or not enforced —
independently. That is the transferable lesson on this page. The lists below are evidence
for it.

!!! quote "The organising principle, in one sentence"
    Do not classify skills once and hope the classification holds. Classify them **once per
    consumer**, and give each classification a mechanism that breaks when it drifts — because
    the partition without an enforcing consumer is the one that has already drifted.

## The three partitions

| Axis | What it groups by | Where it is declared | Who consumes it | Enforcement |
|---|---|---|---|---|
| **Loop position** | Which loop step a skill *is*, or which artifact it reads/writes | `gsd:loop-host` markers in 5 workflow files | The loop-host contract and its extension points | Generated + byte-compared in CI (`scripts/gen-loop-host-contract.cjs`) |
| **Intent domain** | What the user is trying to do | `requires:` + a prose table in six `ns-*` router files | Runtime dispatch; nested install layout on 5 runtimes | Partial — target existence tested, coverage is not |
| **Install tier** | Whether a skill ships at all | `PROFILES` in `src/install-profiles.cts` | The installer's staging pass | Machine-enforced (`scripts/lint-skill-deps.cjs`) |

Two of these are generated and gated. One is prose. Watch which one has already drifted.

---

## Axis 1 — Loop position

### Position is declared one layer *below* the skill

This is the first surprise, and it reshapes what "group by loop position" can even mean.
**Zero of the 71 command files declare a loop position.** Nothing in a skill's frontmatter
says "I am the plan step" or "I run after execute."

Position is declared in the *workflow* layer, in exactly five of the 88 files in
`gsd-core/workflows/`, as a structured HTML comment at the top of the file. Here is the
complete declaration from `gsd-core/workflows/plan-phase.md`:

```markdown
<!-- gsd:loop-host
step: plan
points: plan:pre, plan:post
agent-roles: researcher, planner, checker
produces: PLAN.md
consumes: CONTEXT.md
-->
```

So the loop-position axis is **deliberately sparse**: five skills *are* the loop, and the
other 66 orbit it.

| Step | Marker lives in | Skill shell | Consumes | Produces | Agent roles |
|---|---|---|---|---|---|
| discuss | `gsd-core/workflows/discuss-phase.md` | `commands/gsd/discuss-phase.md` | — | `CONTEXT.md` | orchestrator |
| plan | `gsd-core/workflows/plan-phase.md` | `commands/gsd/plan-phase.md` | `CONTEXT.md` | `PLAN.md` | researcher, planner, checker |
| execute | `gsd-core/workflows/execute-phase.md` | `commands/gsd/execute-phase.md` | `PLAN.md` | `SUMMARY.md` | executor, verifier |
| verify | `gsd-core/workflows/verify-work.md` | `commands/gsd/verify-work.md` | `SUMMARY.md` | `UAT.md` | orchestrator |
| ship | `gsd-core/workflows/ship.md` | `commands/gsd/ship.md` | `UAT.md` | — | orchestrator |

The two columns are the point: the marker is never in the file on the left of the skill
layer. A reader looking only at `commands/gsd/` cannot reconstruct this table.

### The positioning rule for the other 66

The `produces:` / `consumes:` fields are not decoration — they form a clean artifact
dependency chain, `CONTEXT.md → PLAN.md → SUMMARY.md → UAT.md`. That chain gives you the
rule for placing everything else:

!!! tip "A satellite skill's loop position is the artifact it reads or writes"
    You cannot read it off the frontmatter, but you can read it off the description, because
    the descriptions are unusually disciplined about saying so.

The evidence is in the `description:` fields themselves. `commands/gsd/spec-phase.md` says
its `SPEC.md` comes "before discuss-phase." `commands/gsd/mempalace-recall.md` says
"before planning." `commands/gsd/audit-milestone.md` says "before archiving."
`commands/gsd/add-tests.md` says "for a completed phase based on UAT criteria" — which
places it after verify, not after execute.

| Position | Skills |
|---|---|
| **Pre-discuss** | `spec-phase` (ambiguity scoring → `SPEC.md`), `explore`, `sketch`, `spike`, `capture` |
| **Pre-plan** | `mempalace-recall`, `map-codebase`, `graphify`, `ui-phase` (`UI-SPEC.md`), `ai-integration-phase` (`AI-SPEC.md`), `mvp-phase` (SPIDR-splits, then calls plan) |
| **Post-plan** | `review` (cross-AI peer review of plans), `plan-review-convergence` (replan until concerns resolve), `ultraplan-phase` |
| **Post-execute** | `code-review`, `secure-phase`, `validate-phase`, `ui-review`, `eval-review`, `undo` |
| **Post-verify** | `add-tests`, `audit-uat` |
| **Pre-ship** | `pr-branch` — "ready for code review", so it stages the PR that `ship` then opens |
| **Post-ship** | `extract-learnings`, `mempalace-capture`, `docs-update` |
| **Milestone boundary** | `new-milestone`, `review-backlog` (promotes backlog items into the active milestone), `audit-milestone`, `complete-milestone`, `milestone-summary`, `cleanup` |
| **Loop-bypassing** | `quick` ("skip optional agents"), `fast` ("no subagents, no planning overhead") — these substitute for the loop |
| **Loop-driving** | `autonomous` — "Run all remaining phases autonomously — discuss→plan→execute per phase" |
| **Off-loop** | Everything in the meta/management group below: config, workspace, thread, help, stats, and the six routers |

!!! note "These positions are derived, not declared"
    Only some rows are self-declared — the ones whose own `description:` names a position
    (`spec-phase` "before discuss-phase", `mempalace-recall` "before planning",
    `audit-milestone` "before archiving", `add-tests` "based on UAT criteria", and the four
    `retroactiv*` hits below). The rest — `map-codebase`, `graphify`, `undo`, `pr-branch` —
    are placed by artifact dependency and are inference, not repo fact. That gap is itself the
    finding: there is no field to check them against.

`autonomous` deserves its own row because it is the one skill that iterates the entire
five-step contract rather than occupying a position on it — which is exactly why it is one
of only three commands carrying `effort: max`, alongside `plan-phase` and `execute-phase`.

### The retroactive-gate cluster is a confession

Four commands describe themselves as *retroactive* — the word appears in
`commands/gsd/secure-phase.md`, `commands/gsd/validate-phase.md`,
`commands/gsd/ui-review.md`, and `commands/gsd/eval-review.md`. Their descriptions are
almost formulaic: "Retroactively verify threat mitigations for a completed phase,"
"Retroactively audit and fill Nyquist validation gaps for a completed phase," "Retroactive
6-pillar visual audit of implemented frontend code." Add `commands/gsd/audit-uat.md`
("Cross-phase audit of all outstanding UAT and verification items") and
`commands/gsd/add-tests.md`, and you have six catch-up gates.

!!! warning "The design tension worth naming"
    Every optional in-loop gate has a retroactive twin. That is the framework admitting, in its
    own catalogue, that its gates get skipped — and choosing to ship an after-the-fact
    equivalent for each rather than making the gate mandatory. It is a real architectural
    choice with a real cost: six extra skills exist because the loop is advisory.

    If you are building your own framework, this is the fork. Mandatory gates mean fewer
    skills and more friction. Advisory gates mean less friction and a second catalogue of
    remediation skills that must be kept in sync with the first. GSD Core took the second
    road, visibly and consistently.

---

## Axis 2 — Intent domain: the six `ns-*` routers

Six files partition the *other 65*. They share a shape that appears nowhere else in the
71: `allowed-tools: [Read, Skill]` and nothing more, a `description:` in keyword-tag form
rather than prose, no XML tags, no `@` includes, a `requires:` list naming exactly their
routing targets, a two-column `| User wants | Invoke |` table, and the closing line
`Invoke the matched skill directly using the Skill tool.`

| Router file | Declared `name:` | Targets | Exists to | Representative skills |
|---|---|---|---|---|
| `commands/gsd/ns-workflow.md` | `gsd-workflow` | 16 | Drive the phase pipeline end to end | `discuss-phase`, `plan-phase`, `execute-phase`, `verify-work`, `phase`, `progress` |
| `commands/gsd/ns-manage.md` | `gsd-manage` | 18 | Manage the installation and the session, not the work | `config`, `workspace`, `workstreams`, `thread`, `update`, `surface`, `help` |
| `commands/gsd/ns-review.md` | `gsd-quality` | 11 | Gate quality — review, audit, debug, secure | `code-review`, `secure-phase`, `debug`, `forensics`, `audit-fix`, `ui-review` |
| `commands/gsd/ns-project.md` | `gsd-project` | 10 | Handle the project and milestone lifecycle | `new-project`, `onboard`, `new-milestone`, `complete-milestone`, `import`, `ingest-docs` |
| `commands/gsd/ns-context.md` | `gsd-context` | 6 | Acquire and persist codebase intelligence | `map-codebase`, `graphify`, `docs-update`, `extract-learnings`, `mempalace-recall` |
| `commands/gsd/ns-ideate.md` | `gsd-ideate` | 5 | Explore before committing to a plan | `explore`, `sketch`, `spike`, `spec-phase`, `capture` |

Note the `name:` column. All six routers declare hyphen-form names that drop the `ns-`
prefix, and `ns-review.md` declares `gsd-quality` — matching neither its filename nor its
basename. Two mechanisms make this inert on Claude: the canonical roster is built by
reading filenames in `src/command-roster.cts`, and Claude skill generation forces `name:`
to the directory name in `src/runtime-artifact-conversion.cts`. A non-Claude converter path
in the same file does read `name:` with a fallback, so the field may not be inert on every
runtime.

### The cover is not a partition, and nothing checks that it is

The six `requires:` lists sum to **66 entries but only 65 unique targets**. `spec-phase`
appears in both `ns-ideate` and `ns-workflow` — legitimately, since specifying a phase is
both an ideation act and a pipeline step.

The repo knows. `tests/helpers/nested-layout.cjs` builds the child→router map and comments:
"A few skills are intentionally multi-owner (e.g. `spec-phase` is required by both
`ns-ideate` and `ns-workflow`). `ROUTER_STEMS` is sorted, so the alphabetically-last owner
wins." An **alphabetical tie-break** decides which router physically owns a skill on disk.

What is tested is the *forward* direction only. `tests/skill-manifest.test.cjs` asserts each
router's `name:`, that its description is ≤ 60 chars and contains a pipe, that
`allowed-tools` includes `Skill`, that the body carries a `| User wants` table, and that
every route target resolves to a surviving command or a known consolidated parent. The
*reverse* direction — that all 65 non-router commands are covered — is asserted nowhere.

!!! warning "The prose partition is the one that drifted"
    Two of the three axes are generated and byte-compared. The third is six markdown tables,
    and it is off by exactly one entry with a hand-written alphabetical tie-break papering
    over it. This is not a bug — `spec-phase` genuinely belongs to both. It is a demonstration:
    **a taxonomy with no enforcing consumer is decoration, and decoration drifts.**

### This taxonomy was supposed to be the install layout — and got reverted

The most instructive fact about the routers is that they were once **structural**. Under
issue #69, the installer nests each concrete skill inside its owning router's directory, so
the top level of the skills folder shows six entries instead of seventy-one. The machinery
is still there: `buildNamespaceBundleMap()` in `src/install-profiles.cts` derives the map
from the routers' own `requires:` frontmatter, and `transformRouterBodyToNested()` rewrites
the routing table to match.

Then it was reverted on the primary runtime. `tests/install-nested-layout.test.cjs` records
why in its decision matrix: "Claude reverted to flat (#924): Claude Code scans only one
level under `~/.claude/skills/` so nested concretes were never discoverable by the Skill
tool." `tests/runtime-artifact-layout.test.cjs` now asserts the opposite of the original
design — "full profile should have >= 60 top-level skill dirs (flat layout, #924)" — with
the parenthetical "(Previously nested: exactly 6 `gsd-ns-*` router dirs. That broke
Skill-tool discovery.)"

Five runtimes still nest (cline, qwen, hermes, augment, trae); eight are flat (claude,
cursor, codex, copilot, codebuddy, opencode, kilo, antigravity).

!!! note "What this costs you to know"
    The same taxonomy is **load-bearing structure on five runtimes and inert prose on eight**.
    If you build a taxonomy that doubles as your physical layout, you have coupled your
    information architecture to a host's discovery algorithm — and hosts change theirs. GSD
    Core's recovery was to keep the derivation and make the layout a per-runtime decision,
    which is the right shape: the map survives, only its projection varies.

---

## Axis 3 — Install tier: what actually ships

The third partition ignores both of the others. `PROFILES` in `src/install-profiles.cts`
names three tiers, and its doc comment states the rule: "The effective set for any profile
is CLOSURE(base, `requires:` manifest). `standard` is a superset of `core`; `full` is the
identity (all skills)."

The base sets are small and hand-picked. `core` lists eight stems; `standard` lists fifteen,
annotated in-source as "Core loop", "Phase management (hot nodes from audit — required by
38+ skills)", and "Workspace / state". `full` is the `'*'` sentinel.

Running the shipped algorithm — `computeClosure()` over `parseRequires()`, which is
`requires:`-only, with body scanning reserved for deriving *agents* — gives the real
answer:

| Profile | Base stems | Closure | Share of 71 |
|---|---|---|---|
| `core` | 8 | **15** | 21% |
| `standard` | 15 | **23** | 32% |
| `full` | — | **71** | 100% |

The `core` closure is: `code-review`, `config`, `discuss-phase`, `execute-phase`, `help`,
`import`, `new-project`, `phase`, `plan-phase`, `quick`, `review`, `settings`, `surface`,
`update`, `verify-work`. `standard` adds eight more: `ingest-docs`, `manager`,
`map-codebase`, `onboard`, `pause-work`, `progress`, `resume-work`, `workspace`.

Two things fall out of those lists that no reading of the catalogue would predict.

!!! warning "The named profiles install four of the five loop steps"
    `ship` is in neither closure. The loop contract declares five steps and byte-checks that
    declaration in CI, yet `core` and `standard` install discuss → plan → execute → verify and
    stop. A `core` install can produce `UAT.md` and has nothing that consumes it.

!!! warning "The routing layer only exists at `full`"
    None of the six `ns-*` routers appear in either closure. At `core` or `standard` the
    intent-domain taxonomy is not installed at all. The installer anticipates this explicitly —
    `stageSkillsForRuntimeAsSkills()` in `src/install-profiles.cts` comments that "A partial
    surface that drops a whole router cluster falls back to flat automatically," and gates
    nesting on `[...bundles.routerStems].every((r) => present.has(r))`. Axis 2 is a
    `full`-install-only concept, by construction.

!!! warning "The 15-skill minimum installs a redundant pair"
    Both `config` and `settings` are in the `core` closure. `skills/gsd-settings/SKILL.md`
    describes "Configure GSD workflow toggles and model profile";
    `skills/gsd-config/SKILL.md` describes "Configure GSD settings — workflow toggles,
    advanced knobs, integrations, and model profile" — a strict superset — and
    `commands/gsd/config.md` includes the `settings.md`, `settings-advanced.md` and
    `settings-integrations.md` workflow fragments. Only `config` appears in `ns-manage`'s
    `requires:`, so the router partition already treats `settings` as absorbed.

    This is the sharpest evidence on the page for the limits of enforcement. The CI gate
    checks closure *reachability* — can every required skill be reached — and never closure
    *non-redundancy*. So duplication survives in the one tier that is supposed to be
    ruthlessly minimal. **Machine enforcement catches drift; it does not catch overlap.**
    If you build a profile system, know that you are buying the first guarantee and not the
    second.

This axis is the one with real teeth. `scripts/lint-skill-deps.cjs` runs two checks in CI:
that no skill body references `/gsd:<stem>` absent from its own `requires:`, and that for
every non-`full` profile, no skill in the closure requires something outside the closure —
failing with "add the missing skills to the profile base set in `install-profiles.cjs`."

### Dependency in-degree shows where the load sits

Because `requires:` drives the closure, in-degree is a direct measure of structural
centrality. Across all 57 declared `requires:` lists, every target resolves to a real
command file — zero dangling entries.

| Skill | Inbound `requires:` |
|---|---|
| `phase` | 35 |
| `config` | 12 |
| `review` | 9 |
| `plan-phase` | 9 |
| `update` | 7 |
| `progress` | 6 |

Fourteen commands declare no `requires:` at all — `audit-uat`, `capture`, `debug`,
`explore`, `help`, `import`, `ingest-docs`, `milestone-summary`, `next`, `phase`,
`profile-user`, `resume-work`, `update`, `workspace`. `lint-skill-deps.cjs` only fails on
body references *not* listed, so an absent `requires:` is legal for a skill that
cross-references nothing. The consequence is quiet but real: these fourteen contribute no
edges, so a partial install selecting one pulls in nothing alongside it.

---

## The role cross-cut the repo does not ship

Group the same 71 by *role* — planning, execution, verification, diagnostics, context,
meta — and you get a clean, complete taxonomy. You also get one that exists nowhere in the
repo. It is worth building precisely to see why it was never built.

| Role | Count | Exists to | Representative skills |
|---|---|---|---|
| **Planning / intent formation** | 20 | Turn intent into a committed artifact — spec, context, plan, roadmap | `discuss-phase`, `spec-phase`, `plan-phase`, `mvp-phase`, `ui-phase`, `ai-integration-phase`, `review`, `plan-review-convergence`, `phase`, `new-project`, `onboard`, `explore`, `sketch`, `spike` |
| **Execution** | 9 | Turn a plan into committed code | `execute-phase`, `quick`, `fast`, `autonomous`, `add-tests`, `docs-update`, `undo`, `pr-branch`, `ship` |
| **Verification** | 9 | Prove the built thing matches what was planned | `verify-work`, `code-review`, `audit-uat`, `secure-phase`, `eval-review`, `ui-review`, `validate-phase`, `audit-milestone`, `audit-fix` |
| **Diagnostics** | 6 | Answer "what went wrong / where do I stand" | `debug`, `forensics`, `health`, `stats`, `milestone-summary`, `extract-learnings` |
| **Context / intelligence** | 6 | Acquire and persist knowledge the other roles consume | `map-codebase`, `graphify`, `mempalace-recall`, `mempalace-capture`, `docs-update`, `extract-learnings` |
| **Meta / self-management** | 23 | Manage GSD itself, the session, and dispatch | `config`, `settings`, `update`, `surface`, `help`, `workspace`, `workstreams`, `thread`, `pause-work`, `resume-work`, `next`, `progress`, `manager`, `cleanup`, `inbox`, `profile-user`, `complete-milestone`, and the six `ns-*` routers |

The buckets hold 73 slots across 71 skills and cover every one of them, because exactly two
skills are genuinely dual-role: **`docs-update`** is execution (it writes files, verified
against the codebase) and context (it produces reference material), and
**`extract-learnings`** is diagnostics (post-mortem on completed artifacts) and context (it
files what it learns for later recall).

!!! note "Context/intelligence is a sixth bucket on purpose"
    Folding it into planning would be tidier but would erase a distinction the repo itself
    draws: `commands/gsd/ns-context.md` makes codebase intelligence a first-class domain with
    its own router. When your subject already separates two things, do not merge them for
    symmetry.

Two observations about this table matter more than the table.

First, **the meta bucket is the largest** — 23 of 71, nearly a third. A framework of this
size spends a third of its surface managing itself: configuration, workspaces, threads,
session pause/resume, install updates, skill surfacing, and routing. That is not overhead
to be apologised for; it is what the scale costs. Budget for it.

Second, and more importantly: **role cuts straight across the router partition.**
`ns-workflow` alone contains planning skills (`discuss-phase`, `plan-phase`), execution
skills (`execute-phase`, `autonomous`, `fast`), and verification skills (`verify-work`).
`ns-review` mixes verification (`code-review`, `secure-phase`) with diagnostics (`debug`,
`forensics`). The two taxonomies are not refinements of each other — they are genuinely
orthogonal.

!!! quote "Why the repo ships no role taxonomy"
    Role is not a property of a skill; it is a property of *why you invoked it*. `map-codebase`
    is context acquisition when you run it before planning and diagnostics when you run it to
    understand a bug. No consumer in the codebase needs to know a skill's role, so no mechanism
    declares one — and by the principle at the top of this page, an undeclared classification
    is the correct outcome, not an omission.

---

## A minor fourth axis: dispatch primitive

`allowed-tools` yields a fourth grouping for free — by what a skill is permitted to *do*,
which is a proxy for its position in the control graph.

| Primitive | Count | Group | What it means |
|---|---|---|---|
| `Agent` | 30 | Heavy orchestrators | `plan-phase`, `execute-phase`, `code-review`, `map-codebase`, … fan work out to subagents |
| `Skill` | 8 | Routers | The six `ns-*` files plus `manager` and `plan-review-convergence` — intent-to-skill dispatch |
| `SlashCommand` | 3 | State-aware launchers | `next`, `progress`, `resume-work` — detect project state, then hand off. `commands/gsd/next.md` says it plainly: "This is a launcher/router only. It never does the work itself" |
| `TodoWrite` | 1 | — | `execute-phase`, alone |

Note that the eight `Skill` holders are *not* the same set as the six routers, and the three
`SlashCommand` holders are a different dispatch style again — routing by *state* rather than
by *intent*. Three dispatch primitives, three named sets, minimal overlap.

The agent set is itself derived rather than declared: `parseCallsAgents()` in
`src/install-profiles.cts` regex-scans each skill body for `\bgsd-[a-z][a-z-]*` patterns and
stores the hits under `_calls_agents_<stem>` keys, so the installer computes which of the
subagent definitions to stage from the prose of the skills that survived the profile
closure. Nothing declares "this skill needs that agent" — it is read out of the text.

---

## What to take into your own framework

1. **Ask "who consumes this grouping?" before you create it.** GSD Core has three
   partitions because it has three consumers: an extension contract, a dispatcher, and an
   installer. It has no role taxonomy because nothing consumes one. Every taxonomy without a
   consumer is documentation you will have to maintain by hand and will not.

2. **Make the load-bearing partitions generated, not authored.** Loop position lives in
   five markers compiled by `scripts/gen-loop-host-contract.cjs`; install tier lives in one
   `PROFILES` constant expanded by transitive closure and gated by
   `scripts/lint-skill-deps.cjs`. Neither can silently disagree with reality. The one
   partition made of prose tables is the one carrying a documented alphabetical tie-break.

3. **Declare position at the layer that has it.** No skill file says which loop step it is,
   because skills are shells — the workflow underneath does the work and carries the marker.
   Putting the metadata where the behaviour lives means it cannot drift from the behaviour.

4. **A `produces:` / `consumes:` chain is the cheapest ordering you can buy.**
   `CONTEXT.md → PLAN.md → SUMMARY.md → UAT.md` is four fields across five files, and it
   places dozens of satellite skills unambiguously. Artifact dependency beats a hand-kept
   sequence number every time.

5. **Derive the dependency graph from one field and let it do multiple jobs.**
   `requires:` drives partial installs *and* the CI lint *and* the router→child map used by
   the nested layout. One declaration, three consumers, no second source of truth.

6. **Expect a third of your surface to be self-management, and expect a retroactive twin
   for every optional gate.** These are the two size-driven costs visible in this catalogue.
   Neither is a defect; both are predictable, and knowing they are coming lets you design for
   them instead of discovering them at seventy-one files.
