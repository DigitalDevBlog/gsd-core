# Orchestration patterns

Thirty-five agent definitions live in `agents/`. Only thirty-four are reachable. Single-agent
dispatch is common — `audit-milestone.md`, `secure-phase.md`, `ui-review.md`,
`code-review.md`, `validate-phase.md` and `explore.md` each spawn exactly one agent once —
but the interesting behaviour lives in the compositions. Across
`gsd-core/workflows/` the same five compositions recur, and the interesting thing about
them is not the shapes themselves (a researcher/writer/verifier triad is not a novel idea)
but that **every expensive shape in this repo ships with a documented, mechanical route back
to a cheaper one.** The framework never commits to the costly composition; it encodes the
exit alongside it.

This page is about those five compositions: which agents actually participate, how control
returns in each, what each one costs in model dollars, and where the cheap fallback is
wired.

!!! info "Assumed background"
    [Stage internals](../loop/stages.md) covers what a completion sentinel is and why there
    are no exit codes across an `Agent()` boundary. [The Loop](../loop/index.md) covers hook
    points and the stage cycle. Neither is repeated here.

---

## 1. The five compositions

| Pattern | Participants (verified) | Where | How control returns | Cost governor |
|---|---|---|---|---|
| Producer ↔ checker bounded loop | `gsd-planner` ↔ `gsd-plan-checker`; `gsd-code-reviewer` ↔ `gsd-code-fixer` | `gsd-core/workflows/plan-phase.md:1128`; `gsd-core/workflows/quick/steps/plan-checker-loop.md:63`; `gsd-core/workflows/code-review-fix.md:249`; `gsd-core/workflows/plan-review-convergence.md:31` | `sentinel-match` both ways | **Bound differs per site** — 3 + stall detector (plan-phase); 2, no stall detector (plan-checker-loop:67); `MAX_ITERATIONS=3`, no stall detector (code-review-fix:258); `MAX_CYCLES=3` but user-settable via `--max-cycles`, with its own stall check (plan-review-convergence:382) |
| Fan-out then synthesise | 4× `gsd-project-researcher` → `gsd-research-synthesizer` → `gsd-roadmapper` | `gsd-core/workflows/new-project.md:733`, `gsd-core/workflows/new-milestone.md:340` | `sentinel-match`, plus a filesystem re-check | A filesystem re-check after fan-out — neither workflow is capability-gated, and neither contains `onError` |
| Fan-out, no merge | 4× `gsd-codebase-mapper`; N× `gsd-doc-writer` → `gsd-doc-verifier` | `gsd-core/workflows/map-codebase.md:161,187,213,239`, `gsd-core/workflows/docs-update.md:384` | `artifact+query` (each writes its own file) | Nothing to merge, so nothing to fail |
| Wave dispatch then one verifier | N× routed executor → 1× `gsd-verifier` | `gsd-core/workflows/execute-phase.md:575`–`:1289` | Neither — completion inferred from git state | Five independent degrade-to-sequential gates |
| Nested sub-orchestrator | `gsd-debug-session-manager` → `gsd-debugger` | `agents/gsd-debug-session-manager.md:97-99` | `sentinel-match`, including a non-terminal marker | `## CONTINUE_REQUIRED` instead of fabrication |

The "how control returns" column is the `Kind` field from
`gsd-core/references/agent-contracts.md`, which classifies all 35 agents into exactly three
buckets. The split is almost perfectly even — 15 `sentinel-match`, 15 `artifact+query`,
5 `structured-return` — and it correlates with the composition rather than with the agent.
Fan-out patterns skew `artifact+query` because parallel agents writing to one return channel
would interleave; loop patterns skew `sentinel-match` because a loop needs a cheap
termination test on every turn.

---

## 2. Producer ↔ checker: the only pattern with a formal bound

