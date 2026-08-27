# Build your own

The preceding sections took GSD Core apart. This one puts the pieces back down in front
of you and asks the only question that matters if you are here to build something: **which
of these choices should you make, and which are artifacts of somebody else's scale?**

GSD Core grew to 71 commands, 71 generated skills, 35 agents, 46 capability packs, 194
top-level TypeScript modules and 19 harness integrations — on a maintainer count the
`harnesses.md` page puts at one. Some of what it does
is transferable design skill — the kind of thing that is correct on day one of a weekend
project. Some of it is the predictable cost of being that large, and copying it into a
small project buys you the cost with none of the benefit. A few things are load-bearing
mistakes that the repo's own CI is currently failing to catch.

This page separates the three.

!!! info "How to read the claims"
    Two kinds of statement appear below. **Structural claims** cite a repo path and a
    command that produced the number — those are checkable, and you should check them.
    **Scale judgements** ("at ten people this stops paying") are editorial. No path in this
    repository supports them, and none is offered. Where both kinds appear in one paragraph,
    the citation belongs to the first kind only.

## Part 1 — The transferable design skills

These are not principles. They are things you learn to *do*, each one visible in the repo
at a specific place, each one independent of how big your framework gets.

### 1. Compile contracts out of the files that implement them

The strongest single pattern on this site. Do not maintain a registry file describing your
system; put a marker inside each file that *is* the system, compile the markers into an
artifact, commit the artifact, and byte-compare it in CI.

GSD Core's five-stage loop is not written down as a list anywhere an author edits.
`grep -rn "gsd:loop-host" gsd-core/workflows/*.md` returns exactly five hits — one HTML
comment at line 1 of `discuss-phase.md`, `plan-phase.md`, `execute-phase.md`,
`verify-work.md` and `ship.md`. `scripts/gen-loop-host-contract.cjs` parses them into
`gsd-core/bin/lib/loop-host-contract.cjs`: five steps, twelve attachment points, per-step
`agentRoles` and a `coreArtifacts.produces` / `consumes` pair. That file carries
`DO NOT EDIT BY HAND` in its header and is re-derived with `--check` on every CI run.

The gradient shows up repeatedly in this repo, and where I could check it, it ran the same
direction: surfaces with a generator and a `--check` hold; the surfaces I found drifting had
no generator pointed at them.
`npm run lint:generated-sync` chains **14** `--check` invocations
(`node -e "console.log(require('./package.json').scripts['lint:generated-sync'].split('&&').length)"`),
and the surfaces underneath them — the loop contract, all 71 `skills/*/SKILL.md` regenerated
from `commands/gsd/*.md`, the ~308 KB flattened `capability-registry.cjs` — are the ones with
no reported drift. `scripts/check-contract-drift.cjs` reports the three-way agent roster
clean: running it prints `ok check-contract-drift: 35 agents, 38 markers, 0 violations`.

Against that, the surfaces with no generator: three artifact registries
(`gsd-core/templates/README.md`, `src/artifacts.cts`, `gsd-core/references/artifact-types.md`)
that disagree with one another; the six `ns-*` router tables in `commands/gsd/`
(`ls commands/gsd/ns-*.md | wc -l` → 6) with a hand-written alphabetical tie-break; and
`agent_hint`, which has a complete consumer chain and no producer —
`grep -rn agent_hint agents/` returns nothing.

**Cost, stated honestly:** one generator plus one CI script per surface, and generated
output you must commit. The alternative — code review as the enforcement mechanism —
produced each item in the drifted list above.

### 2. Audit a guard's scope as carefully as its rule

This is the refinement that stops "generate everything" from reproducing GSD's exact bugs.
A green check can deliver no coverage in three distinct ways, and all three are live in
this repository.

**(a) A generator proves the artifact matches its source, never that the source is true.**
`gen-plugin-skills.cjs --check` is byte-exact across all 71 pairs. It is also completely
indifferent to the fact that three of the four stage dispatchers describe the CLI three
different ways while the workflows underneath run something else again. Drift migrates to
whichever layer has no generator pointed at it.

