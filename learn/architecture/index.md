# Architecture

Every other section of this site cuts the system one way — by stage, by artifact, by
agent, by module, by capability pack. This page puts the cuts back together and then
tests the assembly against a single real command, `/gsd:execute-phase`, from the moment a
host registers it to the moment a phase is marked complete on disk.

The test matters because the strata do not compose the way the directory listing
suggests. Reading `commands/gsd/execute-phase.md` you would conclude the program is 2,940
bytes long. Reading `gsd-core/workflows/execute-phase.md` you would conclude the model
schedules the work. Reading `src/phase.cts` you would conclude it never does. All three
readings are drawn from real files, and all three are wrong in the same way: **no single
stratum contains the program.** The program is the trace across them.

## The four strata and the surface where they meet

| Stratum | Surface | What it is authoritative for |
|---|---|---|
| Prompt — authored | `commands/gsd/*.md` (71), `agents/*.md` (35) | Registration, tool grants, subagent contracts |
| Prompt — generated | `skills/gsd-*/SKILL.md` (71) | The second host surface; mechanically derived from the first |
| Procedure | `gsd-core/workflows/**` (152 files), `gsd-core/references/**` (106 `.md`) | The actual instructions; loaded lazily via `@`-include |
| Runtime | `gsd-core/bin/gsd-tools.cjs`, `src/*.cts` (194 top-level), `hooks/*.js` (22) + `hooks/*.sh` (4) | Anything whose answer must be identical on two runs |
| Extension | `gsd-core/bin/lib/loop-host-contract.cjs` (12 points), `capabilities/*/capability.json` (46 packs) | Where third-party work may attach |
| State | `.planning/**` | The only thing that survives a context reset |

The extension row is not a fifth peer — it is the seam. Twelve named loop points are the
one vocabulary that the prompt layer, the runtime and third-party packs all speak. That
is why [The Loop](../loop/index.md) argues the contract has to be generated rather than
narrated, and it is why most of the seams at the bottom of this page live there.

```kroki-plantuml
@startuml
skinparam componentStyle rectangle
skinparam shadowing false
skinparam nodesep 10
skinparam ranksep 24
skinparam defaultTextAlignment center

package "Prompt layer" {
  component "commands/gsd/*.md\n71 dispatchers" as CMD
  component "skills/gsd-*/SKILL.md\n71 generated twins" as SK
  component "agents/*.md\n35 contracts" as AG
}

package "Procedure" {
  component "gsd-core/workflows/**\n152 files" as WF
  component "gsd-core/references/**\n106 .md" as REF
}

package "Extension surface" {
  component "loop-host-contract.cjs\n12 points" as LP
  component "capabilities/*/*.json\n46 packs" as CAP
}

package "Runtime" {
  component "bin/gsd-tools.cjs\nrouting hub" as CLI
  component "src/*.cts\n194 top-level" as SRC
  component "hooks/*\n22 .js + 4 .sh" as HK
}

component ".planning/**\nstate tree" as ST

CMD --> SK  : generated, checked in CI
SK  --> WF  : @-include (lazy)
WF  --> REF : @-include (lazy)
WF  --> CLI : gsd_run <command>
CLI --> SRC : dispatchHostCommand
SRC --> ST  : typed read / guarded write
WF  --> AG  : Agent(subagent_type=...)
AG  --> ST  : writes SUMMARY.md, commits
AG  ..> HK  : tool call intercepted
HK  --> ST  : reads persisted sentinel
WF  --> LP  : loop render-hooks <point>
CAP --> LP  : declares attachments
LP  --> SRC : resolved by loop-resolver
@enduml
```

Three edges in that picture carry most of the design.

**`SK → WF` is lazy and enormous.** The host registers a manifest; the manifest pulls in
a procedure two orders of magnitude larger. Everything about progressive disclosure
follows from this edge.

**`AG ⇢ HK` is the edge whose enforcement does not depend on the model choosing to
traverse it.** A subagent's tool call is intercepted by a `PreToolUse` hook whose input is
a file on disk, not the model's arguments. That is the enforcement ceiling described on
[Build Your Own](../build-your-own/index.md).