This is the pattern GSD invests most in, and the only one with an explicit convergence
argument. `gsd-core/workflows/plan-phase.md` §12 is titled, literally, `## 12. Revision Loop
(Max 3 Iterations)`.

```kroki-plantuml
@startuml
skinparam shadowing false
skinparam backgroundColor transparent
skinparam defaultTextAlignment center
skinparam sequenceMessageAlign center

participant "plan-phase.md\n(orchestrator)" as O
participant "researcher\ncapability-supplied\n(optional)" as R
participant "gsd-planner\nopus / opus / sonnet" as P
participant "gsd-plan-checker\nsonnet / sonnet / haiku\nno Write tool" as C

O -> O : gsd_run loop render-hooks plan:pre --raw

alt workflow.research == true
  O -> R : Agent(subagent_type=research_hook.ref.agent)
  R --> O : ## RESEARCH COMPLETE  (sentinel)
else capability off, or onError: skip
  O -> O : proceed with CONTEXT.md only
end

O -> P : Agent(subagent_type="gsd-planner")
P --> O : ## PLANNING COMPLETE  (sentinel)

loop iteration_count < 3
  O -> C : Agent(subagent_type="gsd-plan-checker")
  C --> O : ## VERIFICATION PASSED\nor ## ISSUES FOUND + YAML issues
  alt issue_count == 0
    O -> O : exit loop -> step 13
  else issue_count >= prev_issue_count
    O -> O : "Revision loop stalled"\nAskUserQuestion\nstall_reentry_count capped at 2
  else progress made
    O -> P : Agent(prompt=revision_prompt,\nrun_in_background=true)
    P --> O : ## PLANNING COMPLETE
    O -> O : gsd_stall_watch on mtimes\n(7.99 — no marker to wait on)
  end
end

O -> O : "Max iterations reached" — surface residue, do not retry
@enduml
```

Three things in that diagram are worth pulling out.

**The bound is not just a counter.** A plain max-3 retry would burn three planner passes on a
plan that is not improving. `plan-phase.md:1128` tracks `prev_issue_count` (initialised to
`Infinity`) alongside `iteration_count`, and aborts the moment `issue_count >=
prev_issue_count` — non-decreasing, not merely non-zero. The escape hatch is then itself
bounded: `stall_reentry_count` is capped at 2, after which the workflow prints `Stall
persists after 2 re-planning attempts` and offers only *Proceed anyway* or *Abandon*. Three
nested bounds, none of them open-ended.

**The same bound is hardcoded twice.** `plan-phase.md` caps at 3; `code-review-fix.md:258`
sets `MAX_ITERATIONS=3` for the `gsd-code-reviewer` ↔ `gsd-code-fixer` loop. There is no
shared constant and no config key — two independent literals that happen to agree. The
family is not uniform, though: `plan-checker-loop.md:63` bounds at **2**, and
`plan-review-convergence.md:31` defaults to 3 but exposes `--max-cycles N`, so the same
pattern carries three different bounds and one user-settable one. The
code-review loop is otherwise the more careful of the two: `#3190` at
`code-review-fix.md:255-259` tracks a `CONVERGED` flag specifically so it can decide whether
the per-iteration `.iterN.md` backups are scratch to delete or evidence to retain.

**The revision leg waits on mtimes, not on a marker.** The re-spawned planner is dispatched
with `run_in_background=true`, and the orchestrator rule immediately below it says
`(7.99; no marker, mtimes only)` — it polls `gsd_stall_watch "$TS" "{outputFile}"` rather
than blocking on a return. This is the "verify, never wait" posture applied inside a loop:
the loop's own termination test is a sentinel, but the loop's *liveness* test is the
filesystem.

### The triad is really a dyad with an optional third leg

`plan-phase.md` has a section headed `### Spawn gsd-phase-researcher` at line 347. The
`Agent()` call twenty lines below it (`plan-phase.md:371`) never mentions that agent:

```text
Agent(
  prompt=filled_research_hook_fragment,
  subagent_type=research_hook.ref.agent,
  ...
)
```

The researcher's identity is late-bound from the `plan:pre` hook set. It resolves to
`gsd-phase-researcher` only because `capabilities/research/capability.json` declares a step
at `point: "plan:pre"` with `ref.agent: "gsd-phase-researcher"`, gated on
`when: "workflow.research"` and marked `onError: "skip"`. Turn that config key off and the
researcher/planner/checker triad silently becomes a planner/checker dyad — no code path
changes, no error. The same construction repeats at `plan-phase.md:638` for the
pattern-mapper leg.

!!! warning "The section header names an agent the dispatch cannot see"
    `### Spawn gsd-phase-researcher` is accurate only for the default capability set. A
    reader grepping for `subagent_type="gsd-phase-researcher"` in `plan-phase.md` finds
    nothing and concludes the researcher is dead code; a reader trusting the header concludes
    the agent is hardcoded. Both are wrong. If you build this, name the *slot*, not the
    default occupant — `### Spawn plan:pre researcher (default: gsd-phase-researcher)`.

**When it is worth its cost.** The planner runs `opus` even on the `balanced` profile
(`gsd-core/bin/shared/model-catalog.json:134`); the checker runs `sonnet` on `golden` and
`haiku` on `budget` (`:143`, `routingTier: "light"`). The bet encoded in those two rows is
that judging a plan is a strictly cheaper cognitive task than writing one, so you can afford
to run the judge up to three times. The loop is worth its cost precisely when that asymmetry
holds — and the repo is honest that it does not always hold, which is the subject of §6.

---

## 3. Fan-out then synthesise: one agent type, N dispatches

The most-misread pattern in the codebase. It is **not** four different researcher agents. It
is one agent type — `gsd-project-researcher` — dispatched four times with different focus
areas (stack, features, architecture, pitfalls), joined by a single
`gsd-research-synthesizer`, whose `SUMMARY.md` then feeds `gsd-roadmapper`.

`gsd-core/workflows/new-project.md` writes all four dispatches out longhand at lines 782,
822, 862 and 902. `gsd-core/workflows/new-milestone.md` expresses the identical pattern in
one templated block — line 340 says "Spawn 4 parallel gsd-project-researcher agents. Each
uses this template with dimension-specific fields", with a single `Agent()` at `:377`. Same
composition, two encodings, in two files that were clearly written together. If you grep for
spawn sites you will count six in one and two in the other.

### The extra hop has a named failure mode

Adding a synthesiser buys you one merged artifact instead of four; it costs you a hop where
a fabricated refusal can lose everything. `new-project.md` around line 929 documents the
failure in unusual detail as issue #222: the synthesiser sometimes returns the entire
`SUMMARY.md` document inline while claiming "the runtime is blocking file writes", instead of
writing it. The prescribed response is not a retry — it is deterministic absorption by the
orchestrator:

1. Check `.planning/research/SUMMARY.md` exists and is substantive, via `gsd_run
   verify-summary`, reading the JSON `passed` field because *the command exits 0 regardless*.
2. If missing **and** the return text contains the template's top-level markers
   (`# Project Research Summary`, `## Key Findings`, `## Implications for Roadmap`,
   `## Sources`) — not merely the brief `## SYNTHESIS COMPLETE` confirmation — write the
   returned document to disk with the orchestrator's own Write tool and commit it.
3. If missing **and** the return is only a confirmation, stop. Do not spawn `gsd-roadmapper`.

That third branch is the load-bearing one. The whole elaborate recovery exists to guarantee
a downstream agent never runs against a truncated input. Note also that
`gsd-research-synthesizer`'s `## SYNTHESIS BLOCKED` marker is annotated in
`gsd-core/references/agent-contracts.md` as `(unconsumed: blocked-research return — spawners
detect failure via the #222 SUMMARY.md-on-disk check, no dispatch branch keys on the marker)`.
The sentinel exists, is emitted, and is deliberately not trusted: after #222, the filesystem
outranks the agent's own testimony.

