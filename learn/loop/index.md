# The Loop

GSD Core has two loops with the same name, and telling them apart is the whole lesson of
this section.

The first is **narrated**: `docs/explanation/the-phase-loop.md` walks a reader through
Discuss → UI design (optional) → Plan → Execute → Verify → Ship. It is a description of
what a human experiences, and it is accurate at that job.

The second is **contracted**: five steps and twelve named attachment points, declared in
markers inside five workflow files, compiled into a committed JavaScript module, and
byte-compared in CI. It is a machine-readable description of where the loop can be
extended, what each step produces, and which subagent roles it is allowed to spawn.

The repo's own `docs/` tree names the contract but never teaches it: `docs/adr/894-capability-declaration-format.md`
carries a section titled "The Loop Host Contract", and `docs/ARCHITECTURE.md` and
`docs/INVENTORY.md` each give it a one-line mention. What is missing everywhere is the
construction — how the markers become a module, and why that shape was chosen. This page
is about the second one, because a narration tells you what to expect and a contract
tells you what to build.

!!! info "Reading this as construction, not usage"
    Everything below answers "how is this assembled, and what pressure produced that
    shape?" — not "how do I run a phase." For the user-facing walkthrough, the repo's own
    `docs/explanation/the-phase-loop.md` is genuinely good and this site does not
    duplicate it.

## The stage set is generated, not authored

The most surprising fact about GSD Core's loop is that its stage list is a build artifact.

Each of the five step workflows opens with a structured HTML comment. This is the entire
declaration in `gsd-core/workflows/plan-phase.md`, lines 1–7:

```markdown
<!-- gsd:loop-host
step: plan
points: plan:pre, plan:post
agent-roles: researcher, planner, checker
produces: PLAN.md
consumes: CONTEXT.md
-->
```

`scripts/gen-loop-host-contract.cjs` reads exactly five files —
`discuss-phase.md`, `plan-phase.md`, `execute-phase.md`, `verify-work.md`, `ship.md`,
listed in pipeline order in its `STEP_WORKFLOWS` constant — parses one marker block from
each, and emits `gsd-core/bin/lib/loop-host-contract.cjs`, whose header says
`DO NOT EDIT BY HAND`.

The generator is not a formatter. It runs three validations that a hand-maintained list
could not:

| Validation | What it enforces | Where |
|---|---|---|
| Per-step point ownership | Each step must declare *exactly* its canonical points — `execute` must declare all four of `execute:pre`, `execute:wave:pre`, `execute:wave:post`, `execute:post`, and nothing else | `EXPECTED_POINTS_BY_STEP` in `scripts/gen-loop-host-contract.cjs` |
| Global 12-point coverage | The union of all declared points must equal the canonical twelve — no duplicates, no strays, no gaps | `assertPointsCoverage()` |
| Role → agent cross-check | Every non-`orchestrator` role must map through `ROLE_TO_AGENT` to an agent name that actually appears in that workflow's text | `crossCheckRoles()` |

The third check is the interesting one, and it is honest about its own limits. Its
word-boundary regex exists so that `gsd-plan-checker-v2` cannot satisfy a declared
requirement for `gsd-plan-checker`, and the source comment states plainly that this is
"a presence check (any reference in the file), not a spawn-site check — a known
limitation; spawn-site checks would require AST-level analysis."

That is the shape worth stealing: a cheap structural check that catches the common drift
(a role declared but the agent renamed or removed) while documenting exactly which class
of drift it still lets through.

The loop is then guarded the same way every other generated surface in this repo is.
`package.json` wires `node scripts/gen-loop-host-contract.cjs --check` into the
`lint:generated-sync` script, and `lint:generated-sync` runs inside `lint:ci`. If a marker
changes and nobody regenerates, CI fails. The stage list cannot drift from the workflows
that implement it, because it is derived from them.

## The cycle

```kroki-plantuml
@startuml
skinparam componentStyle rectangle
skinparam shadowing false
skinparam defaultTextAlignment center
skinparam backgroundColor transparent

component "ROADMAP.md\nphase selection\n(outside the contract)" as R

package "Loop Host Contract — 5 steps, 12 points" {
  component "discuss\ndiscuss:pre / discuss:post\nroles: orchestrator" as D
  component "plan\nplan:pre / plan:post\nroles: researcher, planner, checker" as P
  component "execute\nexecute:pre / wave:pre\nwave:post / execute:post\nroles: executor, verifier" as E
  component "verify\nverify:pre / verify:post\nroles: orchestrator" as V
  component "ship\nship:pre / ship:post\nroles: orchestrator" as S
}

R --> D : query roadmap.analyze
D --> P : CONTEXT.md
P --> E : PLAN.md
E --> V : SUMMARY.md
V --> S : UAT.md
V --> P : Gaps section, stable gap_id\n--gaps then --gaps-only
S --> R : produces: nothing\nre-entry is external

note right of P
  capabilities attach at points:
  ui -> plan:pre produces UI-SPEC.md
  (the narrated "UI design" step)
end note
@enduml
```