**`WF → CLI` is unproven.** Nothing verifies that a workflow actually made the call it
says it makes. That single missing edge is the source of the largest seam on this page.

## Trace: `/gsd:execute-phase`, end to end

### Hop 1 — the registered surface is a manifest, not a program

`commands/gsd/execute-phase.md` is 2,940 bytes. Its generated twin
`skills/gsd-execute-phase/SKILL.md` is 2,900. Together they carry the frontmatter
contract ([Skills](../skills/index.md)), the tool grant, and a `<process>` block whose
only per-command content is the parenthesised gate list — "wave execution, checkpoint
handling, verification, state updates, routing."

What they pull in: `gsd-core/workflows/execute-phase.md` at 92,326 bytes / 1,532 lines,
plus 11 step files under `gsd-core/workflows/execute-phase/steps/` totalling 95,590
bytes. The corpus behind the dispatcher is 187,916 bytes — a ratio of roughly **1:64**
against the 2,940-byte artifact the host actually registers.

!!! note "What this means for a builder"
    Everything machine-visible about a 183 KiB program lives in 2,940 bytes: the tool
    grant, the description a router matches against, and a prose list of gate names.
    That is a deliberate trade — the host's context cost is the manifest, not the
    program — but it means the registered surface cannot be used to reason about, lint
    or type-check what the command does.

### Hop 2 — the `@`-include, what [Skills](../skills/index.md) calls the only operator

`@`-references are how the procedure arrives. Across `gsd-core/**`, a grep for
`@[prefix]gsd-core/<path>` returns 147 matches in four spellings: 78 anchored
`@~/.claude/gsd-core/…`, 6 `@$HOME/.claude/…`, 1 `@./.claude/…`, and **62 bare
`@gsd-core/…`**. `execute-phase.md` alone carries 18 of them — 12 anchored, 6 bare.

The step files are not loaded up front. `execute-phase.md:644` reaches
`gsd-core/references/loop-hook-dispatch.md` only when a wave is about to start;
`:967` reaches `steps/post-merge-gate.md` only after a merge. The read-depth budget
described in [Files & State](../files/index.md) is enforced here by the shape of the
include graph, not by a checker.

### Hop 3 — into the CLI

The workflow's calls all go through a `gsd_run` shell function that lands in
`gsd-core/bin/gsd-tools.cjs`. Two normalisations happen before anything dispatches:

- **`gsd-tools.cjs:4204`** strips `query` as a meta-prefix, so
  `gsd_run query phase-plan-index` and `gsd_run phase-plan-index` are the same call.
