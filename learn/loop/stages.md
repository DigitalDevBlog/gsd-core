# Stage internals

A GSD Core stage — bootstrap, plan, execute, verify — is not a function. It is a pair of
files: a small **dispatcher** that a runtime can register as a command or a skill, and a
large **workflow spec** that the dispatcher pulls into context by reference. The dispatcher
declares argument semantics and tool grants. The spec contains all of the procedure.

This page is about how those two files are constructed, what grammar the spec is written in,
how one stage's output becomes the next stage's input, and what tricks make a 1,500-line
markdown document executable by a language model with no interpreter underneath it.

---

## 1. Where a stage spec lives

Every stage exists twice in this repo, and only one copy is authored.

| Stage | Authored source | Shipped artifact |
|-------|-----------------|------------------|
| Bootstrap | `commands/gsd/new-project.md` | `skills/gsd-new-project/SKILL.md` |
| Plan | `commands/gsd/plan-phase.md` | `skills/gsd-plan-phase/SKILL.md` |
| Execute | `commands/gsd/execute-phase.md` | `skills/gsd-execute-phase/SKILL.md` |
| Verify | `commands/gsd/verify-work.md` | `skills/gsd-verify-work/SKILL.md` |

The `skills/` copy is a build output. `scripts/gen-plugin-skills.cjs` reads every
`commands/gsd/*.md`, runs it through `convertClaudeCommandToClaudeSkill`
(`src/runtime-artifact-conversion.cts:452`), and writes the result — deleting and recreating
the whole `skills/` tree each time (`scripts/gen-plugin-skills.cjs:66-72`). The same script's
`--check` mode re-runs the conversion in memory and byte-compares it against every committed
`SKILL.md`, counting missing files, mismatched files and orphaned `gsd-*` directories with no
source command as stale (`scripts/gen-plugin-skills.cjs:81-103`).

The delta is small and entirely mechanical. Diffing each authored command against its
generated skill produces three kinds of change, two of them intended:

**Frontmatter is rebuilt, not edited.** The converter emits fields one at a time in a fixed
order — `name`, `description`, optional `version`/`priority`, `argument-hint`, `agent`,
`context`, then the `allowed-tools` block copied verbatim
(`src/runtime-artifact-conversion.cts:464-517`). Anything not on that list is gone. The
authored `commands/gsd/plan-phase.md` carries `effort: max` and
`requires: [discuss-phase, phase, review, update]`; neither survives into
`skills/gsd-plan-phase/SKILL.md`. `name: gsd:plan-phase` becomes `name: gsd-plan-phase`.

**The body is copied, with one rewrite.** Every `/gsd:<cmd>` reference becomes `/gsd-<cmd>`
so body text matches the hyphenated `name:` the runtime registers
(`src/runtime-artifact-conversion.cts:456-462`). Across all four stages that is three
one-line hunks in total: `/gsd:review` → `/gsd-review` in plan-phase, `/gsd:plan-phase` →
`/gsd-plan-phase` in new-project, `/gsd:execute-phase` → `/gsd-execute-phase` in verify-work.
`commands/gsd/execute-phase.md` has no semantic body change at all.

**And one fingerprint nobody authored.** Every generated skill also gains a blank line
directly under its frontmatter. The converter returns `` `${fm}\n${normalizedBody}` ``
(`src/runtime-artifact-conversion.cts:520`) where `fm` already ends in `---` (`:518`) and the
extracted body already carries its own leading newline, so the join adds one. It is harmless,
but it is worth noticing: even a path whose job is verbatim copying leaves a trace, which is
why `--check` is a byte comparison rather than a semantic one.

!!! note "Why the whitelist matters when you build your own"
    Rebuilding frontmatter from a fixed field list — rather than deleting known-bad keys —
    makes the authored surface a deliberate superset of the shipped one. `requires:` can
    exist purely so `scripts/lint-skill-deps.cjs` can prove that no command body references a
    command missing from its `requires:` list, and that every install profile's dependency
    closure is satisfied (`scripts/lint-skill-deps.cjs:1-16`). The graph is validated at
    build time and then discarded. There is no resolver at runtime, so there is nothing for
    a runtime resolver to get wrong. The default for any new authored key is "does not
    ship", which is the safe default when you target many runtimes.