**(b) A checker whose scope excludes the violation reads as coverage.**
`scripts/lint-compiled-artifact-sync.cjs` runs on every CI pass and reports success over an
empty pair set. `scripts/lint-hooks-runtime-build-seam.cjs` enforces a real rule and misses
its largest violator because of one constant, `SCAN_ROOT = 'hooks'`.

**(c) A checker can be existential where you need universal.**
`check-contract-drift.cjs` fires only when *no* cited consumer matches, so a row with one
stale citation beside one fresh citation passes clean.

**The practical rule:** give a guard's scope constant the same review as its rule, and
write down what the guard deliberately cannot see — in the place a maintainer will be
standing when it matters, which is the guard's own header, not a wiki.

### 3. Know the test for "this rule must become code"

> Does this rule survive a model that has read it, understood it, and decided it does not
> apply?

If yes, prose is fine. If no, it needs to be a hard block. The evidence in this repo is that the
hard-blocking guards I traced were each written *after* the prose version demonstrably
failed.
`hooks/gsd-write-guard.js:7` opens with `Problem (#973, fix 3 of 3)` and states the
recursion at line 14: an advisory will not do, because #973 records an agent reading the
advisory and reasoning past it. `hooks/gsd-agent-isolation-guard.js:15` puts it as a
general law:

> A prose backstop cannot fix a prose defect — it is the same class of artifact the model
> may equally skip.

The migration recurs well outside the hook layer. `gsd-core/bin/verify-reapply-patches.cjs`
exists because, per its own header at line 26, the gate it replaces "previously trusted
Claude's free-text 'verified:'" assertion (#2969). Prose asserting a fact was replaced by
code checking it.

**Two riders.** You usually learn which category a rule is in the expensive way — budget
for retrofitting rather than trying to classify up front. And state the bound:
`hooks/gsd-write-guard.js:19` says out loud that it "stops accidental and single-shot
collapse, not a determined" agent. What a hook buys is the conversion of *ignoring a
sentence* into *taking one deliberate, path-bound, auditable action*. That is achievable.
Prevention is not, and claiming it is how a guard becomes a false sense of safety.

### 4. Treat the context window as a schema and topology problem

Most frameworks respond to finite context by shortening prompts. GSD Core treats one threat
model — bounded window, agent lost mid-task, cannot be trusted to re-read on resume — as a
design input at five unrelated layers.

`gsd-core/references/context-budget.md:16–17` sets a read-*depth* budget keyed to window
size: below 500,000 tokens, read only frontmatter, status fields or summaries and *never*
the bodies of `SUMMARY.md`, `VERIFICATION.md` or `RESEARCH.md`; at or above it, full bodies
are permitted. That single rule is what makes frontmatter the
read surface, and what turns `requires` / `provides` / `affects` into a transitive-closure
index rather than decoration. `gsd-core/workflows/execute-phase.md` implements a three-band
policy where the inline examples get demoted to on-demand `@`-references while the decision
logic stays inline — thinning the illustrations, not the reasoning, which is the direction
most frameworks get backwards. `cmdTemplateSelect` at `src/template.cts:58` picks one of three
SUMMARY tiers mechanically from plan shape, taking the sizing decision away from the
generating agent so it is a reproducible function rather than a judgement that moves with
temperature. Progressive disclosure is implemented as a directory:
`ls gsd-core/workflows/execute-phase/steps/ | wc -l` → 11 step files, read only when
reached, with conditionality expressed as an instruction *not to read a file* — because
`@`-imports are eager and there is no lazy include.

If you accept a context budget, it determines your frontmatter schema and your agent
topology. It is not a prompt-length setting.

### 5. Type the non-terminal outcome, and bound loops on progress

A nested orchestrator that runs out of budget mid-loop has a caller waiting for a terminal
result. Without a third value in the return type, budget exhaustion gets encoded as success
and you find out weeks later.

`agents/gsd-debug-session-manager.md` defines `## CONTINUE_REQUIRED` at line 312 and
explains at line 320 that it is distinct from both terminal shapes *and* from a genuine
user checkpoint. Line 398 makes the commit behaviour differ on that path, because
committing on a non-terminal return would strand work.