- **`gsd-tools.cjs:4218`** splits a dotted canonical form on the **first dot only**
  (`state.update` → `state` + `update`), guarding that head and remainder are both
  non-empty so `.hidden` and a bare `.` are rejected (#3243).

Dispatch is table-driven. `HOST_COMMAND_ROUTERS` is declared at `:3665`; its `init` entry
at `:3678` wraps a best-effort stale-bake warning around `routeInitCommand` inside a
`try/catch` whose comment reads *"guard must never break init"*.
`dispatchHostCommand` at `:3784` rejects `__proto__`, `constructor` and `prototype` as
command keys and uses `Object.prototype.hasOwnProperty` for the lookup — a prototype-
pollution guard on a string that came from a model-authored shell line.

From there: `src/init-command-router.cts:79` holds the `'execute-phase'` handler, which
runs `parseNamedArgs` over `validate` / `tdd` / `wave` and calls
`src/init.cts:854` `cmdInitExecutePhase`.

That dot-split at `:4218` also explains a harmless oddity: `execute-phase.md` spells the
same command two ways, 946 lines apart — `gsd_run query verification status` at `:352`
and `gsd_run query verification.status` at `:1298`. The normaliser absorbs it, which is
why the divergence has neither a consequence nor any pressure to converge.

### Hop 4 — the scheduling already happened

This is the hop that changes how you read the whole system.

`gsd-core/workflows/execute-phase.md:12-14` opens with a `<core_principle>` block:

> Orchestrator: discover plans → analyze deps → group waves → spawn agents → handle
> checkpoints → collect results.

Read that and you conclude the model analyses dependencies and groups waves. It does not.
By the time the model has a plan list, `src/phase.cts:681` `cmdPhasePlanIndex` has already
returned a finished `waves` map computed by `src/phase.cts:629` `computeDependencyLevels`
— Kahn's algorithm with longest-path levelling, O(V+E), using a head index rather than
`Array.shift()` to dequeue (cited to #307).

Three details make it a hard boundary rather than a hint:

- A `depends_on` cycle is a fail-loud `error()` at `src/phase.cts:861`, naming the cycle
  members. Not a warning, not a best-effort order.
- The planner's own declared `wave:` frontmatter is **demoted**. `src/phase.cts:908` sets
  `effectiveWave = computedWave` unconditionally; a disagreement only pushes a string
  onto `warnings` at `:911`.
- `src/phase.cts:883` picks 0- or 1-based wave numbering (`levelOffset`) depending on
  whether any plan declared wave 0 — a presentation decision also taken in code.

So one artifact field (`wave:` in `PLAN.md` frontmatter) is authored by an agent and
overridden by a graph algorithm — and the override is reported through a channel the
consumer does not read. `execute-phase.md:337` enumerates exactly what to parse out of
`phase-plan-index`: `phase`, `plans[]`, `waves`, `incomplete`, `runnable`,
`has_checkpoints`. `warnings` is not in that list. The workflow *does* know how to handle
a `warnings` array — it extracts and displays one at `:1416-1422`, from `phase.complete`'s
result. So the mechanism exists; it is simply not wired to this producer, and a planner
whose declared wave was overruled is told so by nobody.

### Hop 5 — what the model actually decides

The model's real scheduling authority in this command is one veto.
`execute-phase.md:575` requires an intra-wave `files_modified` overlap check *before*
spawning, with pseudocode at `:583-591`. If two plans in a wave touch the same file, the
wave serialises. And the instruction attributes the fault upstream (`:604`):

> The planner should have caught this; flag it as a planning defect so the user can
> replan the phase if desired.

The veto is prose-only. Nothing verifies the check ran. Contrast it with hop 4: the
dependency ordering is a total function over typed input; the file-overlap ordering is a
sentence. Both produce the same kind of decision — "these two plans must not run
together" — and they sit at opposite ends of the enforcement ladder on
[Build Your Own](../build-your-own/index.md).

### Hop 6 — rendering the loop point

`execute-phase.md:644` renders the extension surface:

```bash
WAVE_PRE_HOOKS_JSON=$(gsd_run loop render-hooks execute:wave:pre --raw)
```

which reaches `cmdLoopRenderHooks` in `src/loop-resolver.cts` (`:478`). The resolver
derives its canonical point list from `LOOP_HOST_CONTRACT` at require time — but it also
keeps a hardcoded fallback copy of all twelve point strings (`src/loop-resolver.cts:70-82`),
returned whenever the require throws or the contract yields zero points (`:69`). The
vocabulary *can* drift, and two further hardcoded copies live at
`gsd-core/bin/lib/capability-validator.cjs:28-40` and `scripts/registry-schema.cjs:66`.
Derivation with a silent fallback is weaker than derivation alone.

A census over all 46 manifests (a Node script summing `steps`, `gates` and
`contributions` per point) gives 37 attachments. Their error posture is the interesting
column:

| `onError` | Count | Effect |
|---|---|---|
| `skip` | 29 | The attachment's own error is absorbed; loop continues |
| `halt` | 7 | The attachment's own error stops the loop |
| absent | 1 | `security`'s contribution at `plan:pre` |

Read the column precisely. `gsd-core/references/loop-hook-dispatch.md:92` defines
`onError` as what to do *"if the check itself errors"* — `skip` means treat as
non-blocking and continue. It is not the same field as a gate's `blocking`, which governs
what happens when a check runs successfully and returns a failing verdict. So this table
is about crash posture, not about where the loop can be stopped on the merits.

The absence is schema-legal, not a defect: `gsd-core/bin/lib/capability-validator.cjs`
requires `onError` on steps (`:2738`) and gates (`:2818`) but treats it as optional on
contributions (`:2778`). The seven halting attachments are `ui@plan:pre`,
`ui@execute:wave:post`, `ai-integration@verify:pre`, `nyquist@verify:post`,
`security@verify:post`, `broken-windows@ship:pre` and `security@ship:pre` — five distinct
points, all of them either pre-planning or post-execution quality gates. Everywhere else,
a first-party capability that blows up is absorbed. The default posture of the extension
system is **degrade, don't block**, and that is a load-bearing choice: a plugin surface
whose failure mode is "stop the user's build" gets disabled, and a disabled extension
point is a dead one.

### Hop 7 — dispatch, and the one edge with teeth

The workflow spawns executors with `Agent(subagent_type="{EXECUTOR_TYPE}")`. That call is
intercepted before it runs. `hooks/gsd-agent-isolation-guard.js` is registered on **both**
host surfaces — `hooks/hooks.json:31-36` with matcher `Agent|Task`, and
`src/runtime-hooks-surface.cts:2028-2050` with the same matcher, widened from `Agent`-only
per #3045. Dual registration is the norm rather than the exception: 9 of the 10 scripts
`hooks/hooks.json` references also appear in `src/runtime-hooks-surface.cts` — the same
figure [The hook system](../runtime/hooks.md) reports independently. The isolation guard is
not unusual for being dual-registered; eight other scripts share the property, which is why
it reads as a defect class rather than a one-off.

The guard's input is the part worth stealing. It *does* parse the model's `Agent()`
arguments — `:421` reads `data.tool_input`, `:422` reads `subagent_type`, `:465` tests for
the isolation kwarg — but it does not *trust* them to carry the isolation decision. It also
reads `.gsd/dispatch-isolation-sentinel.json`, which
`hooks/lib/isolation-sentinel.js:20` documents as written *"as an unconditional side
effect of resolving"* the `dispatch-isolation` query itself. The workflow computed the
isolation decision in shell; the sentinel carries that value to the enforcement point
**without passing through the model.** A dropped flag in a model-authored dispatch cannot
launder a worktree-less execution past the guard, because the guard is not reading the
dispatch.

The module also states its residual risk out loud at `:37`: an already-isolated agent can
write a fabricated fresh `{isolation:"none"}` sentinel into the primary checkout. That is
documented as accepted, not designed away.

### Hop 8 — the completion oracle refuses the agent's word

`execute-phase.md:967-969` explains why:

> This addresses the Generator self-evaluation blind spot identified in Anthropic's
> harness engineering research: agents reliably report Self-Check: PASSED even when
> merging their work creates failures.

So completion is inferred, not received. The oracle at `:1061-1064` is three independent
observable checks per `SUMMARY.md`: the first two files listed under `key-files.created`
must exist on disk; `git log --oneline --all --grep="{phase}-{plan}"` must return at least
one commit; and no `## Self-Check: FAILED` marker may be present.

[Orchestration patterns](../agents/orchestration.md) documents the resulting oddity from
the agent side — `agents/gsd-executor.md` declares a `## PLAN COMPLETE` sentinel that the
orchestrator deliberately never matches, and `gsd-core/references/agent-contracts.md`
labels that mismatch *intentional behavior, not a mismatch*. What that page does not say,
and this trace adds, is that the same command distrusts the executor at **both** ends: its
scheduling was taken away in code at hop 4, and its self-report is discarded here.

### Hop 9 — a three-valued gate, written in prose

`execute-phase.md:977-999` guards the tracking writes on the post-merge test result, and
distinguishes three outcomes rather than two:

```bash
if [ "${TEST_EXIT}" -eq 0 ]; then
  # roadmap.update-plan-progress ... complete
elif [ "${TEST_EXIT}" -eq 124 ]; then
  echo "⚠ Skipping tracking update — test suite timed out. Plans remain in-progress."
else
  echo "⚠ Skipping tracking update — post-merge tests failed (exit ${TEST_EXIT})."
fi
```

Exit 124 — timeout — is neither pass nor fail. It is *inconclusive*, and inconclusive must
not advance state.

This is the prompt-layer twin of the code-layer discipline that
[State machinery](../files/dataflow.md) documents under "A zero that is not an answer":
`src/planning-scope.cts`'s frozen four-value enum exists so that `COMPLETE` with zero
items and `UNREADABLE` with zero items cannot be confused. Same insight, applied at two
strata, with two enforcement strengths — one is a frozen type every read-path module
returns, the other is a shell conditional inside a markdown file that nothing verifies
was executed. Noticing that the same idea appears in both places, at very different
prices, is the single most useful thing this trace produces.

### Hop 10 — the terminal transition, which fails closed

`src/phase.cts:2162` `cmdPhaseComplete` carries a plan-coverage gate at `:2243-2318` whose
comment is an incident report:

> `phase.complete` used to gate ONLY on a single `*-VERIFICATION.md` status, so a phase
> could close "complete" while an arbitrary number of its plans […] had no completion
> record (a confirmed production incident closed a phase with 6/30 plans unexecuted,
> including its entire final UI scope, with every tool-reported signal green).

The gate now refuses completion when any plan lacks a matching `*-SUMMARY.md`, exempting
only plans retired via machine-readable `status: superseded` frontmatter (#2349), and it
surfaces the superseded count in the error so the bypass stays visible rather than silent.
Two ordering decisions in the same block are worth copying:

- It runs **before** the verification-gate transaction, so a coverage refusal fails fast
  without mutating `ROADMAP.md` or `STATE.md`.
- An unreadable phase directory is itself a refusal. `scanPhasePlans` swallows
  `readdirSync` errors and returns an empty set, which is indistinguishable from an empty
  phase — so the gate treats it as a failure. The message at `src/phase.cts:2282` states
  the principle: *"a coverage gate that passes when it cannot read the plans is no gate at
  all."*

The same function also demonstrates a small naming discipline. It emits both
`warnings: string[]` and `preservation_warnings: {field, reason}[]` rather than merging
them, and the comment at `:2228-2237` names why: reusing one field name for two element
types is the *"Generative Fix Divergence"* anti-pattern. One command, two shapes, two
names, stated reason.

Control then reaches `execute-phase.md:1496` `delegate_post_completion_to_transition`,
which `@`-includes `transition.md` in post-completion mode and explicitly skips its
`verify_completion` and `update_roadmap_and_state` steps, because re-running
`phase.complete` would double-write state (#1526).

### The trace in one table

| Hop | Artifact | Kind | Decided by |
|---|---|---|---|
| 1 | `commands/gsd/execute-phase.md` → `skills/gsd-execute-phase/SKILL.md` | Authored + generated prompt | Host registration |
| 2 | `gsd-core/workflows/execute-phase.md` + 11 step files | Procedure | Model, lazily |
| 3 | `gsd-core/bin/gsd-tools.cjs` → `src/init-command-router.cts` → `src/init.cts` | Runtime dispatch | Code |
| 4 | `src/phase.cts` `computeDependencyLevels` / `cmdPhasePlanIndex` | Runtime | Code (planner's `wave:` overridden) |
| 5 | `execute-phase.md:575` `files_modified` overlap | Procedure | Model, unverified |
| 6 | `gsd_run loop render-hooks execute:wave:pre` → `src/loop-resolver.cts` | Extension surface | Code + capability manifests |
| 7 | `hooks/gsd-agent-isolation-guard.js` + `.gsd/dispatch-isolation-sentinel.json` | Hook | Code, on persisted shell state |
| 8 | `execute-phase.md:1061-1064` | Procedure | Model reading git + disk |
| 9 | `execute-phase.md:977-999` | Procedure | Model reading `TEST_EXIT` |
| 10 | `src/phase.cts:2162` `cmdPhaseComplete` | Runtime | Code, fail-closed |

Seven artifact kinds, three enforcement strengths, one command.

## The principles the architecture embodies

**1. Predicates are code; the substance they gate is prose.** This is
[Runtime](../runtime/index.md)'s central argument, and the trace confirms it hop after hop:
the *mechanics* of ordering, dispatch, isolation and completion are compiled; *what to
build* and *whether it is any good* never are.

!!! note "A judgement this page owes the reader"
    [Runtime](../runtime/index.md) enumerates five things that got real code — parsing,
    state transitions, concurrency, portability, trust. Wave scheduling
    (`src/phase.cts:629`) does not sit cleanly in any of them. It is not a
    position-in-workflow transition; it is an ordering computation over a graph. My call
    is that it does **not** earn a sixth category, because all five are already instances
    of one rule — *anything whose answer must be identical on two runs is code* — and a
    topological level is a fact about the DAG, not a judgement about the work. That is
    exactly why it can override the planner's declared `wave:` at `:908` and demote the
    disagreement to a warning at `:911`. If you are drawing this line in your own
    framework, apply the rule, not the list.

**2. Distrust the producer's report; observe the artifact.** The completion oracle reads
git and the filesystem instead of the agent's return channel (hop 8). The consent system
recomputes a bundle hash instead of trusting a ledger's own integrity field
([Capabilities](../capabilities/index.md)). Same move, different layer.

**3. "I could not see" is not "there is nothing there."** `src/planning-scope.cts`'s
`UNREADABLE`, `cmdPhaseComplete`'s unreadable-directory refusal, and the `TEST_EXIT=124`
branch are three expressions of one rule: distinguish an absent answer from a negative
one, and never let the absent one advance state.

**4. Carry decisions in state, not in arguments, when a model sits in between.** The
isolation sentinel (hop 7) is the cleanest instance. If enforcement must survive a model
that forgot a flag, the enforcement point cannot read the flag.

**5. Declare the whole extension surface before you have extensions.** All twelve loop
points exist from the start because you cannot renumber a point third parties have bound
to. [The Loop](../loop/index.md) makes the case; the next section is the bill.

**6. When one fact has two representations, generate the second and check it in CI.** 71
skills from 71 commands; the loop contract from workflow markers;
`src/loop-resolver.cts`'s point list derived at require time. Every place the repo does
this, the two copies agree. Most of the drift catalogued below sits where it did not.

## Where the seams are

None of these is fatal, and each is the visible cost of a decision made deliberately
elsewhere.

### `execute:pre` validates, installs, materialises, and never fires

`execute:pre` is one of the canonical twelve in
`gsd-core/bin/lib/loop-host-contract.cjs:51`. It is in the validator's accepted set
(`gsd-core/bin/lib/capability-validator.cjs:33`). It ships as an empty bucket in the
generated registry (`gsd-core/bin/lib/capability-registry.cjs:4358`).
`tests/capability-registry.test.cjs:994` asserts that a step declared at `execute:pre`
consuming `PLAN.md` is **accepted**.

But `grep -l '"execute:pre"' capabilities/*/capability.json` returns 0 files, and
`grep -rl 'render-hooks execute:pre ' gsd-core/workflows/` returns 0 files. So no
first-party capability binds it, and no workflow in the shipped corpus renders it. A
third-party capability declaring a step there would validate, install, obtain consent,
materialise into `byLoopPoint` — and, against the shipped corpus, never be rendered.

I searched `gsd-core/bin/lib/`, `scripts/` and `src/` for a warning about a declared point
with no renderer and found none; the "declared but not rendered" messages that grep did
surface, `src/loop-resolver.cts:385` and `:393`, are about a *fragment path*, not a point.
That is a negative result from one grep, not a proof, but it is the state of the tree as I
read it. The cheap mitigation for a pre-declared surface is a build-time warning when a
declared point has no renderer.

### Point *ownership* is enforced; point *rendering* is not scoped

The loop-host marker assigns each point to exactly one step, and
`scripts/gen-loop-host-contract.cjs` asserts that ownership. Nothing constrains which
workflow may call `loop render-hooks <point>`. Counting files under
`gsd-core/workflows/` that contain a render site (`grep -rl`, with total occurrences in
parentheses where they differ):

| Point | Files rendering it | Attachments bound to it |
|---|---|---|
| `discuss:pre` / `discuss:post` | 1 / 1 | 1 / 1 |
| `plan:pre` | 1 (3 sites) | 13 |
| `plan:post` | 1 | 4 |
| `execute:pre` | **0** | **0** |
| `execute:wave:pre` | 1 | 1 |
| `execute:wave:post` | 1 | 6 |
| `execute:post` | 5 (7 sites) | 3 |
| `verify:pre` | 1 | 1 |
| `verify:post` | **6** | 4 |
| `ship:pre` / `ship:post` | 1 / 1 | 2 / 1 |

Two readings. First, the busiest attachment site is not the busiest render site:
`plan:pre` carries 13 of 37 attachments from a single rendering file, while `verify:post`
carries 4 attachments and is rendered from six. Second, `execute-phase.md:1170` renders
`verify:post` from inside its own `aggregate_results` step — a workflow rendering a point
it does not own. A capability author reasoning from the contract will predict one
invocation and can get up to six, in steps they have never read.

### The `WF → CLI` edge has no linter

`node scripts/lint-command-contract.cjs` reports `71 command files checked, 0 violations`
and `152 workflow files, 152 reachable, 0 unreachable`. Genuinely clean. But what it
checks is reachability and `@`-ref shape, not execution.

Across `execute-phase.md` and its 11 step files there are **76 `gsd_run` call sites**, in
**28 distinct `gsd_run query <sub>` forms** — 16 of them `config-get` alone. Each is a
round-trip the model can simply not make. No linter, hook or test proves that any of them
happened. This is the same gap as hop 5's unverified `files_modified` veto, and it is the
structural reason the runtime keeps taking work *away* from the prompt layer (hops 4, 7
and 10): the only reliable way to know a check ran is to run it in code.

### Two `@`-include spellings, one rewriter

Of the 147 `@gsd-core/…` references under `gsd-core/**`, 62 are the bare form. The
installer's `_applyRuntimeRewrites` in `src/runtime-artifact-conversion.cts` rewrites only
*anchored* prefixes — `~/.claude/`, `$HOME/.claude/`, `./.claude/` and the per-runtime
homes — so bare references are not matched by that rewrite and pass through verbatim into
the installed tree.
`execute-phase.md` uses both spellings, 12 anchored and 6 bare.

I did not verify how a bare `@gsd-core/…` resolves at read time on any host, so this is
stated as an unverified seam, not a defect. On disk today it is clean: stripping trailing
sentence punctuation, every distinct target resolves against the repo layout except a
documentation placeholder (`@gsd-core/workflows/foo.md`, an example inside
`gsd-tools.cjs:2682`) and one prose mention of `gsd-core/cli`. Nothing lints `@`-refs
inside `gsd-core/**` at all — `lint-command-contract`'s Rules 4 and 5 scan only
`commands/gsd/*.md` (`COMMANDS_DIR` at `:35`, globbed at `:146`). The surface happens to be
clean, which is exactly why the gap is easy to miss.

### The one progressive-disclosure tree the reachability linter cannot see

Rule 6's reachability set is `gsd-core/workflows/**` only, and the resolver it uses,
`workflowPathRefs` (`scripts/command-contract-helpers.cjs:98`), matches only paths
containing `workflows/` or `<x>/(steps|modes|templates)/…`. `references/` is invisible to
it **by construction**.

Consequence: of the 106 `.md` files under `gsd-core/references/`, seven have their
filename appear in no markdown under `commands/`, `agents/`, `skills/` or `gsd-core/**` —
`api-coverage.md`, `artifact-types.md`, `decimal-phase-calculation.md`,
`git-planning-commit.md`, `offer-next.md`, `planner-graphify-auto-update.md`,
`planner-human-verify-mode.md`. Their filenames appear in no markdown under `commands/`,
`agents/`, `skills/` or `gsd-core/**` — shipped with the install, byte-checked by install
parity, named by no prompt. (They are not invisible everywhere: `docs/INVENTORY.md`,
`docs/ARCHITECTURE.md` and several tests do name them.)

`artifact-types.md` is among them — the file whose own opening line supplies the thesis
this site uses to judge everything else, that a well-formatted artifact no workflow reads
is inert.

### Descriptors and ADRs that describe an older wiring

`docs/adr/1143-claude-orchestration-capability.md:114-118` states that the capability
registers at `execute:wave:post into:executor` and that *"`execute:wave:pre` and
`execute:pre` are declared in the contract but not wired today."* The shipped
`capabilities/claude-orchestration/capability.json` declares contributions at
`execute:wave:pre` (`:59`) and `plan:post` (`:72`) — not `wave:post` — and
`execute-phase.md:644` renders `execute:wave:pre`. Only the `execute:pre` half of the
ADR's claim still holds.

The same stale belief also lives in a **shipped descriptor**, not just in docs:
`capabilities/external-job/capability.json` explains in-manifest that it registers at
`execute:wave:post` because *"`wave:pre` is declared in the loop host contract but not
rendered."* That justification no longer describes the tree. Documentation drift in
`docs/` is this site's standing assumption; drift inside a capability manifest that ships
to users is a step further, because it is the artifact a third-party author copies.

### A vocabulary the type system does not know about

`src/host-integration.cts:47` freezes `subagentToolkit`'s vocabulary as
`['full', 'read-only', 'built-in-only']`. The TypeScript type at `:88` is
`'full' | 'read-only'`. `capabilities/kimi-code/capability.json` ships
`"subagentToolkit": "built-in-only"`.

No code branches on the third value: negotiation collapses anything `!== 'full'` to
`read-only`, `canNest` does the same, and `resolveDispatchType` keys on
`namedDispatch === false` instead. So the drift is narrow and I traced no runtime
consequence — but it is the *inverse* of the pattern [Skills](../skills/index.md) praises
elsewhere in this codebase, where union literals are mirrored as frozen objects precisely
because the type erases at runtime. Here the frozen object is richer than the type, and
the type is the thing a new call site will be checked against.

### What the seams have in common

Five of the seven are the same shape: **a fact represented twice, where only one copy is
checked.** A point declared in a contract and rendered in a workflow. A wave declared in
frontmatter and computed in a DAG. A capability's registration point in an ADR, in a
manifest and in a workflow. An include spelled two ways with a rewriter that knows one. A
vocabulary frozen at runtime and narrowed in the type system.

The repo already knows the fix and applies it in the places that matter most — 71 skills
generated from 71 commands, the loop contract generated from workflow markers,
`src/loop-resolver.cts` deriving its point list rather than copying it. Each of those is
guarded by a check rather than by discipline, and the guards pass on this tree —
`node scripts/lint-command-contract.cjs` reports 0 violations across 71 command files and
152 workflow files. The seams are the pairs where generation was not applied.

## If you were rebuilding this

Three things this trace argues for, in the order you would need them.

**Make the enforcement strength of every rule explicit and visible.** The trace crosses
prose-only checks (hop 5), prose-with-observable-evidence (hop 8), hook interception (hop
7) and compiled predicates (hops 4 and 10) without ever announcing which is which. A
builder reading `execute-phase.md` top to bottom cannot tell the veto that is enforced
from the veto that is merely written. Annotating that in the source — even as a comment
convention — would cost nothing and would make the enforcement ladder legible from inside
the artifact instead of only from a survey like this one.

**Pre-declare your extension points, then add a renderless-point check the same day.**
`execute:pre` is what the good decision costs when the cheap guard is skipped: accepted,
consented, registered, inert.

**Generate the second copy of every fact, or expect it to drift.** That is not a
generalisation from theory — it is what the seven seams above have in common, and what
the three generated surfaces conspicuously do not.
