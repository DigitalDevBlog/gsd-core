# Files & State

GSD Core has no database, no server and no session object. Everything one loop stage
knows about another is a file on disk under `.planning/`. That makes the artifact set the
system's actual data model, and it makes "which files exist, who writes them, who reads
them" the highest-leverage question you can ask about the design.

The repo answers that question in its own words, in the opening line of
`gsd-core/references/artifact-types.md`:

!!! quote "The repo's thesis about its own state"
    "A well-formatted artifact that no workflow reads is inert — the consumption
    mechanism is what gives an artifact meaning."

That sentence is the test this page applies throughout. It is also, as we will see, a test
several of GSD Core's own registry entries fail.

## Why this page exists at all

[The Loop](../loop/index.md) established that the generated contract in
`gsd-core/bin/lib/loop-host-contract.cjs` declares exactly one artifact per stage —
`CONTEXT.md` → `PLAN.md` → `SUMMARY.md` → `UAT.md`, a single strand with no fan-in. That
is the *minimum*: the artifact whose absence means the stage cannot run.

The actual read-set is larger, is not declared anywhere machine-checkable, and — the part
that makes it genuinely undeclarable — **is not even fixed**. When
`gsd-core/workflows/execute-phase.md` builds an executor prompt it unconditionally reaches
for `STATE.md` (line 41, "Read STATE.md before any operation to load project context")
and `PROJECT.md` (line 431). Then it branches on the configured context window: at
`CONTEXT_WINDOW >= 500000`, "Executor agents receive prior wave SUMMARY.md files and the
phase CONTEXT.md/RESEARCH.md" (line 140) — three more artifacts that a 200k-model run
never sees. The verifier's read-set is different again: "all PLAN.md, SUMMARY.md,
CONTEXT.md files plus REQUIREMENTS.md" (line 141).