The paired idea is the bound. `gsd-core/workflows/plan-phase.md:1131` tracks
`prev_issue_count`, initialised to `Infinity`, alongside the iteration counter, and line
1140 aborts the moment issue count stops *decreasing* rather than only when a cap is hit.
Line 1148 is the sharp bit: re-entry resets `iteration_count` but `stall_reentry_count`
persists across re-entries and is capped at 2. That asymmetry is what stops "adjust the
approach and try again" from being an unbounded loop wearing a bound. The terminal state is
a named marker downstream logic can branch on, not a silent give-up.

The family is not uniform, which is itself the lesson: the same producer/checker shape
carries hardcoded bounds at several sites with no shared constant and no config key. That
is the predictable cost of writing a policy number inline.

### 6. Give every fallible read a scope envelope

`src/planning-scope.cts` is 49 lines with no imports, and it exports one frozen four-value
enum: `complete`, `truncated`, `unscoped`, `unreadable` (lines 25–28). Its header states
the reason at lines 7–9 — zero items under `COMPLETE` is a real answer; zero items under
the other three means the scan could not run.

The sharpest instance of the idea is a deletion rather than an addition. `getMilestoneInfo`
used to fall back to `{version: 'v1.0', name: 'milestone'}`. That default was removed not
because it was wrong but because it was **output-identical to a successful read of a
genuine v1.0 project**. A default indistinguishable from success destroys the caller's
ability to detect failure.

The rule generalises: never emit a default value that a real input could also produce. And
the envelope must survive to the point of consequence rather than being reconciled away in
the middle — in GSD Core it crosses out of TypeScript entirely, with
`gsd-core/workflows/next.md` reading `.scope` off a JSON result and treating any non-`complete`
value as scan-failed. A safety check that reports "clean" over a list it could not populate
is a silent disarm.

### 7. Assign one writer per shared file — in the prompt *and* in code

`gsd-core/workflows/execute-phase.md:707` instructs the executor:
`Do NOT update STATE.md or ROADMAP.md — the orchestrator owns those writes after all
worktree agents in the wave complete.` And `gsd-core/workflows/execute-plan.md` skips those
writes structurally regardless of whether the agent complied, detecting worktree mode three
separate times (lines 433, 490, 529) with the comment
`.git is a file in worktrees, a directory in main repo`.

Belt and braces: the instruction so the agent understands the rule, the code so a
disobedient agent cannot break it. Two consequences worth copying. Durability is defined by
the isolation mechanism, not the medium — a SUMMARY must be committed before the agent
returns, because the orchestrator force-removes the worktree and an uncommitted file is
gone. And one agent definition can legitimately carry two mutually exclusive success
contracts selected by dispatch mode: `agents/gsd-executor.md` in sequential mode *requires*
exactly the writes worktree mode forbids.

Where the medium genuinely must be shared, serialise in code. `withPlanningLock` in
`src/planning-workspace.cts:212` is an `O_EXCL` create whose hardening reads as an incident
list: a pid-liveness gate, an identity re-confirm before stealing a lock, atomic
rename-then-remove, and a named-errno retry set whose comments cite Docker overlay-fs and
NFS by name (lines 102–112).

### 8. Separate the format layer from the protocol layer before you port anything

Nineteen of the 46 capability packs declare a host integration
(`grep -rl '"hostIntegration"' capabilities/ | wc -l` → 19). The bill splits cleanly.

**Format projection absorbed into data.** One installer path, a set of mechanical
frontmatter converters under a naming convention, and a dispatch table where absence of an
entry means identity. Adding a harness's file format is a descriptor edit.

**Protocol adaptation stayed bespoke.** Read the negotiated axes as columns
and some collapse hard while others do not collapse at all. The concrete cost: of the 26
scripts in `hooks/` (`ls hooks/*.js hooks/*.sh | wc -l` → 26), **8** are per-harness
adapters — six `gsd-cursor-*`, two `gsd-windsurf-*` — sharing nothing with each other.

And extract the abstraction from *two* implementations, not one.
`src/host-integration-adapters/imperative-hook-bus.cts` parameterised the variable the
first harness exposed and hardcoded the one it did not, so the second harness got a
copy-shaped sibling rather than a second use of the abstraction.