## What each stage reads and writes

This table is transcribed from `gsd-core/bin/lib/loop-host-contract.cjs`, not composed
for this page. The `coreArtifacts` and `agentRoles` fields are literally what the
generated module exports.

| Step | Points | Agent roles | Consumes | Produces |
|---|---|---|---|---|
| `discuss` | `discuss:pre`, `discuss:post` | orchestrator | — | `CONTEXT.md` |
| `plan` | `plan:pre`, `plan:post` | researcher, planner, checker | `CONTEXT.md` | `PLAN.md` |
| `execute` | `execute:pre`, `execute:wave:pre`, `execute:wave:post`, `execute:post` | executor, verifier | `PLAN.md` | `SUMMARY.md` |
| `verify` | `verify:pre`, `verify:post` | orchestrator | `SUMMARY.md` | `UAT.md` |
| `ship` | `ship:pre`, `ship:post` | orchestrator | `UAT.md` | *(none)* |

Two things follow from reading the contract as data rather than prose.

**The artifact chain is a single strand.** Each step consumes exactly what its predecessor
produced. There is no fan-in at the contract level and no step that reads two upstream
artifacts. That is a deliberate narrowing: the *workflows* read far more than this — the
executor prompt in `gsd-core/workflows/execute-phase.md` pulls `*-CONTEXT.md` and
`*-RESEARCH.md` alongside the plan file — but the contract declares only the **core**
artifact, the one whose absence means the step cannot run. Declaring the minimum makes the
chain checkable; declaring everything would make it noise.

**`agentRoles` is the real discriminator between stages.** It is not decoration; it is
the answer to "what design pressure does this stage answer?"

## Design pressure, read off the role column

| Step | Roles | The pressure that shape answers |
|---|---|---|
| `discuss` | orchestrator only | Extracting a user's intent is not delegable. `gsd-core/workflows/discuss-phase.md` states its own stance: "You are a thinking partner, not an interviewer. The user is the visionary — you are the builder." A fresh-context subagent has no access to the human, so this step must run in the conversation that has one. |
| `plan` | researcher, planner, checker | Planning is where *adversarial separation* pays. Three distinct roles exist so the agent that writes the plan is not the agent that judges it — `gsd-planner` produces, `gsd-plan-checker` finds issues, and a revision loop runs between them. |
| `execute` | executor, verifier | The only step with four points, because it is the only step with an inner iteration: waves. `execute:wave:pre` / `execute:wave:post` fire per wave, not per step, which is what makes parallel work observable to anything attached to the loop. Same producer/judge split as `plan`. |
| `verify` | orchestrator only | Acceptance is a human judgement. `gsd-core/workflows/verify-work.md` says "User tests, Claude records. One test at a time. Plain text responses." Declaring no subagent roles is a design claim, not an omission: there is nothing here a fresh-context agent could do, because the missing information lives in a person's head. |
| `ship` | orchestrator only | Producing a PR body is a *read* of everything already written — `gsd-core/workflows/ship.md` assembles it from the planning artifacts. Nothing new is generated, so nothing needs delegating. |

Notice that the two steps with subagent roles are exactly the two steps that produce
volume, and the three orchestrator-only steps are exactly the three that need either a
human or nothing but existing files. The contract makes that pattern visible; no prose
description of the loop does.

## Why the cycle is closed, and where it isn't

A one-shot pipeline would stop at `ship`. GSD Core's loop closes in two places, at two
different scales — and, being honest about the source, it does *not* close in a third
place people assume it does.

### Closure 1 — the gap cycle (across steps, contracted)

This is the real return edge, and it is mechanical rather than conventional.

1. `verify` writes failures into the `## Gaps` section of `UAT.md` as structured YAML with
   a **stable identifier**: `gap_id: G-{phase}-{N}`
   (`gsd-core/workflows/verify-work.md`).
2. `plan` re-entered with `--gaps` reads those gaps and produces plans tagged
   `gap_closure: true`, whose frontmatter carries the `gap_ids` they address.
3. `execute` re-entered with `--gaps-only` scopes the run to just those plans. The
   projection is automatic: `gsd-core/workflows/plan-phase.md` sets `GAPS_EXEC_FLAG` to
   `--gaps-only` for a `--gaps` planning run so the handoff it prints points at the
   narrow scope, not the whole phase.