Declaring the minimum is what makes the chain checkable. Declaring the whole read-set
would mean declaring a set whose membership depends on a config key — which is why the
contract does not try, and why this page has to. The tiering itself is the subject of
[Mechanism 1](#mechanism-1-a-read-depth-budget-that-forbids-reading-bodies) below.

```kroki-plantuml
@startuml
skinparam componentStyle rectangle
skinparam shadowing false
skinparam backgroundColor transparent
skinparam defaultTextAlignment center

package "declared in loop-host-contract.cjs" {
  component "PLAN.md" as PLAN
  component "SUMMARY.md" as SUM
}

package "read unconditionally" {
  component "STATE.md" as ST
  component "PROJECT.md" as PR
}

package "read only when\ncontext_window >= 500000" {
  component "CONTEXT.md" as CX
  component "RESEARCH.md" as RS
  component "prior-wave\nSUMMARY.md" as PS
}

PLAN -[thickness=3]-> SUM : contract edge\n(consumes / produces)

ST ..> SUM
PR ..> SUM
CX ..> SUM
RS ..> SUM
PS ..> SUM

note bottom of SUM
  Solid = the one artifact the
  contract declares for the
  execute step, and the only
  edge that is machine-validated.
  Dotted = what execute-phase.md
  adds when building the executor
  prompt. Three of those five
  edges exist only on 1M-class
  models. The verifier's read-set
  is a different set again.
end note
@enduml
```

## Three registries, three different answers to "what is an artifact"

Before cataloguing anything, note that GSD Core maintains **three** independent lists of
its own artifacts, organised by three different criteria, and none of them is the union of
the other two.

| Registry | Organising principle | Self-description |
|---|---|---|
| `gsd-core/templates/README.md` | by **template** — one row per file that has a skeleton | "the authoritative index" |
| `src/artifacts.cts` (`CANONICAL_EXACT`, `CANONICAL_PATTERNS`) | by **enforcement** — the set `gsd-health` will not flag | "Enumerates the file names that gsd workflows officially produce at the `.planning/` root level" |
| `gsd-core/references/artifact-types.md` | by **consumption** — shape, lifecycle, location, "Consumed by" | "Each type has a defined shape, lifecycle, location, and consumption mechanism" |

The membership differences are not cosmetic:

- `src/artifacts.cts` carries `WINDOWS.md`, `STATE-ARCHIVE.md` and `milestone.lock`.
  None of the three appears in `gsd-core/templates/README.md`.
- `gsd-core/references/artifact-types.md` carries `HANDOFF.json`, `METHODOLOGY.md`,
  `SPIKE.md` / `DESIGN.md`, spike and sketch `README.md` / `MANIFEST.md`,
  `WRAP-UP-SUMMARY.md`, `DISCUSSION-LOG.md` and `USER-PROFILE.md`. None of those appears
  in either of the other two lists.
- `gsd-core/templates/README.md` carries `BACKLOG.md`, `THREADS.md` and `LEARNINGS.md` as
  `*(inline)*` — canonical artifacts with no template file at all. (It marks
  `RETROSPECTIVE.md` the same way, but that one is a registry error; see
  [below](#the-registry-omits-artifacts-that-have-templates).)

### The enforcement path is inverted

`gsd-core/templates/README.md` states its own enforcement claim plainly: "if a
`.planning/` root file is not listed here, `gsd-health` will flag it as W019
(unrecognized artifact)", and it instructs "Agents should query this file before treating
a `.planning/` file as authoritative."

The rule itself is `checkW019` in
`src/health-diagnostic-rules/milestone-archive-hygiene.cts`. It never reads the README.
It lists `.planning/` root files (via `buildPlanningRootFilesField` in
`src/planning-snapshot.cts`) and calls `isCanonicalPlanningFile` from `src/artifacts.cts`.
Its remedy string then sends the user *back* to the doc it did not consult: "See
templates/README.md for the canonical artifact list."

So the "authoritative index" is documentation about an enforcement set it does not
define, and the enforcement set points at the documentation for its own explanation. Each
is authoritative about the other.

!!! warning "The maintenance procedure points at a file that no longer exists"
    `gsd-core/templates/README.md`'s 3-step "Adding a New Canonical Artifact" opens with:
    "Add the file name to `CANONICAL_EXACT` in `gsd-core/bin/lib/artifacts.cjs`."

    That file is gone. `src/artifacts.cts` records why in its own header: "ADR-457
    build-at-publish: the hand-written `bin/lib/artifacts.cjs` collapsed to a TypeScript
    source of truth." The registry survived the refactor; the instructions for editing it
    did not. This is exactly the drift a generated contract prevents and a hand-written
    doc does not — and it is worth contrasting with the loop contract, where
    `scripts/gen-loop-host-contract.cjs --check` runs in CI precisely so this cannot
    happen.

### Two ways W019 is weaker than the prose implies

Reading `src/artifacts.cts` and `checkW019` together turns up two gaps that a prose
description of the same rule would hide.

**The pattern list has a catch-all.** `CANONICAL_PATTERNS` contains
`/^v\d+\.\d+(?:\.\d+)?-MILESTONE-AUDIT\.md$/i` — precise — followed by
`/^v\d+\.\d+(?:\.\d+)?-.*\.md$/i`, commented "other version-stamped planning docs". The
second subsumes the first. Any file named `v1.2-anything-at-all.md` is canonical. For
version-prefixed names, W019 is off.

**The rule only inspects Markdown.** `checkW019` opens with
`if (!filename.endsWith('.md')) continue;`. `CANONICAL_EXACT` lists `config.json` and
`milestone.lock`, so neither can ever be reached by W019.

`isCanonicalPlanningFile` does have other consumers — `src/planning-snapshot.cts`,
`src/artifacts.cts`, `src/health-diagnostic-rules/milestone-archive-hygiene.cts`, and
`scripts/lint-planning-artifact-writer-drift.cjs`, which runs in the `lint:ci` chain
(`package.json:122`). That linter resolves `config.json` against the set, so `config.json`
is reachable after all. `milestone.lock` is the one genuinely inert row: no consumer
resolves it. The registry is *mostly* live, with a single dead entry — a narrower defect
than a whole inert section, and a more realistic one.

The transferable point is not "GSD Core has bugs here." It is that **a registry is only
as real as the code path that consults it**, and a project that keeps three registries
will keep three different truths. If you are building this, pick the enforcement set as
the source and generate the human-readable table from it.

## The artifact set, grouped by role

There are 46 files under `gsd-core/templates/` — 32 Markdown templates at the top level
plus the `gsd-core/templates/README.md` registry, 7 in `gsd-core/templates/codebase/`,
5 in `gsd-core/templates/research-project/`, and `gsd-core/templates/config.json`. That
number is *not* the artifact count, for three reasons worth stating before the tables:

1. Some canonical artifacts have no template — `BACKLOG.md`, `THREADS.md`,
   `LEARNINGS.md` are `*(inline)*` in the registry.
2. Some templates are not artifacts at all. `gsd-core/templates/planner-subagent-prompt.md`
   and `gsd-core/templates/debug-subagent-prompt.md` produce *prompts*, not files.
3. Some templates produce files outside `.planning/` entirely —
   `gsd-core/templates/claude-md.md` targets project-root `CLAUDE.md`,
   `gsd-core/templates/copilot-instructions.md` targets a host-integration file, and
   `gsd-core/references/artifact-types.md` locates `USER-PROFILE.md` at
   `~/.claude/gsd-core/USER-PROFILE.md`.

The useful grouping is not by directory but by **write pattern** — how often the file
changes, and who is allowed to change it. That is the axis that predicts everything else
about how the file is designed.

### 1. Identity and scope — written rarely, read constantly

| Artifact | Template | What it pins down |
|---|---|---|
| `.planning/PROJECT.md` | `gsd-core/templates/project.md` | Project identity, core value, full decision log |
| `.planning/ROADMAP.md` | `gsd-core/templates/roadmap.md` | Phase plan, milestones, progress |
| `.planning/REQUIREMENTS.md` | `gsd-core/templates/requirements.md` | Numbered acceptance criteria with traceability |
| `.planning/config.json` | `gsd-core/templates/config.json` | Per-project GSD configuration |

These are the files everything else `@`-references. They change at milestone boundaries,
not at plan boundaries. `gsd-core/templates/config.json` is worth flagging as a *seed*
rather than a schema: it ships 44 leaf keys against the 78 in
`gsd-core/bin/shared/config-defaults.manifest.json`, and documented keys such as
`workflow.context_guard_mode` (defaulted in the manifest, read by
`gsd-core/references/execute-phase-context-guard.md`, documented in
`gsd-core/references/planning-config.md` and `gsd-core/references/context-budget.md`)
appear nowhere in the scaffold. The default arrives by absence.

### 2. Position and session — high-churn digests, some designed to be deleted

| Artifact | Template | Write pattern |
|---|---|---|
| `.planning/STATE.md` | `gsd-core/templates/state.md` | Continuously rewritten; hard 100-line cap |
| `.planning/STATE-ARCHIVE.md` | *(none)* | Eviction target for `cmdStatePrune` in `src/state.cts` |
| `.planning/HANDOFF.json` | *(none)* | One-shot; deleted after resume |
| `.../.continue-here.md` | `gsd-core/templates/continue-here.md` | One-shot; "not permanent storage" |
| `.planning/milestone.lock` | *(none)* | Claim file (`src/milestone-lock.cts`) |

This bucket is where the finite-context design is most visible, and it gets its own
section below.

### 3. The phase chain — written once per phase, then read forever

| Artifact | Template | Produced by |
|---|---|---|
| `NN-CONTEXT.md` | `gsd-core/templates/context.md` | `/gsd:discuss-phase` |
| `NN-RESEARCH.md` | `gsd-core/templates/research.md` | `/gsd:plan-phase` |
| `NN-SPEC.md` | `gsd-core/templates/spec.md` | phase spec authoring |
| `NN-MM-PLAN.md` | `gsd-core/templates/phase-prompt.md` | `/gsd:plan-phase` |
| `NN-MM-SUMMARY.md` | `gsd-core/templates/summary.md` (+ 3 tiers) | `/gsd:execute-phase` |
| `NN-VERIFICATION.md` | `gsd-core/templates/verification-report.md` | verifier |
| `NN-UAT.md` | `gsd-core/templates/UAT.md` | `/gsd:validate-phase` |
| `DISCOVERY.md` | `gsd-core/templates/discovery.md` | pre-planning discovery |
| `NN-DISCUSSION-LOG.md` | `gsd-core/templates/discussion-log.md` | `/gsd:discuss-phase` |
| `{phase}-USER-SETUP.md` | `gsd-core/templates/user-setup.md` | plans with `user_setup` |

Each destination is declared in the template's own first prose line — e.g.
`gsd-core/templates/summary.md` opens "Template for
`.planning/phases/XX-name/{phase}-{plan}-SUMMARY.md`". That convention means the
template file *is* the routing table; you never have to hunt for where an artifact lands.

`gsd-core/templates/verification-report.md` is the interesting member. It closes a
goal-backward loop opened in `gsd-core/templates/phase-prompt.md`, which defines
`must_haves: {truths, artifacts, key_links}` in PLAN.md frontmatter on the stated premise
that "Task completion ≠ Goal achievement. A task 'create chat component' can complete by
creating a placeholder." The verification report consumes exactly those three lists back
as three tables, with a status vocabulary that distinguishes `✓ EXISTS + SUBSTANTIVE`
from `✗ STUB` from `✗ NOT WIRED` from `⚠️ PRESENT_BEHAVIOR_UNVERIFIED` from
`✓ VERIFIED (coincidental-reliance)`. The schema names "present but never exercised" and
"true only by accident" as distinct outcomes — which is only possible because the
expectation was persisted at plan time rather than re-derived at verify time.

### 4. Conditional contracts — exist only when a capability is active

| Artifact | Template | Gate |
|---|---|---|
| `NN-UI-SPEC.md` | `gsd-core/templates/UI-SPEC.md` | `/gsd:ui-phase` |
| `NN-SECURITY.md` | `gsd-core/templates/SECURITY.md` | `/gsd:secure-phase` |
| `NN-AI-SPEC.md` | `gsd-core/templates/AI-SPEC.md` | `/gsd:ai-integration-phase` |
| `NN-VALIDATION.md` | `gsd-core/templates/VALIDATION.md` | `/gsd:plan-phase` (Nyquist) |
| `NN-PATTERNS.md` | *(inline)* | pattern mapper |
| `NN-REVIEWS.md` | *(inline)* | `/gsd:review` |

These are the on-disk footprint of the capability system — the artifacts a loop-point
attachment declares in its `produces`. Their optionality is the reason the loop contract
declares only *core* artifacts: a chain that included `UI-SPEC.md` would be unsatisfiable
on a project with UI phases turned off.

Notice also that all four templated members of this bucket are **literal skeletons** —
files that are themselves valid artifacts, copyable byte-for-byte, rather than a skeleton
wrapped in a fenced block with advisory prose around it. `gsd-core/templates/UI-SPEC.md`,
`gsd-core/templates/SECURITY.md` and `gsd-core/templates/VALIDATION.md` open directly on
`---` frontmatter; `gsd-core/templates/AI-SPEC.md` opens directly on the artifact's own
H1. Compare the ~38 templates elsewhere in the tree that open `# <Name> Template` and
carry the artifact inside a fence. Contract artifacts get contract-shaped templates.

### 5. Cross-phase ledgers — append and accumulate

| Artifact | Template | Accumulation model |
|---|---|---|
| `.planning/WINDOWS.md` | *(none)* | Defect register; `src/broken-windows.cts` |
| `.planning/LEARNINGS.md` | *(inline)* | Phase retrospective learnings |
| `.planning/BACKLOG.md` | *(inline)* | Deferred work |
| `.planning/THREADS.md` | *(inline)* | Persistent discussion threads |
| `.planning/MILESTONES.md` | `gsd-core/templates/milestone.md` | Completed-milestone log |
| `.planning/RETROSPECTIVE.md` | `gsd-core/templates/retrospective.md` | Living, updated at each close |
| `.planning/debug/[slug].md` | `gsd-core/templates/DEBUG.md` | APPEND-only sections |

This is the bucket the placeholder for this page called "the ledger", and the literal
ledger is `.planning/WINDOWS.md`. Its module docstring in `src/broken-windows.cts`
describes it as "a cross-phase ledger of small defects (stubs, TODOs, skipped tests, lint
warnings, unrun verifies, unmet truths, deviations)" and it is the sharpest example in
the repo of the dual-channel design discussed below.

`.planning/LEARNINGS.md` has a global counterpart that is easy to miss: `src/learnings.cts`
maintains a *cross-project* store at `~/.gsd/knowledge/`, one JSON file per learning with
SHA-256 content-hash deduplication. Project memory and practitioner memory are separate
stores with separate lifetimes.

### 6. Derived exports — regenerated, never hand-edited

| Artifact | Template | Generator |
|---|---|---|
| project-root `CLAUDE.md` | `gsd-core/templates/claude-md.md` | `generate-claude-md` + `generate-claude-profile` |
| copilot instructions | `gsd-core/templates/copilot-instructions.md` | host integration |
| `dev-preferences` | `gsd-core/templates/dev-preferences.md` | `/gsd:profile-user` |
| `USER-PROFILE.md` | `gsd-core/templates/user-profile.md` | `profile-user` |
| `.planning/codebase/*.md` | `gsd-core/templates/codebase/` (7 files) | codebase-mapping agent |
| `.planning/research/*.md` | `gsd-core/templates/research-project/` (5 files) | research synthesis |

The `CLAUDE.md` generator is the one to study. `gsd-core/templates/claude-md.md` defines
seven marker-bounded regions of the form
`<!-- GSD:{name}-start source:{file} -->` … `<!-- GSD:{name}-end -->`, each with declared
fallback text, a fixed ordering, and a `source:` attribute the template says "enables
targeted updates when source files change." Ownership is split at the marker, not the
file: `generate-claude-md` owns six regions, `generate-claude-profile` owns `profile`
exclusively. That is how you get idempotent regeneration of a file that two independent
producers and one human all write to.

The two subdirectory families are otherwise uniform in shape and inconsistent in naming.
`gsd-core/templates/codebase/architecture.md` produces `.planning/codebase/ARCHITECTURE.md`;
`gsd-core/templates/research-project/ARCHITECTURE.md` produces
`.planning/research/ARCHITECTURE.md`. Same destination casing, opposite source casing.

### 7. Archive — frozen snapshots

`/gsd:complete-milestone` moves `vX.Y-ROADMAP.md`, `vX.Y-REQUIREMENTS.md`,
`vX.Y-MILESTONE-AUDIT.md` and optionally `vX.Y-phases/` into `.planning/milestones/`.
`gsd-core/templates/README.md` encodes a state-machine expectation in prose here: finding
a `vX.Y-MILESTONE-AUDIT.md` at the `.planning/` root after completion "indicates the
archive step was skipped." A file's *location* is a status field.

## Lifecycle of the load-bearing artifacts

`gsd-core/references/artifact-types.md` supplies shape / lifecycle / location /
consumed-by for the core set. What it does not supply — and what turns out to be the most
design-bearing column — is **who is allowed to write**.

| Artifact | Created when | Read by | Updated by |
|---|---|---|---|
| `PROJECT.md` | `/gsd:new-project` | Everything, via `@.planning/PROJECT.md` | Orchestrator, at milestone boundaries |
| `ROADMAP.md` | `/gsd:new-project`, `/gsd:new-milestone` | `plan-phase`, `discuss-phase`, `execute-phase`, `progress`, `state` | **Orchestrator only** — executors are explicitly forbidden |
| `REQUIREMENTS.md` | `/gsd:new-milestone` | `discuss-phase`, `plan-phase`, CONTEXT generation | Executor marks items complete |
| `STATE.md` | `/gsd:new-project`, `/gsd:health --repair` | All orchestration workflows; `resume-project`, `progress`, `next` | Code, through one locked write seam in `src/state.cts` |
| `STATE-ARCHIVE.md` | First `state prune` past the cutoff | Humans, forensic audit | `cmdStatePrune` only |
| `NN-CONTEXT.md` | `/gsd:discuss-phase` | `plan-phase` (decisions), `execute-phase` (code_context, canonical_refs) | Superseded by the next phase, not edited |
| `NN-RESEARCH.md` | `/gsd:plan-phase` | Planner | Write-once |
| `NN-MM-PLAN.md` | Planner agent | Executor; task commits reference plan IDs | Planner revision loop, pre-execution only |
| `NN-MM-SUMMARY.md` | Executor, at plan completion | Orchestrator (progress), planner (future context), verifier, `milestone-summary` | Executor appends `## Self-Check: PASSED\|FAILED` at runtime |
| `NN-UAT.md` | `/gsd:validate-phase` | `ship`, and `plan-phase --gaps` | OVERWRITE + APPEND per `<section_rules>` |
| `WINDOWS.md` | First `windows append` | `ship:pre` gate, via `jq` on frontmatter | `windows append` / `waive` / `fixed` |
| `debug/[slug].md` | `/gsd:debug` | The debugger after a context reset | Per-section IMMUTABLE / OVERWRITE / APPEND |
| `HANDOFF.json` | `/gsd:pause-work` | `resume-project` | Replaced wholesale; **deleted after resume** |

### Single-writer discipline is stated in the prompt, not enforced by code

The `updated-by` column hides the sharpest rule in the system. When
`gsd-core/workflows/execute-phase.md` dispatches a parallel executor into a git worktree,
the prompt it builds says, in the `<objective>` block:

> Do NOT update STATE.md or ROADMAP.md — the orchestrator owns those writes after all
> worktree agents in the wave complete.

and repeats it in `<parallel_execution>`: "IMPORTANT: Do NOT modify STATE.md or
ROADMAP.md." The reason is concurrency. Multiple executors run simultaneously; two of
them updating a shared progress file would produce a lost update. So the shared files get
exactly one writer — the orchestrator, after the wave joins — while each executor writes
only files it uniquely owns (`SUMMARY.md`, and `REQUIREMENTS.md` for its own requirement
IDs).

That is a *prompt-level* invariant, but it is not left to the prompt alone. The same
block records a code-level backstop: "execute-plan.md auto-detects worktree mode (`.git`
is a file, not a directory) and skips shared file updates automatically." Belt and
braces — the instruction states the rule so the agent understands it, and the runtime
enforces it so a disobedient agent cannot break it.

The same file also records the durability rule that follows from putting state in files:

> REQUIRED: SUMMARY.md MUST be committed before you return. … the orchestrator
> force-removes the worktree after you return, and any uncommitted SUMMARY.md will be
> permanently lost.

If your state medium is the filesystem and your isolation mechanism is a worktree, then
"committed to git" — not "written to disk" — is what durable means.

### STATE.md: prose written by agents, frontmatter derived by code

`STATE.md` is not the only artifact written by a code module — `ROADMAP.md` goes through
`platformWriteSync` in `src/roadmap.cts` (lines 1081 and 1311), `WINDOWS.md` through
`src/broken-windows.cts`, and `milestone.lock` through `src/milestone-lock.cts`. What makes
`STATE.md` distinctive is the *split*: its prose is authored by agents while its
frontmatter is derived by code, in the same file. Reading `src/state.cts` explains why. Three things happen on every write:

1. **Frontmatter is re-derived from the body, not authored.** `syncStateFrontmatter`
   strips the frontmatter, hands the *body* to `buildStateFrontmatter`, and rebuilds
   the machine layer from what the prose actually says. The template's own frontmatter
   admits this — `gsd_state_version: '1.0'  # placeholder; syncStateFrontmatter
   overwrites on first state.* call`.
2. **The write is atomic under a lock.** The read-modify-write helper "holds the lock
   across the entire read → transform → write cycle, preventing the lost-update problem
   where two agents read the same content and the second write clobbers the first."
3. **The composition is a single named function.** `syncAndPreserveStateMd` exists
   because, per the ADR quoted in its docstring, "Assembling the stages at a call site is
   a re-derivation even when every step calls the owner" — a drift found live in a
   different command where every individual step called the right owner and the
   composition still diverged.

The generalisable design here: **the digest that everything reads is the one artifact you
cannot afford to let an LLM format.** Let the agent write the narrative; derive the
queryable fields from the narrative in code; serialise the writes.

## The finite-context argument, assembled

Everything above is downstream of one threat model, stated repeatedly across the repo:
the agent reading these files has a bounded context window, will lose that context
mid-task, and cannot be trusted to re-read everything on resume. Five distinct mechanisms
answer it.

### Mechanism 1 — a read-depth budget that forbids reading bodies

`gsd-core/references/context-budget.md` is prescriptive, not advisory:

| Context Window | SUMMARY.md | VERIFICATION.md | PLAN.md (other phases) |
|---|---|---|---|
| < 500k (200k model) | Frontmatter only | Frontmatter only | Current phase only |
| ≥ 500k (1M model) | Full body permitted | Full body permitted | Current phase only |

It also bans reading agent definition files (`subagent_type` auto-loads them) and bans
inlining large files into subagent prompts — "tell agents to read files from disk
instead."

`gsd-core/workflows/execute-phase.md` implements the tiering by querying
`config-get context_window` and branching: at ≥ 500k "Executor agents receive prior wave
SUMMARY.md files and the phase CONTEXT.md/RESEARCH.md"; below 200k, executor prompts drop
their inline examples and load them on demand from
`gsd-core/references/executor-examples.md`. **The read-set is a function of the budget.**
The artifact set stays constant; how much of it gets read is configuration.

### Mechanism 2 — frontmatter as the read surface

Mechanism 1 makes frontmatter design load-bearing, and
`gsd-core/templates/summary.md`'s `<frontmatter_guidance>` says so directly: "Frontmatter
is first ~25 lines, cheap to scan across all summaries without reading full content", and
`requires` / `provides` / `affects` "create explicit links between phases, enabling
transitive closure for context selection."

Read those two rules together and the causal chain is explicit: **frontmatter is the read
surface because the budget rule forbids reading bodies.** Planning context is assembled by
scanning YAML across dozens of summaries and computing a transitive closure over the
dependency edges — never by reading summary prose.

`.planning/WINDOWS.md` is the purest instance because there the split is enforced. From
`src/broken-windows.cts`:

> Frontmatter holds scalar counts (the FAST path the ship gate reads via jq without
> parsing JSON). The JSON code block is the AUTHORITATIVE entries source. The two must
> agree; read paths cross-check and fail closed on drift.

And `gsd-core/workflows/ship.md` consumes exactly that fast path —
`jq -r '.ledger.open_count'` — with a stated fail-closed policy: an unreadable or
non-numeric count blocks the ship rather than passing it, because "the gate is strict
equality to `0`; never ship on an unreadable ledger."

That is the complete pattern worth stealing: a cheap scalar summary for the hot read
path, an authoritative structured body for correctness, a cross-check between them, and a
fail-closed default when the cheap path is unreadable.

### Mechanism 3 — size treated as a correctness property

`gsd-core/templates/state.md` does not ask nicely:

> Keep STATE.md under 100 lines. It's a DIGEST, not an archive. … The goal is "read once,
> know where we are" — if it's too long, that fails.

It also prescribes the eviction policy — keep 3–5 recent decisions (full log lives in
`PROJECT.md`), drop resolved blockers — and `src/state.cts` implements it as a command:
`cmdStatePrune` resolves the current phase, computes `cutoff = currentPhase - keepRecent`
(default 3), and moves decisions, completed summaries and resolved blockers older than the
cutoff into `.planning/STATE-ARCHIVE.md`.

The same stance appears elsewhere. `gsd-core/templates/DEBUG.md`'s `<size_constraint>`
requires "No narrative prose - structured data only" and reframes a long Evidence list as
a *diagnostic signal* of circular investigation. `gsd-core/templates/phase-prompt.md`'s
scope guidance caps a plan at "2-3 tasks per plan, ~50% context usage maximum".

A framework that persists state between stages needs an answer to unbounded growth. GSD
Core's answer is a stated cap, a named eviction rule, and an archive file — not a
convention that someone will trim it later.

### Mechanism 4 — the tier is chosen by code, not by the agent

`SUMMARY.md` ships in three sizes — `gsd-core/templates/summary-minimal.md`,
`gsd-core/templates/summary-standard.md`, `gsd-core/templates/summary-complex.md` — and
`cmdTemplateSelect` in `src/template.cts` picks one from the PLAN, mechanically:

```text
taskCount  = count of /###\s*Task\s*\d+/g
fileCount  = distinct backticked paths containing '/'
hasDecisions = /decision/gi matches anywhere in the plan

minimal   if taskCount <= 2 && fileCount <= 3 && !hasDecisions
complex   if hasDecisions || fileCount > 6 || taskCount > 5
standard  otherwise (and on any read error)
```

Taking the sizing decision away from the generating agent makes summary size a
*reproducible function of plan shape* rather than a judgement call that drifts with model
temperature. The fallback direction is the tell: any read error lands on `standard`, the
middle tier, never on `minimal`.

The heuristic is also crude in a way worth noticing. `hasDecisions` is a bare
case-insensitive `/decision/gi` scan over the whole plan text, so a plan that merely uses
the word "decision" in a sentence of prose is routed to the richest tier.

### Mechanism 5 — mutability contracts that survive a context reset

`gsd-core/templates/DEBUG.md` labels every section with one of three write modes in a
`<section_rules>` block:

| Mode | Sections | Why |
|---|---|---|
| IMMUTABLE | `trigger`, `created`, Symptoms (after gathering) | The original problem statement must not drift as theories change |
| OVERWRITE | `status`, `updated`, Current Focus, Resolution | Always reflects now; no history needed |
| APPEND-only | Eliminated, Evidence | "Prevents re-investigating dead ends after context reset" |

The same three-way vocabulary reappears in `gsd-core/templates/UAT.md`'s
`<section_rules>` (`phase` / `source` / `started` IMMUTABLE, Current Test and Summary
OVERWRITE, Gaps APPEND-only).

And it is paired with a mandated **read order** for resume, in the same template's
`<resume_behavior>` block:

> 1. Parse frontmatter → know status
> 2. Read Current Focus → know exactly what was happening
> 3. Read Eliminated → know what NOT to retry
> 4. Read Evidence → know what's been learned
> 5. Continue from next_action
>
> The file IS the debugging brain.

That last sentence is the design premise made explicit. `gsd-core/templates/UAT.md`'s
`<lifecycle>` carries an isomorphic four-step "Resume after `/clear`". Two artifacts,
same recipe — because the recipe is not about debugging or UAT, it is about surviving
amnesia.

!!! tip "The pairing is the point"
    APPEND-only alone is just an audit trail. APPEND-only **plus** a mandated read-order
    that puts the appended section third — before the agent forms a new hypothesis — is
    what makes negative knowledge actually prevent repeated work. If you build the
    mutability contract without the read recipe, the file grows and nobody consults the
    part that saves time.

## Designed-to-be-deleted state

Not every persisted artifact is meant to last. GSD Core has a small class of files whose
correct end state is deletion, and it treats that as a first-class category.

`gsd-core/workflows/pause-work.md` writes *two* files at the same moment, for two
different readers: "Create structured `.planning/HANDOFF.json` and `.continue-here.md`
handoff files … The JSON provides machine-readable state for `/gsd:resume-work`; the
markdown provides human-readable context." `gsd-core/workflows/resume-project.md` reads
`HANDOFF.json` first, then searches for `.continue-here*.md` at up to three locations
(phase, non-phase, legacy root fallback), and instructs: "**After successful resumption,
delete HANDOFF.json** (it's a one-shot artifact)."

`gsd-core/templates/continue-here.md` states the same contract from the template side —
"This file gets DELETED after resume - it's not permanent storage" — and imposes a
requirement that only makes sense for a throwaway: "The `<next_action>` should be
actionable without reading anything else."

This is the dual-channel principle taken to its limit. `SUMMARY.md` keeps the machine
layer and the human layer in one file, separated by `---`. The pause handoff separates
them into two files entirely, because the two readers arrive at different times through
different paths and neither should have to parse the other's format.

## Copy-forward for constraints, `@`-reference for background

There is one more state-handoff rule, and it is a deliberate exception to "don't
duplicate data."

`gsd-core/templates/research.md`'s `<user_constraints>` instructs the researcher to copy
locked decisions **verbatim** out of `CONTEXT.md` into `RESEARCH.md`, with the
justification "These MUST be honored by the planner." Meanwhile
`gsd-core/templates/phase-prompt.md`'s `<context>` and
`gsd-core/templates/planner-subagent-prompt.md`'s `<planning_context>` pass background by
`@`-include — `@.planning/PROJECT.md`, `@.planning/ROADMAP.md`, `@.planning/STATE.md`.

So the rule is: **constraints an agent must not violate travel by value; background an
agent may consult travels by reference.** The cost is duplication and the staleness risk
that comes with it. The benefit is that each agent's input is self-contained — a
constraint cannot be missed because a referenced file was never opened, and it cannot be
lost when the referencing file is truncated to fit a budget.

Once you accept that agents read on a budget, "the constraint is in a file you were told
to read" is not good enough. That is the whole argument.

## Where the map disagrees with the territory

The findings below are all verifiable in the repo as it stands. They are collected here
rather than scattered because together they show something more useful than any one of
them: registries drift toward the code that consults them, and the ones nothing consults
drift furthest.

### The same artifact has two homes

`gsd-core/templates/README.md` lists `NN-DEBUG.md` as a *phase-subdirectory* artifact.
`gsd-core/templates/DEBUG.md`'s own first line says "Template for
`.planning/debug/[slug].md` — active debug session tracking", and its `<lifecycle>`
resolves by moving the file to `.planning/debug/resolved/`. Two incompatible locations
for one template.

A similar split affects `CLAUDE.md`. `gsd-core/templates/README.md` files it under
"`.planning/` Root Artifacts", a table that opens "These files live directly at
`.planning/`". `gsd-core/templates/claude-md.md` says "Template for project-root
`CLAUDE.md`."

### The registry omits artifacts that have templates

`gsd-core/templates/README.md`'s phase-subdirectory table has no row for `NN-SPEC.md`
(`gsd-core/templates/spec.md`), `NN-VERIFICATION.md`
(`gsd-core/templates/verification-report.md`), `DISCOVERY.md`
(`gsd-core/templates/discovery.md`), `NN-DISCUSSION-LOG.md`
(`gsd-core/templates/discussion-log.md`) or `{phase}-USER-SETUP.md`
(`gsd-core/templates/user-setup.md`). The README notes that this table is "NOT checked by
W019 (which only inspects the `.planning/` root)", so nothing breaks — but the same file
tells agents to query it "before treating a `.planning/` file as authoritative", which a
missing row defeats.

It also lists `RETROSPECTIVE.md` with Template = *(inline)* while
`gsd-core/templates/retrospective.md` exists as a full skeleton, and gives
`gsd-core/templates/milestone-archive.md` no row at all despite that template's own usage
note saying "Save to `.planning/milestones/v{VERSION}-{NAME}.md`".

### One template is the odd file out on three axes at once

`gsd-core/templates/milestone-archive.md` is the only template with a `## File Template`
heading and **no fence** — so the artifact's own `## Overview` / `## Phases` /
`## Milestone Summary` headings sit at the same document level as the template's own
`## Usage Guidelines`, leaving the output/instruction boundary unmarked. It is one of only
four `{{mustache}}` templates in a repo that otherwise uses `[brackets]` for
LLM-authored prose and `{single_brace}` for scalar substitution. And it twice cites
`PROJECT-STATE.md` (lines 82 and 109) — a filename that appears nowhere else in the
repository, in no registry, and that does not correspond to either canonical name
(`PROJECT.md`, `STATE.md`).

### The tiered summaries drift from their own canonical template, asymmetrically

`gsd-core/templates/summary.md` marks `requirements-completed: []` as "REQUIRED — Copy
ALL requirement IDs from this plan's `requirements` frontmatter field". None of
`gsd-core/templates/summary-minimal.md`, `gsd-core/templates/summary-standard.md` or
`gsd-core/templates/summary-complex.md` declares the key.

Worse, the divergence runs the wrong way:

| Field | minimal | standard | complex |
|---|---|---|---|
| `actuals:` | ✓ | ✓ | **✗** |
| `requires:` | ✗ | ✗ | ✓ |
| `patterns-established:` | ✗ | ✗ | ✓ |
| `requirements-completed:` | ✗ | ✗ | ✗ |

The *richest* tier is missing a field both leaner tiers carry. Since all three do carry
the `coverage:` comment block and an identical `**Status (#2830):**` paragraph, this is
not simple staleness — the files were maintained together and diverged anyway.

### Three authoritative-sounding lists of required PLAN.md frontmatter

| Source | Declares required |
|---|---|
| `gsd-core/templates/phase-prompt.md` (Frontmatter Fields table) | phase, plan, type, wave, depends_on, files_modified, autonomous, requirements, must_haves |
| `gsd-core/references/agent-contracts.md` (Planner → Executor) | …the same, minus `must_haves` and `user_setup` |
| `src/template.cts` `cmdTemplateFill` case `'plan'` | …the same, minus **`requirements`** — the one field `gsd-core/templates/phase-prompt.md` bolds as "**MUST** list requirement IDs from ROADMAP" |

The code emits a plan skeleton without the field the template calls mandatory.

### A section that is regex-matched exists in no template

`gsd-core/references/agent-contracts.md`'s "Executor → Verifier (via SUMMARY.md)" contract
requires a `Self-Check | PASSED or FAILED` section and a `metrics` frontmatter key.
Neither string exists anywhere under `gsd-core/templates/`. `Self-Check` is appended at
*runtime* — `gsd-core/workflows/execute-plan.md` instructs "Append `## Self-Check: PASSED`
or `## Self-Check: FAILED` to SUMMARY", mirrored in `agents/gsd-executor.md` — and
`gsd-core/workflows/execute-phase.md` then matches on `## Self-Check: FAILED`.

So a section that control flow branches on is invisible to anyone reading the artifact's
template. This is the clearest single illustration of the reference doc's own thesis
inverted: here the consumption mechanism exists and the *declaration* is the thing that
is missing.

!!! note "The repo is unusually honest about this class of problem"
    `gsd-core/references/agent-contracts.md` opens by declaring itself descriptive —
    "This doc describes what IS, not what should be. Casing inconsistencies are documented
    as they appear in agent source files." It grants named, permanent exemptions
    (`gsd-verifier`'s title-case `## Verification Complete` is "an auditable exemption,
    never deleted and never silently passed"), records a marker that was *deleted*
    because it case-collided with another agent's, and defines an
    `(unconsumed: <reason>)` annotation so a deliberately-orphaned marker is visibly
    distinct from a real one. And unlike the artifact registries, its `Consumed by` / `Kind`
    columns are machine-checked — `scripts/check-contract-drift.cjs` cross-references the
    table against what agents emit and what workflows consume, so "a stale row is a
    violation the check will report."

    Compare the two: the contract table has a drift checker and stays accurate. The
    artifact registries do not, and they are where every inconsistency on this page lives.

## What to persist between stages, and why

Reading the whole artifact set back, GSD Core's implicit answer to the transferable
question is a short list.

**Persist the decisions, not the conversation.** `CONTEXT.md` captures what was decided
and why; `DISCUSSION-LOG.md` keeps the audit trail separately and
`gsd-core/references/artifact-types.md` states plainly that it is "Consumed by: Human
review; not read by automated workflows." Separating those two means the automated read
path never pays for the audit trail.

**Persist the expectation before the work, so verification can be goal-backward.**
`must_haves` in `PLAN.md` frontmatter exists at plan time specifically so
`VERIFICATION.md` can check the goal rather than re-derive it from what was built.

**Persist negative knowledge explicitly.** The APPEND-only `Eliminated` section is the
only artifact in the system dedicated to what is *not* true, and its stated purpose is
saving a post-reset agent from repeating work.

**Persist one digest and cap it.** `STATE.md` at 100 lines with a prune command is the
"read once, know where we are" surface. Everything else is reachable from it.

**Persist a scannable projection of everything else.** Frontmatter on `SUMMARY.md`,
`PLAN.md`, `VERIFICATION.md` and `WINDOWS.md` exists so the orchestrator can join across
dozens of files without opening any of them.

**Do not persist the resume note.** `HANDOFF.json` and `.continue-here.md` are explicitly
one-shot. State that outlives its moment becomes misleading state.

!!! tip "If you are building this"
    - **One registry, generated.** Make the enforcement set the source of truth and
      generate the human-readable table from it, with a `--check` in CI. Three
      hand-maintained lists produce three truths; this repo has the receipts.
    - **Split every artifact into a scan layer and a read layer**, and write down which
      one is authoritative. `src/broken-windows.cts` does this explicitly — counts in
      frontmatter for speed, JSON body for truth, cross-checked, fail closed.
    - **Give the digest a hard cap and an eviction command.** A size limit without a
      `prune` is a suggestion.
    - **Label sections IMMUTABLE / OVERWRITE / APPEND, and publish the resume read-order
      next to them.** The labels without the recipe do not save any work.
    - **Copy constraints by value; reference background.** Assume any `@`-referenced file
      may go unread under budget pressure.
    - **Assign exactly one writer per shared file**, state it in the prompt, and back it
      with a runtime check. Parallelism turns a shared progress file into a lost-update
      bug immediately.
    - **Derive machine fields from agent-written prose in code**, not by asking the agent
      to keep two representations in sync.
    - **Name the artifacts that are meant to be deleted**, and delete them.

## Where this goes next

The write-ownership rules above are enforced by prompts belonging to specific subagents;
the marker vocabulary those agents use to hand control back is covered in
[Agents](../agents/index.md). The code that owns `STATE.md`'s write seam, the health
diagnostics that consume `src/artifacts.cts`, and the frontmatter parser behind the
scan-layer reads all live in [Runtime](../runtime/index.md). The conditional artifacts in
bucket 4 — `UI-SPEC.md`, `SECURITY.md`, `AI-SPEC.md` — exist only because a capability
attached at a loop point declared them, which is [Capabilities](../capabilities/index.md).
And the template-authoring mechanics glimpsed here (the fence as an output/instruction
boundary, the four placeholder idioms, pseudo-XML doing two different jobs) are a
prompt-engineering subject rather than a state-design one.