`effort:` is the one omission the source argues for explicitly, in two separate comments
(`src/runtime-artifact-conversion.cts:468` and `:513`): a static effort value in skill
frontmatter changes `output_config.effort` when the skill is invoked, which invalidates the
*caller's* prompt cache. Effort is kept where the caller owns the turn — the command — and
dropped where the stage is invoked inside somebody else's context.

For the rest of the generation pipeline — the install-time include rewriting, the other
runtime converters — see [Skills](../skills/index.md) and [Runtime](../runtime/index.md).
From here on, "the stage spec" means the authored command plus the workflow it loads.

---

## 2. The dispatcher is a routing header, not a procedure

Here is the entire `<process>` section of the execute stage, the most complex of the four:

```text
Execute end-to-end.
Preserve all workflow gates (wave execution, checkpoint handling, verification, state
updates, routing).
```

That is `skills/gsd-execute-phase/SKILL.md:61-64`, against a `gsd-core/workflows/execute-phase.md`
of 1,532 lines. The dispatcher does four things and nothing else.

**It loads the spec declaratively.** `<execution_context>` is a list of bare `@`-imports, one
per line — `skills/gsd-execute-phase/SKILL.md:34-37` pulls the workflow plus a brand
reference; `skills/gsd-new-project/SKILL.md:37-41` pulls five files, including two output
templates. That list is a checked edge: `scripts/lint-command-contract.cjs` verifies every
`execution_context` `@`-reference resolves to a file that exists on disk, that each sits alone
on its own line with no trailing prose, and — separately — that every file in
`gsd-core/workflows/` is reachable from at least one loader
(`scripts/lint-command-contract.cjs:3-16`). Unreachable spec files are a lint failure, not a
silent orphan.

**It states argument semantics defensively.** Execute spends roughly ten lines drawing a line
between a documented flag and an active one: "A flag is active only when its literal token
appears in `$ARGUMENTS`" (`skills/gsd-execute-phase/SKILL.md:26-29`), then repeats the rule
per flag and closes with "Do not infer that a flag is active just because it is documented in
this prompt" (`:51-56`). Flag prose outruns the `<process>` stanza about five to one. This is
a prompt defending against a known model failure mode — over-applying options it can see —
and treating argument parsing as a correctness surface rather than documentation.

**It names the gates.** Since `<process>` delegates everything, the parenthesised gate list is
the dispatcher's only semantic payload: the one assertion about what a fresh-context model
must not shortcut in the spec it has just loaded. The four stages carry four different lists —
`validation, approvals, commits, routing` (new-project), `validation, research, planning,
verification loop, routing` (plan-phase), `wave execution, checkpoint handling, verification,
state updates, routing` (execute-phase), `session management, test presentation, diagnosis,
fix planning, routing` (verify-work). These are hand-specialized, not boilerplate.

**It grants tools.** `allowed-tools` is authored per command and copied through unchanged
(`src/runtime-artifact-conversion.cts:476-482`). The grants differ sharply and deliberately:
bootstrap gets five tools; plan gets nine including `WebFetch` and `mcp__context7__*`; execute
is the only one of the four granted `TodoWrite`; verify is the only one *without*
`AskUserQuestion` — which is not an oversight but a match to its interaction contract
(§4.4).

One more thing lives in the dispatcher: portability. `<runtime_note>` appears on three of the
four stages and tells the model to substitute `vscode_askquestions` for `AskUserQuestion` under
Copilot. Plan-phase adds a stronger clause — "Do not skip questioning steps because
`AskUserQuestion` appears unavailable" (`skills/gsd-plan-phase/SKILL.md:38`). Portability here
is inline prose substitution, not an abstraction layer over a question API. It is cheaper, and
it survives verbatim body copying into every runtime's skill format.