4. `verify` re-entered runs `<step name="reconcile_gaps">` first: for each gap still
   marked `status: failed`, it looks for a `*-PLAN.md` whose `gap_ids` include that
   `gap_id` **and** a matching `*-SUMMARY.md` proving the plan executed. If both exist the
   gap flips to `status: resolved` with `resolved_by` and `resolved_at`.

The workflow states the failure mode this prevents, in its own words: without
reconciliation, "verify-work re-diagnoses them as fresh blockers and spawns new gap plans
— losing the verification state."

That is the whole design in one sentence. **A closed loop needs identity, not just a back
edge.** If the return path cannot recognise the work item it emitted on the previous
iteration, re-entering the loop destroys progress instead of converging. The stable
`gap_id`, the `gap_ids` tag on the fix plan, and the SUMMARY-exists proof are three parts
of one idempotency mechanism. And the loop stays honest about regressions: a resolved gap
reported broken again gets a *fresh* `gap_id` rather than reopening the old one, so
"fixed once, broke again" is distinguishable from "never fixed."

### Closure 2 — the revision loop (inside one step)

`plan` contains its own convergence loop, and it is a real state machine rather than a
retry counter. `gsd-core/workflows/plan-phase.md` tracks `iteration_count` (capped at 3)
alongside `stall_reentry_count` (capped at 2), detects a stall when the checker's issue
count stops decreasing rather than only when the cap is hit, and offers an interactive
branch on stall. Re-entry resets `iteration_count` but `stall_reentry_count` persists
across re-entries — which is what stops "adjust the approach and try again" from being an
infinite loop. The terminal state is an explicit marker, `## PLANNING INCONCLUSIVE`, not
a silent give-up. Stall watching itself is factored out into
`gsd-core/workflows/plan-phase/steps/stall-detection-helpers.md`.

The transferable point: bounded iteration needs *two* counters at different scopes, and a
named terminal state that downstream logic can branch on.

### Where it does not close: `ship` produces nothing

Read the contract's last row again. `ship` consumes `UAT.md` and its `produces` list is
empty — the marker in `gsd-core/workflows/ship.md` literally reads `produces:` with no
value, and the generator explicitly supports empty list fields.

So the contracted artifact chain **terminates**. Nothing inside the five steps produces
the input to the next iteration's `discuss`. Re-entry comes from outside: phase selection
is a roadmap query, not a loop artifact. `gsd-core/workflows/plan-phase.md` describes this
as an orchestrator responsibility, because `query init.plan-phase` and
`query roadmap.get-phase` both require an explicit phase number — the orchestrator runs
`gsd_run query roadmap.analyze` and reads `next_phase`, the first phase whose `disk_status`
is `no_directory`, `empty`, `discussed`, or `researched`.

The milestone loop sits further out still and is also uncontracted:
`gsd-core/workflows/complete-milestone.md` carries **no** `gsd:loop-host` marker — only
five files do. It archives the roadmap and requirements, then "offer[s] to create next
milestone inline," which is a prose handoff, not a declared edge.

!!! note "Three loops, one contract"
    The **phase loop** (discuss → ship) is contracted and machine-checked. The **phase
    sequence loop** (ship → next phase) runs on `ROADMAP.md` plus a CLI query. The
    **milestone loop** (last phase → archive → new milestone) runs on prose. All three
    are real; exactly one of them is enforceable. If you are rebuilding this, that is the
    scope decision to make consciously — GSD Core contracted the layer where extensions
    needed to attach, and left the outer layers as convention.

## Why *points*, not just stages

Five stages would be a taxonomy. Twelve points are an interface — and the difference is
the entire reason the contract is generated rather than narrated.

Points are attachment sites. `capabilities/*/capability.json` files declare `steps` and
`gates` that bind to a point by name. From `capabilities/ui/capability.json`:

```json
{
  "point": "plan:pre",
  "ref": { "skill": "ui-phase" },
  "produces": ["UI-SPEC.md"],
  "consumes": ["CONTEXT.md"],
  "when": "workflow.ui_phase",
  "onError": "skip"
}
```

Of the 46 capability manifests in `capabilities/`, 19 bind to a loop point, through three
different arrays — 16 `steps`, 10 `gates` and 11 `contributions`, 37 attachments in all.
The distribution is itself informative: `plan:pre` is by far the busiest site with 13 of
those 37, because most quality machinery wants to run *before* planning commits to an
approach rather than after execution has to be undone. The next busiest,
`execute:wave:post` with 6, is the other natural place to intervene — the moment a wave's
output first exists.