### 9. Put identity on the return edge of a closed loop

An arrow back is not a loop; it is a way to lose state. GSD Core's verify→fix→verify cycle
carries an identity token the whole way round. `gsd-core/workflows/verify-work.md:411`
defines `gap_id: G-{phase}-{N}` as a stable id. Gap-closure plans tag it in frontmatter;
re-entered verification runs a `reconcile_gaps` step (line 430) that flips a gap to
`resolved` only when both a plan citing its `gap_id` *and* a SUMMARY proving that plan
executed exist — line 444 leaves the gap open if either is missing. The workflow states the failure mode it prevents in its own
words: without reconciliation, verify re-diagnoses closed gaps as fresh blockers.

The regression case is handled by deliberately *not* reusing identity: line 451 says a
resolved gap reported broken again gets a **fresh** `gap_id`, so "fixed once, broke again"
stays distinguishable from "never fixed". Stable id, tag on the fix, and existence proof
are three parts of one idempotency mechanism, and you need all three.

### 10. Make the source format and the distribution format different on purpose

`package.json`'s `files[]` array does not contain `capabilities` — all 46 first-party packs
are build-time *inputs* that do not ship in the npm package. `scripts/gen-capability-registry.cjs`
flattens them into one committed ~308 KB `gsd-core/bin/lib/capability-registry.cjs` with
fragment text inlined and cross-capability invariants already resolved. Authors get one
diffable folder per capability; the runtime gets one validated table and no filesystem walk
on the hot path. Critically, the *same* build function runs on the load path, so no derived
view can drift from the built one.

The payload/packaging boundary is a directory, not a convention: `gsd-core/` is recursively
copied into every host's config dir, and top-level `bin/` is packaging that never is. That
rule is asserted in the header of `bin/gsd-mcp-server.js` (lines 6–9) — the file most likely
to be misfiled — where it explains that this is "a PACKAGE bin the host spawns via
`npx gsd-mcp-server` … NOT a per-runtime artifact copied into a host's config dir", because
placing it under `gsd-core/bin/` "would leak it into every runtime install".

---

## Part 2 — Reuse, adapt, drop

Same eight choices, three scales. The columns are **Solo** (one maintainer, possibly you in
six months as the second maintainer), **Small team** (2–5, everyone reads everyone's
diffs), and **Team of ten** (nobody reads all the diffs; that is the whole difference).

| GSD Core's choice | Solo | Small team (2–5) | Team of ten |
|---|---|---|---|
| **Spec-driven loop** | **Reuse** the staged artifacts; **drop** the attachment-point contract | **Reuse**; contract as a plain ordered list | **Reuse** both — compile and `--check` the contract |
| **File-based state** | **Reuse** as-is | **Reuse** + one-writer rule | **Reuse** + one-writer rule enforced in code + locking |
| **Generated skill files** | **Reuse** | **Reuse** | **Reuse** |
| **Subagent contracts** | **Adapt** — keep the return types, drop the roster | **Adapt** — roster as one file, checked one way | **Reuse** — three-way roster + drift checker |
| **Runtime code** | **Reuse** the five categories; **drop** the shape | **Reuse** categories; **adapt** shape (real modules + cycle checker) | **Reuse** categories; **drop** the flat bag outright |
| **Hooks as enforcement** | **Reuse** — one host, retrofit per incident | **Reuse** — same, plus write down the bound | **Adapt** — name the single enforcement host in the README |
| **Capability packs** | **Drop** the extension contract; **reuse** build-input-vs-distribution | **Adapt** — packs as folders, one flattened registry | **Reuse** in full |
| **Multi-harness adapters** | **Drop** | **Drop**, or format projection only | **Adapt** — projection everywhere, enforcement to one |

### Spec-driven loop — reuse the premise everywhere, buy the machinery once

The premise (a fixed sequence of stages, each producing a named artifact the next one
consumes) is correct at every scale and costs nothing. `loop-host-contract.cjs` declares a
single strand — `CONTEXT.md → PLAN.md → SUMMARY.md → UAT.md` — with no fan-in and no step
reading two upstream artifacts. That narrowness is not a limitation, it is the feature: it
is what makes the chain checkable at all.