!!! warning "The guard proves transformation, not truth"
    `gen-plugin-skills.cjs --check` can only prove that `skills/` matches `commands/`. It says
    nothing about whether the prose inside the command is accurate — and that is where all the
    observable drift has landed. Three of the four dispatchers describe the CLI three different
    ways: `gsd-tools query init.execute-phase` (`skills/gsd-execute-phase/SKILL.md:58`),
    `init verify-work` (`skills/gsd-verify-work/SKILL.md:33`), and `gsd-tools.cjs`
    (`skills/gsd-plan-phase/SKILL.md:42`). What the workflows actually run is
    `gsd_run query init.<workflow>` (`gsd-core/workflows/verify-work.md:45`,
    `gsd-core/workflows/execute-phase.md:85`). Drift migrates to whichever layer has no
    generator pointed at it. Docstrings drift too: the header comment at
    `scripts/lint-command-contract.cjs:5` still names a command count that no longer matches the
    directory the script actually globs (`:146-147`).

---

## 3. The shared grammar of a workflow spec

Workflows are XML-ish markdown. There is no schema and no parser — the tags exist to give the
model unambiguous scope boundaries and to make sections addressable in prose ("see
`<runtime_compatibility>`"). Across the four stages the recurring vocabulary is:

| Tag | Role |
|-----|------|
| `<purpose>` | One paragraph; the stage's contract in prose |
| `<required_reading>` | Files to load before doing anything |
| `<available_agent_types>` | Exact subagent names, with an explicit ban on falling back to `general-purpose` |
| `<runtime_compatibility>` | What changes per host, and what may never change |
| `<process>` | The steps |
| `<success_criteria>` | A terminal checklist |

The interesting property is that **the grammar is recursive**. When a workflow constructs a
subagent prompt, it writes that prompt in the same tag vocabulary. The research spawn at
`gsd-core/workflows/new-project.md:755-815` embeds `<required_reading>`, `<downstream_consumer>`,
`<quality_gate>` and `<output>` inside an `Agent(prompt="…")` call. `<output>` names an exact
destination path *and* the template to fill:

```text
<output>
Write to: {research_dir}/STACK.md
Use template: ~/.claude/gsd-core/templates/research-project/STACK.md
</output>
```

`<downstream_consumer>` is the sharpest of these. It states, inside the *producing* prompt,
who will read the artifact and what they need from it. In bootstrap it is a style directive
("Your STACK.md feeds into roadmap creation. Be prescriptive", `:761`). In plan-phase it is a
full structural contract on PLAN.md — required frontmatter fields, the mandatory
`<read_first>` and `<acceptance_criteria>` on every task, and precise rules for lifting spec
sections into `must_haves` including a flat-scalar marker shape the verifier branches on
(`gsd-core/workflows/plan-phase.md:810`). Notice who the consumer is there: not the next
stage, but the verifier two stages later. The handoff contract is authored where the artifact
is produced.

**Step notation is not standardized.** Bootstrap and plan use numbered markdown headings
(`## 1. Setup`, 13 and 37 of them respectively). Execute and verify use `<step name="…">`
elements (18 and 17). Both conventions live in the same tree and are read by the same model;
nothing in the repo's linters normalizes them. If you are building this, pick one — the cost
of two is a reader (human or model) who cannot pattern-match step boundaries across specs.

---

## 4. What is distinctive per stage

### 4.1 Bootstrap — outside the loop it creates

Five workflows carry a `<!-- gsd:loop-host -->` header, the machine-readable declaration of
where capabilities may attach: `discuss-phase.md`, `plan-phase.md`, `execute-phase.md`,
`verify-work.md`, `ship.md`. `gsd-core/workflows/new-project.md` carries none. Bootstrap is
structurally not a loop stage — it runs once, to produce the ROADMAP.md the loop then
iterates. Its dispatcher is also the only one of the four whose sections run in a different
order (`<runtime_note>` → `<context>` → `<objective>` → `<execution_context>` → `<process>`,
`skills/gsd-new-project/SKILL.md:13-46`), which is more evidence that section order here is a
habit rather than a template.

It has the widest `@`-import fan-out of the four — five files, including
`gsd-core/templates/project.md` and `gsd-core/templates/requirements.md` pulled in *at the dispatcher*
(`skills/gsd-new-project/SKILL.md:37-41`). That placement is a real decision: bootstrap's
output *is* the artifact set, so the shape of the artifacts must be in context from the first
turn rather than fetched mid-run.

Its `<process>` opens with a hard ordering constraint — "MANDATORY FIRST STEP — Execute these
checks before ANY user interaction" (`gsd-core/workflows/new-project.md:28`) — followed by the
init call and a literal enumeration of the JSON keys to parse, including the greenfield /
brownfield discriminators `has_existing_code`, `is_brownfield`, `needs_codebase_map`. The
branch that decides what kind of project this is happens in the CLI; the workflow only reads
the answer.

### 4.2 Plan — a revision loop that is a real state machine

```text
step: plan
points: plan:pre, plan:post
agent-roles: researcher, planner, checker
produces: PLAN.md
consumes: CONTEXT.md
```

That is `gsd-core/workflows/plan-phase.md:1-7`. The stage orchestrates three subagents and a
convergence loop with genuinely non-trivial state:

- `iteration_count` starts at 1 after the initial plan-and-check pass; while it is `< 3` the
  loop re-spawns the planner with the checker's issues (`:1130`, `:1134`).
- **Stall detection** is separate from the iteration cap: if the issue count is not
  *decreasing* between iterations, the loop reports "Revision loop stalled — issue count not
  decreasing" and stops re-spawning blindly (`:1141`).
- On a stall the user gets an interactive branch (adjust approach / proceed / stop). Choosing
  "Adjust approach" re-enters full replanning and increments `stall_reentry_count`.
- The counters have **asymmetric lifetimes**, stated explicitly in prose: "re-entry resets
  `iteration_count` and `prev_issue_count` but `stall_reentry_count` persists across
  re-entries and is capped at 2" (`:1148`). Without that asymmetry, adjusting the approach
  would reset the whole budget and the loop could never terminate.
- At `iteration_count >= 3` it stops and shows the remaining issues (`:1198-1200`); a planner
  that returns nothing usable terminates in `## PLANNING INCONCLUSIVE` (`:896`).

`<runtime_compatibility>` (`:30-56`) forbids the orchestrator from absorbing any of the three
roles inline, on any runtime, and explains why: "Independent agent contexts are required for
the plan-checker gate to be meaningful." A checker sharing the planner's context is not a
gate, it is a rubber stamp. The same block bans introspection — never decide the Agent tool is
unavailable by self-assessment, only by an actual tool-unavailable error from a real call.

One detail worth stealing: the workflow states plainly where the CLI stops and the model
starts. "`query init.plan-phase` and `query roadmap.get-phase` require an explicit number, so
this is an orchestrator step" (`:180`) — the model runs `query roadmap.analyze`, reads
`next_phase`, and falls back to asking the user. Deterministic work is pushed into the CLI;
ambiguity is explicitly handed back to the model, and the boundary is documented at the point
of use.

### 4.3 Execute — the stage with an inner loop

```text
step: execute
points: execute:pre, execute:wave:pre, execute:wave:post, execute:post
agent-roles: executor, verifier
produces: SUMMARY.md
consumes: PLAN.md
```

Four attach points instead of two (`gsd-core/workflows/execute-phase.md:1-7`) — the only stage
whose iteration is itself an extension surface. `<core_principle>` states the shape in one
line: "Orchestrator coordinates, not executes… discover plans → analyze deps → group waves →
spawn agents → handle checkpoints → collect results" (`:12-14`), which the dispatcher restates
as a budget: "Context budget: ~15% orchestrator, 100% fresh per subagent"
(`skills/gsd-execute-phase/SKILL.md:31`).

Context sizing is a **three-band policy** (`gsd-core/workflows/execute-phase.md:135-148`):

| `context_window` | Behaviour |
|---|---|
| `>= 500000` | Executors also receive prior-wave SUMMARY.md, phase CONTEXT.md and RESEARCH.md; verifiers receive all PLAN/SUMMARY/CONTEXT files plus REQUIREMENTS.md |
| default (implicit — the shell default is `200000` when `config-get` returns nothing) | Standard prompts |
| `< 200000` | Extended examples and edge-case lists are **removed** from the inline prompt and replaced with on-demand `@`-references to `gsd-core/references/executor-examples.md` and `gsd-core/references/planner-antipatterns.md` — "~40% less executor static overhead while preserving behavioral correctness" |

The thinning direction is the one most frameworks forget: the same prompt has to survive a
small window, and the answer is to demote examples to lazy references while keeping the
decision logic inline. It is also the band that is easiest to lose track of —
`docs/ARCHITECTURE.md:539` describes only two regimes, enrichment above 500,000 and
"truncated versions" for standard 200K windows, with no mention of the sub-200K band or the
reference files it swaps in.

Execute also carries the least inline procedure of the four relative to its complexity,
because eleven step files live beside it under `gsd-core/workflows/execute-phase/steps/` and
are read only when reached — `per-plan-worktree-gate.md` per plan (`:637`),
`per-plan-executor-routing.md` per plan (`:658`), `post-merge-gate.md` after merge (`:971`),
`wave-post-gate-hooks.md` per active gate (`:1021`). Progressive disclosure implemented as a
directory.

Three of those reads are additionally gated by section markers:

```text
<!-- gsd:section id="partial-wave" when="flag:--wave" -->
If `section_manifest` is `null` or `"partial-wave"` is in its `included` list: read and
execute `gsd-core/workflows/execute-phase/steps/partial-wave.md`. Otherwise skip — do not
read the file.
<!-- /gsd:section -->
```

(`:1194-1196`; siblings at `:1251` keyed on `state:gap-closure-phase` and `:1255` on
`state:has-prior-phases`.) The marker is machine-readable metadata for a manifest generator;
the sentence beneath it is the runtime behaviour, written for a model. Both are needed
because an `@`-import cannot be conditional — a static include behind a condition still loads
eagerly. Conditionality has to be expressed as an instruction to *not read a file*.

### 4.4 Verify — a conversation with a state machine underneath

```text
step: verify
points: verify:pre, verify:post
agent-roles: orchestrator
produces: UAT.md
consumes: SUMMARY.md
```

No subagent roles for the main path (`gsd-core/workflows/verify-work.md:1-7`); the spawns it
does declare — `gsd-planner`, `gsd-plan-checker` — are for the *repair* path. The stage's
interaction contract lives in a `<philosophy>` block (`:20`):

> **Show expected, ask if reality matches.** Claude presents what SHOULD happen. User confirms
> or describes what's different. — "yes" / "y" / "next" / empty → pass. Anything else →
> logged as issue, severity inferred. No Pass/Fail buttons. No severity questions.

That is why this is the one stage of the four whose `allowed-tools` omits `AskUserQuestion`:
structured pickers would contradict the design. `<severity_inference>` (`:910`) then supplies a lookup
table from user phrasing to severity, with a default of `major` and an explicit prohibition:
"Never ask 'how severe is this?' — just infer and move on." (`:922`)

Writes are batched rather than continuous — "on issue, every 5 passes, or completion"
(`<success_criteria>`; rules in `<update_rules>`, `:890`) — so a long UAT session survives a `/clear` without paying a file write
per turn.

Its final criterion is the loop's back edge: "Ready for `/gsd:execute-phase --gaps-only` when
complete." And `--gaps-only` is defined on the other side as "Execute only gap closure plans
(plans with `gap_closure: true` in frontmatter)"
(`skills/gsd-execute-phase/SKILL.md:48`). Failure does not exit the loop; it re-enters
execute with a filter. That is the mechanism behind calling this a loop rather than a
pipeline.

---

## 5. How state actually moves

Not through variables. The `gsd:loop-host` headers declare a file chain, and every consumer
re-derives what it needs from disk.

| Stage | consumes | produces | Written state (from the four workflows' `query state.*` / `query roadmap.*` calls) |
|-------|----------|----------|-----------------------------------------------------------------------------------|
| discuss | — | `CONTEXT.md` | — |
| bootstrap | *(outside the loop)* | `PROJECT.md`, `REQUIREMENTS.md`, `ROADMAP.md`, `STATE.md`, `config.json` | — |
| plan | `CONTEXT.md` | `PLAN.md` | `query state.planned-phase --phase … --name … --plans …` (`plan-phase.md:1393`) |
| execute | `PLAN.md` | `SUMMARY.md` | `query state.begin-phase` (`execute-phase.md:325`), `query roadmap.update-plan-progress … complete` (`:987`), `query state.update last_gate_trip` (`:204`), `query phase.complete` (`:1407`) |
| verify | `SUMMARY.md` | `UAT.md` | — |
| ship | `UAT.md` | — | — |

Two things follow from this table.

**State is never passed, only re-resolved.** The dispatchers hand over no variables at all;
they say so out loud. "Context files are resolved inside the workflow via
`… init.execute-phase` and per-subagent `<required_reading>` blocks"
(`skills/gsd-execute-phase/SKILL.md:58`); "resolved inside the workflow (`init verify-work`)
and delegated via `<required_reading>` blocks" (`skills/gsd-verify-work/SKILL.md:33`). Every
consumer — orchestrator or subagent — derives its state from files, so no stale value can be
inherited from a caller. That is what makes fresh-context orchestration safe: there is no
context to lose.

**The state writes are verbs, not file edits.** A stage never hand-edits STATE.md or
ROADMAP.md; it calls a named CLI operation and lets the runtime own the write path. Which is
also why the only stage that both begins and completes a phase — execute — is the only one
with four distinct state verbs, and why `phase.complete` carries an explicit warning against
double-writing state on the resume path (`gsd-core/workflows/execute-phase.md:1498-1501`).

---

## 6. What makes a markdown file executable

There is no interpreter. Everything below is a technique for getting deterministic behaviour
out of a model reading prose. This set is the transferable core of the page.

**1. A JSON envelope at the top of every stage.** Each workflow's first act is a single call —
`gsd_run query init.new-project` (`new-project.md:33`), `init.plan-phase "$PHASE"` with flag
params (`plan-phase.md:84`), `init.execute-phase` (`execute-phase.md:85`),
`init.verify-work` (`verify-work.md:45`) — followed by a literal list of the keys to parse.
All branching data (models per role, paths, feature flags, `response_language`) arrives in one
deterministic payload instead of being discovered by exploration.

**2. Spill-to-file indirection for large payloads.** Every one of the four follows the init
call with:

```bash
if [[ "$INIT" == @file:* ]]; then INIT=$(cat "${INIT#@file:}"); fi
```

The CLI may return a sentinel pointing at a file instead of the payload itself. A stdout
contract that can outgrow stdout needs an escape hatch, and this is a two-line one.

**3. Resolution by search path, because the spec is runtime-agnostic.** The first fenced bash
block of each workflow defines `gsd_run()` via a long chain of existence tests —
project `gsd-core/bin/`, then `.claude/`, `.codex/`, then `PATH`, then roughly fifteen
per-runtime home directories (`~/.claude`, `~/.hermes`, `~/.cursor`, `~/.codex`, `~/.gemini`,
`~/.copilot`, …), ending in an actionable install error
(`gsd-core/workflows/new-project.md:32`, `gsd-core/workflows/verify-work.md:40`). One spec
text, fifteen install layouts, zero conditionals authored per host.

**4. Literal markers as a return protocol.** Subagents signal outcome by emitting an exact
heading, and the orchestrator matches on the string: `## PLANNING COMPLETE`,
`## PHASE SPLIT RECOMMENDED`, `## ⚠ Source Audit`, `## CHECKPOINT REACHED`,
`## PLANNING INCONCLUSIVE` (`gsd-core/workflows/plan-phase.md:882`), plus `## ISSUES FOUND`
from the checker (`:1077`). There are no exit codes across an Agent boundary, so the return
type is a sentinel string — and a return containing *no* recognized marker is itself a defined
case, routed to stall handling (`:901`).

**5. Polling instead of scheduling.** With no event loop, waiting is written as a helper the
model re-runs: `PLANNER_STALL_RESULT=$(gsd_stall_watch "$TS" "{outputFile}" … )` "repeat …
while waiting/active — `marker_received` → step 9; `stalled` → 9a" (`:882`), with the helper
itself lazily read from a step file (`:683`).

**6. Counters narrated as prose, with their lifetimes stated.** `iteration_count`,
`prev_issue_count` and `stall_reentry_count` exist only as text the model is told to track.
Because nothing enforces them, the spec has to say what resets and what persists in the same
sentence (`:1148`). Any state a prompt asks a model to carry must have its lifetime written
down, or the model will pick one.

**7. Template interpolation with no template engine.** Subagent prompts embed shell
expansions — `${AGENT_SKILLS_RESEARCHER}`, populated earlier by
`gsd_run query agent-skills gsd-project-researcher` (`new-project.md:35-37`, used at `:759`
and `:803`) — and, in one place, a JavaScript ternary the model is expected to evaluate:
`${SPECLESS_FALLBACK_DISABLED ? … : ''}` (`plan-phase.md:804`). Two different interpolation
syntaxes in one document, neither of which any program will ever execute.

**8. Conditional loading expressed as an instruction not to read.** See §4.3. `@`-imports are
eager, so "load this only when X" has to become "if X, read this path; otherwise skip — do
not read the file."

**9. A terminal self-test.** `<success_criteria>` is a checklist the model runs against its own
output before declaring the stage done — 13 boxes for plan-phase, 11 for verify-work. Cheap,
and the last line of defence against a stage that narrated more than it did.

---

## 7. Reconstructing a stage spec

From the observed grammar, the minimum viable stage in a framework of this shape is:

1. **An authored dispatcher** with: `name`, `description`, `argument-hint`, an explicit
   least-privilege tool list, an `<execution_context>` block of one-`@`-per-line imports, a
   `<context>` section defining every flag *and* the rule that a flag is active only when its
   literal token is present, and a `<process>` that delegates while naming the gates that must
   not be skipped.
2. **A generator** that produces the runtime's artifact format from that dispatcher, plus a
   `--check` mode in CI. Route the generator through the same converter the installer uses, so
   the committed artifact is a fixture of real installer output.
3. **A lint on the dispatcher's edges** — every `@`-reference resolves, every spec file is
   reachable from some loader. This is the part a generator cannot do for you.
4. **A workflow spec** opening with a machine-readable header declaring `produces`/`consumes`
   and its extension points, then `<purpose>`, `<required_reading>`,
   `<available_agent_types>`, `<runtime_compatibility>`, a numbered `<process>`, and
   `<success_criteria>`.
5. **An init call** returning one JSON envelope, with the parse keys enumerated in the spec,
   and a spill-to-file escape hatch.
6. **Named state verbs** for every write, so no stage hand-edits shared state files.
7. **A `<downstream_consumer>` block in every producing prompt**, naming who reads the artifact
   and what structure they require.
8. **A marker vocabulary** for subagent returns, including a defined case for "no marker".
9. **Explicit lifetimes** for any counter the model is asked to track.

And one caution the repo demonstrates better than it states: a generator proves that the
artifact matches its source, never that the source is true. Every claim your dispatcher makes
in prose — CLI syntax above all — is outside any generator's reach. Point a linter at the
claims you can check mechanically, and assume the rest will drift.