This resolves the apparent conflict between the narrated loop and the contracted one.
`docs/explanation/the-phase-loop.md` narrates an optional "UI design" step between Discuss
and Plan producing `UI-SPEC.md`. The contract has no `ui` step — because UI design is not
a stage, it is `capabilities/ui/capability.json` attaching at `plan:pre` with the same
inputs and outputs, gated on `workflow.ui_phase`. Both descriptions are correct. One
describes what the user sees; the other describes where a third party may hook in. **You
cannot rebuild an extension point you do not know exists**, which is why a system with a
plugin surface needs the second description even when the first reads better.

Because attachments declare their own `produces` / `consumes`, the same vocabulary that
describes the host steps also types the extensions — and the validator exploits that.
`gsd-core/bin/lib/capability-validator.cjs` imports `LOOP_HOST_CONTRACT` and builds
`HOST_ARTIFACT_EARLIEST_POINT_IDX` by mapping each core artifact to the index of the
`:post` point of the step that produces it. It then checks *consumes-satisfiability*: a
capability step cannot consume an artifact at a point that runs before anything produces
it, and cannot satisfy its own `consumes` from its own `produces`. Ordering among
attachments at the same point is resolved by a topological sort over their
produces/consumes edges, with cycle detection that fails the build rather than picking an
arbitrary order.

`src/loop-resolver.cts` is the query side of the same data. Given a point name it filters
the materialized capability registry by config activation and returns the active hooks —
and it derives its canonical point list from `LOOP_HOST_CONTRACT` at require time
specifically "so it cannot drift from the host contract," rather than keeping its own
copy.

That is the closed circle: one marker set defines the points, one generator validates and
compiles them, one validator type-checks extensions against them, one resolver serves them
at runtime, and one CI check keeps all of it matching the workflows.

## The same repo declares stage structure twice — one way is enforceable

Worth ending on, because it is the sharpest contrast in the codebase.

The command layer also names stage structure. Thin dispatcher commands carry a `<process>`
block whose only per-command content is a parenthetical list of gate names —
`commands/gsd/plan-phase.md` says its gates are "validation, research, planning,
verification loop, routing"; `commands/gsd/execute-phase.md` says "wave execution,
checkpoint handling, verification, state updates, routing." Thirteen of the 71 files in
`commands/gsd/` carry such a list in the strict `Preserve all workflow gates (...)` form;
`routing` appears in exactly 5 of those. In 10 of the 13, that parenthetical is the *only*
per-command content in the whole `<process>` block — everything around it is identical
two-line boilerplate. (A further 9 files mention "workflow gates" in looser phrasings,
which is why a careless grep reports 22 — the vocabulary is not standardised either.)

Nothing parses those names. There is no canonical gate vocabulary, no coverage assertion,
no `--check`. They are advisory prose aimed at a model, and they cost nothing to write.

The loop points are the same kind of claim — "this workflow has these stages" — expressed
as data, and every one of them is validated against a canonical set at build time by
`scripts/gen-loop-host-contract.cjs`, type-checked against extension manifests by
`gsd-core/bin/lib/capability-validator.cjs`, and byte-compared in CI via
`lint:generated-sync`.

The lesson is not that prose is wrong. Cheap advisory vocabulary is the right call when
nothing else consumes it — the gate names never leave the prompt. The lesson is that the
moment a second party needs to bind to your stage structure, the structure has to become
data, and the data has to be derived from the implementation rather than maintained
beside it. GSD Core made that transition for exactly one of its two stage vocabularies,
and you can see precisely which one by asking which has a generator.

!!! tip "If you are building this"
    - Author stage structure **inside** the file that implements the stage, as a marker.
      A separate registry file is a drift vector.
    - Generate the machine-readable form and commit it, then `--check` it in CI. Committed
      + regenerable beats gitignored, because it stays diffable in review.
    - Declare only the *core* artifact each stage consumes and produces. The minimum is
      checkable; the full read-set is not.
    - Give the return edge an **identity**, not just an arrow. Stable ids on the work
      items are what make re-entry converge instead of duplicating.
    - Name your extension points before you have extensions. Adding them later means
      renumbering something that third parties already bound to.

## Where this goes next

The stages above are implemented in `gsd-core/workflows/`, where the files run from
roughly 30 KB to 92 KB. The command and skill layer sitting in front of them is a thin
dispatcher measured in kilobytes — covered in [Skills](../skills/index.md), not here. The
subagent roles the contract declares are covered in [Agents](../agents/index.md), the
artifact chain's on-disk form in [Files & State](../files/index.md), and the attachment
mechanism in [Capabilities](../capabilities/index.md). Per-stage internals — the wave
scheduler, the revision loop's full state machine, the gate taxonomy — belong on sibling
pages under this section rather than in an overview.