The machinery is a separate purchase. `gsd-core/bin/lib/capability-validator.cjs` type-checks
third-party extension steps against the host: it maps each core artifact to the index of the
`:post` point that produces it, rejects a step consuming an artifact at a point before
anything produces it, refuses self-satisfaction from a step's own `produces`, and resolves
same-point ordering by topological sort with cycle detection that fails the build. That
validator is 3,749 lines (`wc -l gsd-core/bin/lib/capability-validator.cjs`), and **it only
pays if third parties exist**. Solo,
they do not. You are the third party, and you can read the ordering off the file.

At ten people, extensions arrive whether or not you designed for them, and the failure you
are buying insurance against is somebody's plugin silently consuming an artifact that has
not been written yet.

!!! warning "The declared chain understates real coupling"
    The contract's `consumes` lists are *minimums*, not read-sets. Above a configured
    context-window threshold, executors additionally receive prior-wave SUMMARY, phase
    CONTEXT and RESEARCH; the verifier's set differs again. The full read-set could not be
    declared even in principle, because its membership is a function of a config key.
    Declare the minimum — it is checkable, and the full set is not — but write the gap down
    somewhere, because a reader who trusts the contract will underestimate what a stage
    touches.

### File-based state — reuse at every scale; the cost is parallelism, not headcount

Markdown files with YAML frontmatter as the state store is the cheapest correct choice on
this list. It survives process boundaries, diffs in review, resumes after a crash, and is
readable by the model without a tool call. Nothing about it gets more expensive with more
people.

What gets more expensive is *concurrency*, and that is driven by whether you dispatch
agents in parallel, not by headcount. A solo builder running one agent at a time needs none
of `withPlanningLock`'s pid-liveness gate or errno retry set. The moment you fan out, you
need the one-writer rule from skill 7 — and you need it in code, because the prompt version
alone has a known failure mode and this repo implements both.

The one thing to copy at all scales, free: **mutability contracts**. Label every artifact
IMMUTABLE, OVERWRITE or APPEND, and publish the read-order a resuming agent should follow.
The labels alone are an audit trail. The read-order is what turns negative knowledge —
"this was already tried and failed" — into work that does not get repeated.

### Generated skill files — reuse, and scale genuinely does not move this

The only row with the same verdict in all three columns, so it owes you a reason.

The cost is one generator plus one `--check` step, and it is constant. It does not grow
with team size, corpus size, or anything else. Meanwhile the drift it prevents starts
accruing on day one, because your framework's second maintainer is you in six months, and
you will not remember that `commands/gsd/foo.md` and `skills/gsd-foo/SKILL.md` were supposed
to agree.