**When it is worth its cost.** Only when the merge is genuinely needed. The synthesiser is
priced at `sonnet/sonnet/haiku` with `routingTier: "light"` — cheap — but the #222 machinery
around it is expensive in author-hours and orchestrator context. Which is why the third
pattern exists.

---

## 4. Fan-out with no merge step

`gsd-core/workflows/map-codebase.md` spawns four `gsd-codebase-mapper` agents with
`run_in_background=true` (`:151`) at lines 161, 187, 213 and 239 — and then does not
synthesise. Each mapper writes its own file under `.planning/codebase/`, and the orchestrator
verifies completion with `ls`/`wc -l`, per `agent-contracts.md`'s row: *"No marker (writes
docs directly)"*, Kind `artifact+query`.

`gsd-core/workflows/docs-update.md` is the same pattern at larger scale, structured into
waves: three `gsd-doc-writer` agents for wave 1 (`:384` — "foundational docs with no
cross-references needed, making them ideal for parallelisation"), more in wave 2, all with
`run_in_background=true`. Verification is a separate per-document pass by `gsd-doc-verifier`,
which writes `.planning/tmp/verify-{doc_filename}.json` — one JSON per doc, again no merge.

The distinction matters more than it looks. Fan-out-then-synthesise has a join point, and a
join point is a single point of failure that needs a #222-class recovery. Fan-out-with-no-merge
has N independent outputs and N independent failures, each detectable by a file existence
check. **Partitioning the output space is cheaper than merging it**, and the four mappers can
do that because the codebase decomposes cleanly into four documents while project research
does not decompose into four roadmaps.

The prior-art doc `docs/explanation/multi-agent-orchestration.md` describes the mapper case
correctly as "4 parallel sub-probes" — and then, four rows later in the same table, describes
the researcher case as four differently-named agents, asserting that "the four researchers in
a `plan-phase` run simultaneously, not sequentially." `plan-phase.md` spawns exactly one
researcher, and it is a hook-supplied slot. The correct reading of the pattern and the
incorrect one sit in the same table.

---

## 5. Wave dispatch: the pattern with the most guards and the fewest sentinels

Execution is the only pattern where parallel agents write to a **shared** output — the git
repository. Consequently it carries more safety machinery than the other four combined, and
it abandons sentinel-matching entirely.

### The dispatched agent is a slot, not an agent

`gsd-core/workflows/execute-phase.md` dispatches `subagent_type="{EXECUTOR_TYPE}"`, resolved
per plan by `gsd-core/workflows/execute-phase/steps/per-plan-executor-routing.md` (#1689). A
plan opts into a specialist by declaring `agent_hint:` in its PLAN.md frontmatter; the
orchestrator calls `gsd_run query resolve-agent --name "$PLAN_HINT" --raw` and **fails closed
to `gsd-executor`** when the name does not resolve on the active runtime. That step also
prints a warning rather than silently ignoring a hint that cannot be honoured, because the
`orchestrator-worktree` isolation backend spawns executors as processes with no
`subagent_type` at all.

### Completion is inferred from git, not declared by the agent

`gsd-executor` declares `## PLAN COMPLETE` as a `sentinel-match` marker. `execute-phase.md`
does not match it. `agent-contracts.md` closes with a NOTE recording that the workflow detects
completion by SUMMARY.md existence and git commit state instead, and calls this "intentional
behavior, not a mismatch". Stall detection (#3212, `execute-phase.md:845`) works the same way:
every N minutes the orchestrator runs `git log "$EXPECTED_BRANCH" --since="$DISPATCH_TS"` and,
on threshold, offers continue-waiting / kill-and-retry / kill-and-switch-to-inline.

The reason is structural. `Agent()` backgrounds by default, and under parallel dispatch the
completion signal may never route back — `plan-phase.md:351` even warns that
`CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS` breaks this routing, because "a subagent's completion
can fail to route to the orchestrator" (issue #1355). An observable side
effect in a shared repository is a more reliable completion oracle than a string the agent
promises to print.

### Five independent gates, each degrading to sequential

| Gate | Where | Degrades |
|---|---|---|
| Intra-wave `files_modified` overlap | `execute-phase.md:575-604` | `PARALLELIZATION=false` for that wave |
| Per-plan submodule gate (#2474) | `gsd-core/workflows/execute-phase.md:635` → `gsd-core/workflows/execute-phase/steps/per-plan-worktree-gate.md` | `USE_WORKTREES_FOR_PLAN=false` |
| Fork-base divergence (#683) | `execute-phase.md:130` | `USE_WORKTREES` / `ISOLATION` for the whole run |
| Stall surveillance (#3212) | `execute-phase.md:845-857` | User-chosen kill-and-switch-to-inline |
| Copilot forced-sequential | `execute-phase.md:160` | `COPILOT_SEQUENTIAL=true` overrides `parallelization` |

The overlap check is the most instructive. It runs *before* any spawn, compares every pair of
plans in the wave, and when two share even one file it not only serialises the wave but
reports the condition as a **planning defect** — "The planner should have caught this; flag it
as a planning defect so the user can replan the phase if desired"
(`execute-phase.md:604`). The runtime guard does not silently paper over the upstream
mistake; it attributes it. Every one of the five fails toward slower-but-correct.

### One verifier per phase, not per wave

`gsd-verifier` is spawned once, at `execute-phase.md:1289`, after the wave loop has finished
— not once per wave. So this is not a triad in the sense the plan loop is; it is
fan-out-then-single-verify at phase granularity. The verifier is `artifact+query`
(`*-VERIFICATION.md` plus `gsd_run query verification.status`), and its own `## Verification
Complete` marker is an audited orphan: `agent-contracts.md` Marker Rule 2 records it as an
intentional title-case marker that nothing matches, carried as an explicit `(unconsumed: …)`
annotation rather than deleted.

### State ownership flips with the isolation mode

The wave pattern solves the distributed-write problem by removing the distribution. In
worktree mode, the executor prompt states: *"Do NOT update STATE.md or ROADMAP.md — the
orchestrator owns those writes after all worktree agents in the wave complete"*, and
`gsd-core/workflows/execute-plan.md` auto-detects worktree mode (`.git` is a file, not a
directory) to skip shared-file updates. The *same agent definition* in sequential mode gets
`success_criteria` that **require** STATE.md and ROADMAP.md updates. One `agents/gsd-executor.md`,
two mutually exclusive success contracts, selected by dispatch mode.

---

## 6. Nested sub-orchestration: the only agent that spawns an agent

Across all 35 definitions there is exactly one outbound dispatch from one agent to another:
`agents/gsd-debug-session-manager.md:99` spawning `gsd-debugger`. Its own frontmatter
describes the job — *"Manages multi-cycle /gsd:debug checkpoint and continuation loop in
isolated context. Spawns gsd-debugger agents... Returns compact summary to main context."*
Two other agent files mention a dispatch primitive, but both are describing how *they* are
spawned, not spawning anything: `agents/gsd-advisor-researcher.md:11` and
`agents/gsd-assumptions-analyzer.md:11` each read "Spawned by ... via `Task()`" — a different
spelling of the call every workflow writes as `Agent(`. The framework that
machine-enforces its return contracts (§9) has never enforced the name of its own dispatch
primitive.

This is context-window management expressed as topology. A debug session is an unbounded
number of investigate/checkpoint cycles; running it in the main orchestrator's context would
exhaust the window and take the whole workflow down. Moving the loop one level down means the
main context sees one dispatch and one compact summary, regardless of how many cycles ran.

What makes it worth studying is the honesty rule at `agents/gsd-debug-session-manager.md:309`.
A sub-orchestrator that runs out of budget mid-loop is under structural pressure to
manufacture a terminal result, because its caller is waiting for one. The definition names
that pressure and forbids it:

> **Non-terminal early stop — check this FIRST.** Before returning any summary below, ask: is
> your own turn/context budget exhausted while the debugger (`gsd-debugger`) is still
> investigating [...] If so, do NOT fabricate a `DEBUG SESSION COMPLETE` or `ABANDONED`
> summary to fit this shape. Return the non-terminal marker instead.

The marker is `## CONTINUE_REQUIRED`, declared in `agent-contracts.md` alongside `## DEBUG
SESSION COMPLETE` and consumed by `gsd-core/workflows/debug.md`. **A nested orchestrator needs
a "not done, not failed" return value in its type**, or exhaustion will be encoded as success.
This is the single most transferable idea on this page.

---

## 7. What each pattern actually costs

Model assignment is not declared on agents. Exactly one of 35 definitions pins a model
(`agents/gsd-mempalace-curator.md:5`, `model: sonnet`); every other spawn resolves at dispatch
via `gsd_run query resolve-model <agent>`, backed by the committed table at
`gsd-core/bin/shared/model-catalog.json`. That centralisation is what makes the cost structure
of each pattern legible in one file.

| Agent | golden | balanced | budget | routingTier |
|---|---|---|---|---|
| `gsd-planner` | opus | opus | sonnet | heavy |
| `gsd-executor` | opus | sonnet | sonnet | standard |
| `gsd-plan-checker` | sonnet | sonnet | haiku | light |
| `gsd-verifier` | sonnet | sonnet | haiku | standard |
| `gsd-codebase-mapper` | sonnet | haiku | haiku | light |
| `gsd-research-synthesizer` | sonnet | sonnet | haiku | light |
| `gsd-security-auditor` | **opus** | sonnet | sonnet | **heavy** |
| `gsd-code-reviewer` | **opus** | sonnet | sonnet | standard |

The received wisdom — *checkers are priced below producers* — holds for the two agents it is
usually cited about. `gsd-plan-checker` sits a full tier under `gsd-planner` on every profile,
and `routingTier: "light"` versus `"heavy"` compounds it: `src/model-resolver.cts:803,865`
uses `routingTier` for `routing_tier_defaults` on effort and fast-mode, a second independent
axis from model choice.

But the rule breaks, and it breaks informatively. `gsd-security-auditor` is `golden: opus,
routingTier: heavy` — priced identically to the planner. `gsd-code-reviewer` is `golden: opus`,
level with the `gsd-code-fixer` it reviews. So the discount does not track the checker *role*;
it tracks **how much adversarial depth the check requires**. Validating that a plan has the
required sections is a light task. Finding a threat mitigation that was specified but not
implemented, in code you did not write, is not.

!!! note "The naming carries no information — twice"
    The `-checker` / `-auditor` / `-verifier` suffixes predict neither price nor tool grants.
    On price: `gsd-security-auditor` is opus/heavy while `gsd-ui-auditor` is sonnet/light. On
    tools: `gsd-plan-checker`, `gsd-ui-checker`, `gsd-integration-checker` and
    `gsd-security-auditor` have **no Write tool** and return verdicts; `gsd-verifier`,
    `gsd-ui-auditor`, `gsd-eval-auditor`, `gsd-doc-verifier`, `gsd-dom-verifier` and
    `gsd-nyquist-auditor` **do** have Write and produce reports. Both splits are deliberate
    and recorded; neither is visible in the name. Name your agents after their return shape,
    not their attitude.

---

## 8. Tool grants and return contracts: necessary, not sufficient

`agent-contracts.md` Rule 4 defines `structured-return` as *"the agent has no way to write
files (no `Write` tool) or simply doesn't"*. Cross-tabulating the frontmatter `tools:` line of
all 35 agents against their declared `Kind` gives a clean result in one direction and a hole
in the other:

| | `sentinel-match` | `artifact+query` | `structured-return` |
|---|---|---|---|
| **has Write** | 12 | 14 | 0 |
| **no Write** | 3 | **1** | 5 |

Every one of the 5 `structured-return` agents lacks Write — the rule holds. The converse
fails: 4 more agents lack Write without being `structured-return`. Three of those are benign
(`gsd-plan-checker`, `gsd-ui-checker`, `gsd-security-auditor` return sentinels, which needs no
filesystem — and `agents/gsd-security-auditor.md:93` says so explicitly: *"the orchestrator
writes SECURITY.md from this data — the auditor does NOT write any files (#2119)"*).

The fourth is a defect.

!!! danger "`gsd-mempalace-curator` is contracted to write artifacts it cannot write"
    `agents/gsd-mempalace-curator.md:4` grants `tools: Read, Bash, Grep, Glob`. No Write. No
    `mcp__*`. Yet `gsd-core/references/agent-contracts.md` classifies it `artifact+query` —
    *"No marker (writes the session diary + cross-links)"* — and `gsd-core/workflows/ship.md:536`
    dispatches it by name at `ship:post`. The contract asserts an artifact the grant forbids.

    It compounds: the agent's own step 1 instructs it to call
    `mempalace_diary_write(agent_name=..., entry=..., topic="phase-ship", wing=...)` and
    `mempalace_diary_read` — MCP tool forms it has no grant for. Compare
    `agents/gsd-dom-verifier.md`, which explicitly declares `mcp__chrome-devtools__*,
    mcp__claude-in-chrome__*` when it needs them. No `capability.json` under `capabilities/`
    declares a `tools` key at all, so a pack cannot supply the grant either — grants can only
    be narrowed at the agent definition, never widened.

    It does not fail loudly for two reasons. The instruction carries a parenthetical CLI
    fallback (`mempalace hook run`) reachable through Bash, and the capability step is
    `onError: skip`, so a silent no-op looks like a clean run.

---

## 9. Where the machine enforcement actually stops

The headline claim about GSD's agent layer is that return contracts are machine-enforced, not
conventional. Marker Rule 6 in `agent-contracts.md` states that the `Consumed by` / `Kind`
columns are cross-checked by `scripts/check-contract-drift.cjs` against *"what every
`gsd-core/workflows/**`, `commands/**`, and `agents/**` file actually consumes"*, wired into
`npm run lint:ci` via `package.json:122`. That is true, and it is a good design: an
agent's return protocol is an API, and API drift becomes a build failure rather than a review
comment. The checker catches `case_collision`, `agent_without_contract`,
`duplicate_registry_row`, `unknown_producer`, `case_only_match` and `vestigial_marker`.

It is also more careful than a first reading of the helper's doc comment suggests. That comment
(`scripts/command-contract-helpers.cjs:452-455`) describes only the corpus-wide test —

> for sentinel-match rows specifically — every declared marker must have at least one
> EXACT-CASE consumer **somewhere in `consumerTexts`**, excluding the producing agent's own
> file

— but the implementation at `scripts/command-contract-helpers.cjs:628-685` adds a second arm the
comment omits. Alongside `no_consumer` and `case_only_match` there is
`declared_consumer_no_match`, which fires when a marker "IS consumed somewhere, but by none of
the consumers the row cites". The inline comment there is explicit about why: the `Consumed by`
cell "is documentation the read-tag arm and humans both navigate by, so a cell naming files that
never match is registry drift even when the corpus at large happens to contain the token." The
column is checked. Running the gate confirms the state of the tree:

```console
$ node scripts/check-contract-drift.cjs
ok check-contract-drift: 35 agents, 38 markers, 0 violations
```

The residual gap is narrower, and it is a quantifier. `declaredConsumerMatch` is set by **any**
one entry in the cell (`:648`); the violation fires only when **none** match. The test is
existential over the cited set, not universal — so a single correct entry vouches for every
incorrect one beside it.

That is not hypothetical either. `gsd-security-auditor`'s row cites
`gsd-core/workflows/secure-phase.md`, `gsd-core/workflows/validate-phase.md` and
`agents/gsd-nyquist-auditor.md`. The third contains no occurrence of `OPEN_THREATS` or
`## SECURED` at all. The row passes clean because `secure-phase.md` contains `OPEN_THREATS`
twice, and one match satisfies the whole cell. Given there is exactly **one** agent-to-agent
dispatch edge in the repo (§6), the agent-file entries in that column generally record shared
vocabulary between sibling auditors rather than any wiring.

The producer-exclusion rule has the same shape of hole. `## ISSUES FOUND` is declared by
**both** `gsd-plan-checker` and `gsd-ui-checker`. Exclusion is by filename —
`if (consumerId === 'agents/' + agent + '.md') continue` (`:644`) — so when checking
`gsd-ui-checker`'s row, `agents/gsd-plan-checker.md` is not the producer and its three
exact-case occurrences of `## ISSUES FOUND` count as consumption. All three are
`gsd-plan-checker` *emitting* its own identically-named marker. Nothing is broken here, because
`plan-phase.md` (5 occurrences) and `gsd-core/workflows/ui-phase.md` (1) legitimately consume
it — but the
mechanism cannot distinguish a consumer from a co-producer of the same string.

!!! tip "The generalisable lesson"
    `check-contract-drift.cjs` enforces **vocabulary reachability** and does it well: no marker
    declared-but-unreachable, no case collisions, no agent without a row, no cell whose entries
    all miss. It cannot enforce the **call graph**, because a shared marker vocabulary makes
    "file contains string" ambiguous between emitting and consuming, and because one good cell
    entry covers for the rest. If you build this, either narrow the check to per-entry
    (universal, not existential) or accept that the column is navigational documentation and say
    so. The failure mode worth avoiding is a column that reads as machine-verified while a stale
    entry beside a fresh one goes unreported — which is the state of that
    `agents/gsd-nyquist-auditor.md` reference today.

---

## 10. Reconstructing this layer

If you are building an equivalent framework, the pieces worth copying, in order of value:

1. **Give every nested orchestrator a non-terminal return value.** `## CONTINUE_REQUIRED`
   plus an explicit "do NOT fabricate a completion to fit this shape" instruction
   (`agents/gsd-debug-session-manager.md:309`). Without it, budget exhaustion is silently
   encoded as success, and you will not find out for weeks.
2. **Bound loops on progress, not just count.** `iteration_count < 3` is table stakes;
   `issue_count >= prev_issue_count` is what stops you paying an opus planner to not improve.
3. **Prefer partitioned fan-out over fan-out-plus-merge** when the output space decomposes.
   Four mappers writing four files need no join and no #222-class recovery. Add a synthesiser
   only when a single merged artifact is genuinely required downstream.
4. **Infer completion from observable side effects when the channel is unreliable.** Under
   backgrounded parallel dispatch, `git log --since="$DISPATCH_TS"` on the expected branch beats
   waiting for a sentinel that may never route.
5. **Make every parallel path independently opt-out-able, each failing toward sequential.**
   Five orthogonal gates is not over-engineering when the shared resource is a git repository.
6. **Centralise model assignment in committed data.** One `model-catalog.json` makes the cost
   shape of every pattern readable in one place; 35 agents each pinning their own model does
   not.
7. **Name the slot, not the occupant** — in section headers, in agent names, and in dispatch
   sites. `research_hook.ref.agent`, `{EXECUTOR_TYPE}`, `resolve-agent --name` are all the
   right shape. The `### Spawn gsd-phase-researcher` header above one of them is not.