GSD Core's version: `commands/gsd/*.md` is the sole authoring surface; everything else is
output. Across all 71 pairs the transformation reduces to a handful of mechanical changes.
The body is *nearly* verbatim: `transformContentToHyphen` rewrites `/gsd:<cmd>` to
`/gsd-<cmd>` inside the prose (#3583/#2808), so 18 of the 71 bodies differ — `ship.md`'s
"After /gsd:verify-work passes" becomes "After /gsd-verify-work passes". The other 53
carry through byte-for-byte. Three structural rules make it
work, and all three are worth copying verbatim:

- **Reconstruct frontmatter from a whitelist, do not filter it.** The authored surface is a
  deliberate superset of the shipped one, so the default for any new key is "does not ship."
  `requires:` has two consumers: `scripts/lint-skill-deps.cjs` proves closures, and
  `src/install-profiles.cts` (`parseRequires` at `:121`) computes the transitive closure
  deciding which skills land on disk. Both are build- and install-time — there is no
  *runtime* resolver to get wrong, which is the property that makes it safe.
- **Separate the two surfaces with a greppable token.** Commands use `gsd:`, skills use
  `gsd-`. A linter aimed at the wrong surface finds *nothing* rather than finding something
  wrong, which is the difference between a loud failure and a quiet one.
- **Record the reason for every dropped field in the code that drops it.**
  `src/runtime-artifact-conversion.cts:468–472` explains in five lines of comment why
  `effort:` is not emitted into skill frontmatter: a static value changes `output_config.effort`
  on invocation and invalidates the caller's prompt cache (#3151). Nobody would ever
  reconstruct that reason from the source, and without it the omission looks like a bug.

Two costs to accept. The generated tree must be committed, because npm and plugin-only
installs run no build; staleness becomes a CI failure rather than a merge surprise, which
is the trade you want. And build-at-publish buys a single source of truth while costing
cross-boundary type checking. This repo has a live example:
`scripts/gen-plugin-skills.cjs:44` calls `convertClaudeCommandToClaudeSkill` with **five**
arguments, while its definition at `src/runtime-artifact-conversion.cts:452` declares
**four**. The trailing argument is silently discarded, and nothing catches it — because the
call crosses a `require()` into a gitignored emit that TypeScript never sees.

### Subagent contracts — the return types are free; the roster is a headcount artifact

Split this one hard, because the two halves have opposite verdicts.

**Free at every scale, copy immediately:** the non-terminal return value
(`## CONTINUE_REQUIRED`), progress-bounded loops, named terminal markers instead of silent
give-up, and inferring completion from observable side effects rather than the agent's own
declaration. That last one is worth stressing — under backgrounded parallel dispatch,
`execute-phase.md` detects completion from SUMMARY existence and git commit state rather
than matching the executor's declared success string.
`gsd-core/references/agent-contracts.md:97` records that as deliberate: `## PLAN COMPLETE`
is the executor's completion marker and execute-phase.md does not regex-match it — "this is
intentional behavior, not a mismatch". When the return channel is unreliable, believe the filesystem.

**Not free:** a 35-agent roster kept aligned across `agents/`,
`gsd-core/bin/shared/model-catalog.json` and `gsd-core/references/agent-contracts.md`, with a
CI script maintaining the three-way. That alignment exists because at 35 agents nobody can
hold the roster in their head. At five agents you can, and the checker is pure overhead.

One structural idea from that machinery *is* worth copying at any size, though, because it
inverts an intuition most people hold: **make some file other than the directory listing
your roster.** `src/agent-install-check.cts:159` derives the expected roster from
`Object.keys(MODEL_PROFILES)` — from the catalog, not from `agents/`. An agent file dropped
in with no catalog entry is invisible to the install checker. That sounds like a bug until
you notice what it buys: "I added a file and nothing happened" becomes impossible to ship
silently, because the registration is explicit.

The corollary, also cheap: **declare position at the layer that has it.** Zero of the 71
command files declare a loop position
(`grep -l "loop-host\|loop_position" commands/gsd/*.md | wc -l` → 0), because the workflow
underneath does the work and carries the marker. Metadata that cannot drift from behaviour is metadata you never have to
check.

### Runtime code — reuse the five categories, drop the shape

GSD Core wrote deterministic TypeScript for five things: parsing and serialising the model's
own markdown output, state transitions, concurrency, platform portability, and trust
boundaries. Everything else — what to build, whether a plan is sound, how to phrase a review
finding — lives in prompts. **Those five categories transfer unchanged to any scale.** They
are a good answer to "where does the code stop" and I would not modify the list.

`src/uat-predicate.cts` shows the asymmetry cleanly: it exists because a naive whole-file
regex false-matches `result:` lines inside frontmatter, fenced blocks, blockquotes and HTML
comments. The model's *judgement* about whether verification passed is trusted; the
runtime's *reading* of that judgement is not left to a regex. The same line is drawn
structurally in `gsd-core/bin/lib/capability-validator.cjs:2802–2805`, where
`gates[].check.agentVerdict` is *forced* to `blocking: false` with the message
`non-deterministic checks may not halt the loop`. In a framework whose premise is
model-driven workflow, an LLM verdict can advise and structurally cannot halt. That is a
better place for that rule than any style guide.

Two more habits that cost nearly nothing and pay at every scale: errors as frozen `reason`
enums plus a JSON error mode, so tests assert typed fields instead of grepping stderr
(`grep -l "Object.freeze(" src/*.cts | wc -l` → 53 of the 194 top-level modules); and an
inline `#NNNN` or `ADR-NNNN` on non-obvious branches, which is what turns a codebase into a
searchable incident log — `verify-reapply-patches.cjs:26` explains its own existence in one
line because of this habit.

**The shape is the part to drop.** `src/` is a flat bag: 194 top-level `.cts` files
(220 including subdirectories) with prefix-only grouping, guarded by 22 bespoke `local/*`
ESLint rules — every one of them wired at `error` in at least one config block
(`grep -oE "'local/[a-z0-9-]+': 'error'" eslint.config.mjs | sort -u | wc -l` → 22) — plus
40 `scripts/lint-*.cjs` scripts, 35 steps chained into `lint:ci`, and 821 test files
under `tests/`.

That is an enormous amount of machinery bolted onto a structure with no boundaries. The
bill it produces: **eight modules over 100 KB** with nowhere to split them —
`runtime-hooks-surface`, `runtime-artifact-conversion`, `init`, `state-transition`, `state`,
`commands`, `install-engine`, `phase` (`find src -name '*.cts' -size +100k`) — and **no import-cycle
tooling I could find** — `grep -rniE "madge|dpdm|no-cycle|circular" package.json
eslint.config.mjs scripts/*.cjs` returns nothing, and `eslint-plugin-import` is not among the
14 `devDependencies`. Acyclicity across 220 files with two interleaved export styles is
asserted in comments and enforced by no tool in that search.

!!! tip "If you go flat, buy the cycle checker on day one"
    It is the one piece of tooling whose absence compounds silently. At 194 files there is
    no cheap moment to add it, because you will find cycles and have nowhere to move code
    to. At 20 files it is a five-minute install.

The counter-example in the same repository is worth more than the rule.
`bin/install.js` is a single file of **642,054 bytes** (`wc -c bin/install.js`). Every
individual decision inside it is defensible — splice rather than re-serialise so a user's
comments and key ordering survive, marker-delimited ownership with a named inverse per
writer, confinement roots checked before any recursive delete. What it lacks is the build step and the
guard layer that made the module boundary the path of least resistance in `src/*.cts` — a
`.cts` file gets compiled, linted by the nine `local/*` rules the `src/**/*.cts` block promotes to `error` (out of 22 bespoke rules in `eslint-rules/`) and swept by the 28 of 40 `lint-*` scripts that `lint:ci` actually runs;
`bin/install.js` gets `eslint` and nothing structural. **Module boundaries come from forcing functions, not intentions.** And those 22
ESLint rules, each naming the incident that motivated it, are a better architecture document
than any style guide, because a rule that cites its own incident cannot be argued away by
someone who was not there.

### Hooks as enforcement — reuse, but decide up front which host gets it

The `#973` evidence in `hooks/gsd-write-guard.js` says the first hard block earns itself
immediately. A solo builder has exactly one host, so "which host gets enforcement" is not a
question — reuse the pattern, retrofit each guard when an incident proves prose insufficient,
and write the bound in the header the way `gsd-write-guard.js:19` does.

The thing that scales badly is *how many hosts* get enforcement. GSD Core projects artifacts
to 19 harnesses and guards meaningfully far fewer, and the gap is visible in the shape of the
directory: 8 of the 26 scripts in `hooks/` are per-harness adapters sharing nothing. Worse,
the same `.planning/` write policy is implemented three times at three different enforcement
strengths from character-identical regexes, and the strictest copy sits on a host nothing
calls.

At ten people that ambiguity is a real cost, because nobody can tell from the code which
host is authoritative. The honest shape is one sentence in the README: *artifacts project to
everything; enforcement is Claude Code only.* State it rather than letting it be discovered
in the shape of `hooks/`.

!!! danger "One registration surface per hook"
    This is a **drop** at every scale, not an adapt. Where a hook is registered in two places
    — a plugin manifest and an installer — the hook-layer defects documented elsewhere on this
    site are drift between them: a config flag that silently does nothing on plugin installs, a matcher
    narrowed on one surface only, a guaranteed no-op that still spawns a subprocess. One
    surface, generated into the others if you must.

### Capability packs — the packaging idea is cheap, the extension contract is not

Two separable ideas again.

**Cheap, copy at any scale:** the source format and the distribution format should differ,
and the boundary should be a directory. Authoring in 46 folders and shipping one flattened
committed table is a good trade even for a solo project with three packs, because the
flattening function is also the load function and therefore cannot drift.

**Expensive:** a validated extension contract with declared attachment points, artifact
type-checking and third-party trust boundaries. That machinery exists to let people who are
not you extend the loop safely. If nobody is doing that, it is elaborate self-defence.

The best evidence that install-time composition is genuinely hard sits in the profile system.
Three install profiles are computed as a transitive closure over one `requires:` field.
Machine enforcement catches unreachability. It does not catch *completeness*, and the result
is the single most instructive fact on this page:

!!! failure "The minimal profile omits the entry point to the step CI byte-checks"
    Resolving the closure over `requires:` in `commands/gsd/*.md` from the base sets at
    `src/install-profiles.cts:73–104` gives `core` = **15** skills and `standard` = **23**.
    `ship` is in **neither** — and none of the 15 `core` members routes to it
    (`grep -ln ship commands/gsd/{phase,progress,review,quick,verify-work,surface}.md`
    returns nothing). Meanwhile `loop-host-contract.cjs` byte-checks in CI that the loop has
    five steps, the fifth being `ship`, whose `consumes` list is `UAT.md`.

    Be precise about what is missing. `gsd-core` **is** in `package.json`'s `files[]`, so
    `gsd-core/workflows/ship.md` is present on disk in a `core` install — the loop *logic*
    ships. What does not ship is the **invocation surface**: no `skills/gsd-ship/`, and no
    `core` skill that reaches it. So the minimal profile can produce `UAT.md` and offers the
    user no installed way to invoke the step that consumes it. The same closure also puts
    both `config` and `settings` into the tier meant to be minimal.

    I found no comment or marker recording that omission as deliberate
    (`grep -rn ship src/install-profiles.cts scripts/lint-skill-deps.cjs` turns up only
    unrelated uses of the word). Two CI checks pass over this: one proves the contract
    matches its markers, the other proves every declared dependency resolves. Neither is
    positioned to ask whether the shipped profile can *reach* the contract. That is skill 2
    in the wild — the scope of the check, not the check.

Copy the closure idea — one declaration, several consumers, no second source of truth. But
add the assertion nobody wrote: **every install profile must expose an entry point for
every loop step.**

### Multi-harness adapters — drop it unless portability is the product

The hardest verdict on the list, and the one I hold most firmly. Nineteen harness
integrations is an extraordinary amount of surface for a framework with one maintainer, and
the return is real only if reaching users on other harnesses is the point of the project.

If it is not, the costs are: 8 per-harness hook adapters sharing nothing; one policy
implemented three times at three strengths; and a `tools:` grant translation surface where a
bad translation *aborts the entire agent load* rather than degrading.
`src/runtime-artifact-conversion.cts:2231–2232` spells that out: an invalid tool name "fails
frontmatter validation … and aborts the entire agent load (#1394)". That failure mode is the
one to weigh, because a portability layer that fails loudly on the host you
do not test is worse than not supporting that host.

At a team of ten with portability as a goal, the shape that works is the one this repo
arrived at without stating: **full artifact projection to every harness, enforcement to
one.** Projection is descriptor data and scales. Enforcement is bespoke and does not.

And when you do build the abstraction, extract it from two implementations rather than one.
The imperative hook bus parameterised what the first harness varied and hardcoded what it did
not, so the second harness produced a sibling rather than a second caller. One data point
does not define an axis.

---

## The one thing to take if you take nothing else

The pages on this site kept arriving at the same finding from different directions, so it is
worth stating plainly:

> **Pick the surfaces that must not drift, generate them from a single source, and compare
> them byte-for-byte in CI. Then audit the scope of each comparison as carefully as you
> audited its rule.**

The first half is what GSD Core got right: of the surfaces examined across this site, the
ones with a generator and a `--check` are the ones still in agreement. The second half is
what it is still getting wrong, and the evidence is a `core` profile that ships an incomplete
loop straight past two green checks — neither of which was ever asked the right question.
