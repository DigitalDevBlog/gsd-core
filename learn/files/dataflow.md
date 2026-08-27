# State machinery

GSD Core has no database. Its entire state lives in markdown and JSON under `.planning/`,
written partly by code and partly by an LLM following a workflow document. That choice
forces a specific engineering problem to the surface: **every read is a parse, and every
parse can fail partially.** A directory listing that threw. A milestone section whose
window closed early. A `STATE.md` whose `## Current Position` heading is missing.

The interesting part of this codebase is not the parsers. It is the machinery built around
them so that a partial read is never silently rounded to a confident answer — and so the
loop can be killed mid-execution and pick up where it left off.

This page is about that machinery: how roadmap state is parsed, how it is reconciled
against disk, what the snapshot layer is for, and what actually survives an interruption.

## The artifact graph

Reads and writes are not symmetric here. Writes go straight at files. Reads mostly do not —
they route through a query layer that composes a scoped projection first. The one place
that *does* stat a file directly is the interruption checkpoint, and that asymmetry is
deliberate.

```kroki-plantuml
@startuml
left to right direction
skinparam componentStyle rectangle
skinparam shadowing false
skinparam defaultFontSize 12
skinparam linetype ortho

package "Stages (gsd-core/workflows/*.md)" {
  rectangle "next.md\n(router)" as N
  rectangle "plan-phase.md" as P
  rectangle "execute-phase.md" as X
  rectangle "verify-work.md" as V
  rectangle "pause-work.md" as PA
  rectangle "resume-project.md" as R
}

package "Read path (src/*.cts)" {
  rectangle "gsd_run query\n(read-only commands)" as Q
  rectangle "single-owner derivations\n+ SCOPE envelope" as S
}

package "Artifacts (.planning/)" {
  file "ROADMAP.md" as RM
  file "XX-VERIFICATION.md" as VF
  file "XX-YY-PLAN.md" as PL
  file "XX-YY-SUMMARY.md" as SM
  file "XX-CONTEXT.md" as CX
  file "STATE.md" as ST
  file ".continue-here.md\nHANDOFF.json" as CH
}

N --> Q : query
P --> Q : query
V --> Q : query
Q --> S
S --> RM : reads
S --> VF : reads
S --> PL : reads
S --> SM : reads
S --> ST : reads

P --> CX : reads
X --> CX : reads
X --> PL : reads
P ..> PL : writes
X ..> SM : writes
X ..> ST : writes
V ..> VF : writes
PA ..> CH : writes
N --> CH : stat only (Gate 1)
R --> CH : reads (find)
R --> ST : reads
@enduml
```

The read band is one node in the picture and two implementations underneath it.
`planning.inspect` (`src/planning-inspect.cts`) reaches the shared derivations through
`buildPlanningSnapshot`, as does `src/verify.cts` (imported at line 56, invoked at 1559
and 1636) — while `roadmap.analyze` (`src/roadmap.cts`) composes the same
owners directly and never touches the snapshot. That split matters later: it is why the
containment gap in [Where the machinery leaks](#where-the-machinery-leaks) lands on the
inspect path and not on the analyze path.

Artifact roles, locations and consumers are inventoried in
`gsd-core/references/artifact-types.md`, which opens with the rule that makes the graph
above the real specification: *"A well-formatted artifact that no workflow reads is inert —
the consumption mechanism is what gives an artifact meaning."*

## A zero that is not an answer

The spine of the read path is 49 lines with zero imports: `src/planning-scope.cts`. Its
whole content is a frozen four-value enum — `COMPLETE`, `TRUNCATED`, `UNSCOPED`,
`UNREADABLE` — and a `Scope` type. Its header states the problem it exists to solve:

> `SCOPE.COMPLETE` with zero items is a real answer — a phase genuinely has no plans yet.
> `SCOPE.TRUNCATED`, `SCOPE.UNSCOPED`, and `SCOPE.UNREADABLE` with zero items are NOT —
> they mean the scan could not see (part of) the phase directory, so a caller must not treat
> that zero as "this phase has no plans."

Modules on the read path — `src/roadmap-parser.cts`, `src/planning-snapshot.cts`,
`src/planning-inspect.cts`, `src/state-document.cts`, `src/verification.cts` — import it and
return `{ value, scope }` envelopes rather than bare values. Two design decisions are worth
stealing whole:

**It is a leaf module on purpose.** The header says so: *"this is a leaf module (mirrors
`src/phase-id.cts`). It imports nothing, so any consumer can depend on it without risking a
cycle."* A discriminator that everything returns has to be importable from everything. If it
carries dependencies, it becomes a cycle magnet within a release.

**It is a frozen enum, not a message string.** The stated reason is that `CONTRIBUTING.md`
bans raw-text matching on outputs and requires a typed IR, so callers branch on a value
rather than pattern-matching prose. The same rule is then applied to a place most codebases
exempt — subprocess output. `buildPlanningTrackedField` in `src/planning-snapshot.cts`
determines repo presence from `git rev-parse --is-inside-work-tree`'s *exit code* rather
than matching git's stderr, because git localizes its error text; the comment calls this
*"exactly the 'raw text matching on subprocess output' CONTRIBUTING.md bans, applied to
production code rather than a test."*

!!! note "The module admits it is provisional"
    `src/planning-scope.cts`'s header marks its own contract `PROVISIONAL ... may be revised
    before the epic (#3180) ships`. A contract module that says out loud how settled it is
    tells a downstream author how much to build on it — cheap to write, and absent from
    almost every codebase.

## Two folds and a hard guard

A scope on a single read is easy. The coordination problem is combining several.

`src/planning-snapshot.cts` defines a severity ordering (`COMPLETE` 0 → `UNREADABLE` 3) and
`worstScope(...scopes)` around line 77-92, described in-file as *"the one new piece of
coordination logic"* the module introduces. The comment is careful about its own
epistemics: the ordering *"is a genuine design choice, not inherited from anywhere"*, and
`TRUNCATED` vs `UNSCOPED` *"are not ranked against each other by any upstream decision"* —
the ordering exists only so a future rule can name which failure was worse.

`src/planning-inspect.cts` imports the same function and folds again, per phase row
(`worstScope(phase.scope, plans.scope, uat.scope, goal.scope, dependencies.scope)` at line
1174) and once more at the top level.

The chain terminates in refusal. `src/health-diagnostic-rules/roadmap-disk-consistency.cts`
opens both of its rules — line 187 and line 238 — by returning an empty diagnostic list
whenever `snapshot.roadmapDeclaredPhases.scope` is not `COMPLETE`. The comment names the
failure mode precisely: an empty declared-phase list *"must not be mistaken for 'the roadmap
legitimately declares zero phases' here."*

That is the shape: **propagate, fold, then guard at the point of consequence.** The scope
value never gets reconciled away in the middle; it survives to the exact site where acting
on it would produce a wrong claim.

## Withhold rather than guess

The same decision is expressed three independent times, by three authors solving three
different problems — which is the strongest evidence available that it is a real house rule
rather than one person's taste.

| Site | Mechanism |
|---|---|
| `cmdRoadmapAnalyze` (`src/roadmap.cts`) | Computes a **second, independent** `progressScope`; emits `progress_percent: null` unless it is `COMPLETE`, plus a separate `progress_scope` field so a consumer can see *why* |
| `makeFraction` (`src/planning-inspect.cts`, line 876) | Returns `percent: null` and pushes a `PERCENT_WITHHELD` diagnostic — *"a percentage derived from an incomplete read would be a confident wrong answer"* |
| `getMilestoneInfo` (`src/roadmap-parser.cts`) | Its old `{version:'v1.0', name:'milestone'}` default was **deleted**, because that default was output-identical to a successful read of a genuine v1.0 project |

The third row is the sharpest of the three. The bug was not that the fallback was wrong —
it was that the fallback was *indistinguishable from success*. Any default value that a real
input could also produce destroys the caller's ability to detect failure. That is a lint you
can run in your head over your own defaults.

The `progress_scope` field deserves a second look too. Two scopes coexist in one payload and
are allowed to disagree — the top-level `scope` describes the heading-windowing that
produced `phases`/`total_plans`, while `progressScope` describes the directory enumeration
that produced the percentage. The comment refuses to reconcile them, because *"a consumer
seeing `progress_percent: null` needs a field to tell WHY without reading source."* Merging
two independent confidences into one number is itself a form of guessing.

## Disk is the authority; the checkbox is metadata

Reconciling parsed state with disk needs an answer to "who wins." GSD Core answers it once,
in one function, and then makes every other module route through it.

`isPhaseComplete` (`src/verification.cts`) is documented as *"the single canonical owner of
'is phase P complete?'"* and is **disk-strict**: it reads `*-VERIFICATION.md`
unconditionally, plan count is not a precondition, and — the load-bearing sentence — *"A
ROADMAP checkbox has no machine authority and is never consulted — this function never reads
ROADMAP.md."*

Two follow-through details make that stick:

- **Two non-answers are distinguished.** `complete` is exactly `verification.status ===
  'passed'`, but the full routing result rides along so a caller can tell a *failing*
  verdict (`gaps_found`, `human_needed`, `stale`) from an *absent* one (`missing`). And
  `scope` goes `UNREADABLE` when the phase directory itself could not be listed — a caller
  *"must not read `value.complete: false` here as a confident 'not complete'"*.
- **The demotion is published, not just enforced.** `planning-inspect` emits
  `roadmap_acceptance.authoritative: false` in its payload. An external consumer learns the
  authority model from the data, not from source archaeology.

`src/planning-snapshot.cts` still parses the ROADMAP checkbox — into
`roadmapPhaseCheckboxes` — and defends doing so with one line worth memorizing: *"reading
the data is not re-litigating who is authoritative."* Authority and observability are
separate questions. The checkbox is useful evidence about human intent even when it has no
vote.

## Windowed and un-windowed, kept on purpose

The snapshot carries two phase enumerations that look redundant and are not.
`phaseDirs` is milestone-**windowed** (only directories whose phase id is declared by the
current milestone); `allPhaseDirNames` is described in-field as its *"un-windowed twin."*

The reason is in `roadmap-disk-consistency.cts`'s header, and it is a genuinely subtle
class of bug. W007 is the rule "this phase directory has no matching ROADMAP entry." Source
it from the windowed set and it becomes *"structurally inert: every member of the set is
already provably claimable."* The rule would run, pass, and prove nothing.

What lifts this above a plausible argument is that the header records an empirical check —
a `node -e` trace against a real `buildPlanningSnapshot` in which a genuine orphan directory
(`04-extra`) was *"silently absent from `phaseDirs.value` and W007 fired zero diagnostics."*

!!! tip "The generalizable rule"
    Some checks are defined by what a filter **excludes**. Any correctly-scoped view is the
    wrong input for them, and the failure is silent-pass rather than error. When you narrow
    a collection for one consumer, ask whether another consumer's question lives in the
    discarded remainder.

## What the snapshot is for

`buildPlanningSnapshot(cwd)` builds a parsed projection of `.planning/` — deliberately
additive, and growing — `src/planning-inspect.cts:23` puts numbers on it ("4 fields at
Phase 10, 20+ by Phase 12"), while `src/planning-snapshot.cts:118` states the
additive-only rule without the counts. Two
constraints explain its existence, and neither is "caching."

**Rules may not do I/O.** Every health diagnostic is a `check(snapshot)` with one signature
and no ambient reads. So the probes live in the builder instead: `git ls-files -- .planning`,
`git worktree list`, `checkAgentsInstalled`. The comments openly admit that `agentInstall`
and `worktreeHealth` are *not* `.planning/`-sourced and are exposed anyway — named as such
*"so a future reader does not mistake them for §7 derivations."* The uniform rule signature
was judged worth the impurity, and the impurity was labelled rather than hidden.

**The internal shape must not leak.** `src/planning-inspect.cts` refuses to serialize the
snapshot, and says why at length: `PlanningSnapshot` is *"explicitly additive and still
growing (4 fields at Phase 10, 20+ by Phase 12)"* while schema-v1 is a frozen external
contract, so handing the churning shape to external consumers is *"a Hyrum's-Law break
waiting to happen."* It declares `PLANNING_INSPECT_SCHEMA_VERSION = 1`, its own frozen
`INSPECT_DIAGNOSTIC` vocabulary, and maps into a flat schema of its own. The invariant it
states — *"Adding a field to `PlanningSnapshot` must never change what `planning.inspect`
emits"* — is the whole firewall in one sentence.

There is a smaller convention worth copying: **one builder, several named outputs**.
`buildStateFields` returns three fields from a single `STATE.md` read *"rather than reading
and parsing the same file three times"*, and — the better half of the argument — so that
the degraded-read warning *"stays a single call site instead of a risk of tripling on one
degraded read."* A sibling builder states the inverse constraint just as crisply: *"the
three subjects are independent QUESTIONS, not independent SCANS."*

## One grammar, filtered views

Consistency-with-disk is impossible if two call sites disagree about *what the current
milestone is*. `src/roadmap-parser.cts` collapses that to a single grammar constant —
`MILESTONE_HEADING_LINE_SOURCE` at line 202, described as *"the ONE textual expression"* of
the milestone-heading grammar in the file. `listMilestoneHeadings` (version-agnostic
enumeration) and `locateMilestoneHeadings` (version-filtered) are both built from it, the
latter documented as *"a version-FILTERED VIEW over ... the SAME grammar"* rather than a
second expression of it.

The layering continues: `selectMilestoneHeading` is the sole selection owner,
`computeMilestoneSectionEnd` the sole window-end owner, `sliceMilestoneWindow` the sole
composition of the two, and `classifyMilestoneWindow` a pure, I/O-free decision table.

The most transferable idea here is that **composition counts as duplication**.
`sliceMilestoneWindow` exists because two call sites had each independently composed
`locateMilestoneHeadings` + `computeMilestoneSectionEnd` into a window *"and they disagreed
(one skipped CLOSED headings, the other did not)"* — with the diagnosis spelled out:
*"calling the owner and then re-assembling the result locally is indistinguishable from
re-deriving it."* Extracting shared *functions* is not enough; the shared *assembly* has to
be owned too.

And the convention is enforced by tooling rather than trust. `scripts/lint-phase-id-drift.cjs`
matches `/^\s*\/\/.*phase-id-owner:/` and requires a dedicated comment line to sanction any
hand-rolled phase-id regex — which is why `src/roadmap.cts` carries sanction lines naming
exactly what a "fix" would break. `scripts/lint-milestone-window-drift.cjs` designates
`src/roadmap-parser.cts` as its `OWNER_FILE`.

!!! warning "The guard's blind spot is documented inside the file it exempts"
    Because the owner file is exempt by construction, intra-file copies are invisible to the
    linter. `src/roadmap-parser.cts` records that a byte-identical inline copy of
    `hasVersionedMilestones` existed in two of its own functions and *"was found by review
    instead"*, because *"the guard does not catch intra-owner-file copies by construction."*
    If you build a drift lint, write down what it cannot see — in the place a maintainer will
    be standing when it matters.

## Surviving interruption

Four independent layers, none of which assumes the others worked.

### 1. The write mutex

`src/planning-workspace.cts` owns the path seam (`planningDir`, `planningRoot`,
`planningPaths` — the nine canonical file paths) and `withPlanningLock`, the only write
mutex in this set. It is an `O_EXCL` create (`flag: 'wx'`) whose body is
`{pid, cwd, acquired}` JSON, and its hardening reads as a list of specific incidents rather
than defensive habit:

- **Pid-liveness gate.** A dead holder is stolen promptly; a live one is waited on. The
  prior implementation *"unconditionally unlinked WHATEVER lock existed — even a fresh, live
  holder's"*, force-stealing a live writer's critical section.
- **Identity re-confirm before the steal.** `(dev, ino, body)` is snapshotted at decision
  time and re-checked immediately before the steal, so a racer that recreated a fresh lock
  in the decision→steal gap never has its replacement deleted. The *body* is part of the
  identity because `(dev, ino)` alone is defeated by inode reuse.
- **Atomic rename-then-remove.** Only one racer can win the rename; a failed rename means
  someone else already stole it, so the code *"must NOT fall through to a delete."*
- **A deadman ceiling above the timeout.** 60s ceiling over a 10s lock timeout, because the
  lock body carries no start time and liveness alone cannot detect pid reuse. Without it, a
  false-alive holder *"would make `withPlanningLock` throw on every call with no self-heal."*
- **A retry set of transient errnos**, added for a Docker overlay-fs race.
- **A `process.on('exit')` drain** of locks held by this process.

One of the comments is a small masterclass in why swallowed errors are expensive. An earlier
`catch { /* ok */ }` around the directory-create hid a real `EACCES`/`ENOSPC` failure; the
lock write then failed with `ENOENT`; `ENOENT` was in the retry set; the loop burned the full
10-second budget and reported *"a PHANTOM 'held by a live process' contention pointing at a
nonexistent holder."* A swallowed error did not stay small — it re-emerged as a confident
lie about a different subsystem.

The lock is also a **test seam**: `_setLockProbes` injects `isPidAlive`, and
`_setPlanningLockTestHooks` exposes a `beforeSteal` hook that fires after the steal decision
but before the identity re-confirm, so a test can script the exact race window. Designing the
seam at the decision boundary is what makes the concurrency hardening testable at all.

### 2. The checkpoint

`gsd-core/workflows/pause-work.md` writes a two-file handoff: `HANDOFF.json` (machine) plus
`.continue-here.md` (human). Location is context-dependent — phase directory, spike, sketch,
deliberation, or `.planning/` root — and the workflow instructs that when no context is
detectable, the ambiguity is *recorded in the file* rather than resolved by guessing. The
same discipline as `SCOPE`, applied to a prose artifact.

`gsd-core/workflows/resume-project.md` searches for them with a bounded `find` across those
locations plus a legacy fallback.

### 3. The resume gate

`gsd-core/workflows/next.md` runs safety gates before any routing decision. Gate 1 is a hard
stop on `.continue-here.md` existing — a plain `[ -f ... ]` test, deliberately not routed
through the parser, because "did a previous session leave a checkpoint" must be answerable
even when every parse is failing. Gate 2 hard-stops on `status: error` in `STATE.md`.

Then Route 0 detects interrupted execution structurally: scan phases in ROADMAP order and
stop at the first where `plans.length > summaries.length` — *"at least one PLAN.md has no
matching SUMMARY.md."* Interruption is not recorded as a flag someone has to remember to
write. It is inferred from the artifacts a completed plan would have left behind. A `kill -9`
between writing a PLAN and writing its SUMMARY is indistinguishable from a graceful pause,
which is exactly the property you want.

### 4. Scope, enforced in a shell script

The best evidence that the envelope is a real contract rather than internal decoration is
that it crosses out of TypeScript. Route 0's own bash in `gsd-core/workflows/next.md` reads
`.scope` off the `roadmap.analyze` JSON and treats any non-`complete` value as
**scan-failed**, warning and falling through rather than proceeding:

> roadmap.analyze succeeded and returned a well-formed document, but its milestone window did
> not see all of its input, so `.phases[]` is a NON-answer rather than a real empty. Looping
> it would run the invariant over a phase list the scan could not populate and report
> "clean" — the silent disarm.

A safety invariant that reports "clean" because it could not see its input is worse than one
that does not run. The workflow's own acceptance checklist pins this as a requirement.

## Two mutation-safety models for one tree

`.planning/` is mutated under two different safety models, and the split is principled.

| | Read-modify-write (`src/roadmap.cts`) | Migration (`src/roadmap-upgrade.cts`) |
|---|---|---|
| Risk | Concurrent writers | A half-applied structural rewrite |
| Mechanism | `withPlanningLock(cwd, ...)` around every `ROADMAP.md` edit | `git status --porcelain` must be empty before starting |
| Undo | Nothing to undo — the critical section is exclusive | `performedRenames` replayed in reverse (line 631) plus a `fileBackups` map populated by `snapshotFile(...)` before each write (line 622) and replayed from the catch block (line 637) |
| Scope envelope | Yes | No — the module imports neither `planning-scope.cjs` nor the parser |

The stated reason for the second model is the one people get wrong: a `git reset --hard` +
`git clean` rollback *"restores NOTHING for a gitignored `.planning/`"* — and `commit_docs:
false` is the default. **Your undo strategy is only as good as the tracking status of the
files it covers.** Verify that before you make version control your safety net.

`roadmap-upgrade.cts` is structurally outside the whole single-owner regime that
`docs/adr/3180-planning-semantic-model-single-owner.md` establishes: it imports no
`roadmap-parser.cjs` and no `planning-scope.cjs`, carrying its own three module-level regexes
and its own line-indexed parse. It does share the phase-id vocabulary from `src/phase-id.cts`.
That is a defensible boundary for a one-shot migration tool — it should not be coupled to a
model it is migrating *away from* — but see the last tension below for what it cost.

## Where the machinery leaks

The conventions above are unusually well enforced, which makes the gaps informative. These
are the ones with behavioural consequences.

**1. A phase status outside its own vocabulary, and the router cannot see it.** In
`collectAnalyzePhases` (`src/roadmap.cts`), the markdown-table branch (#3577) emits
`disk_status: dirMatchA ? 'ok' : 'no_directory'` at line 514. `'ok'` is not in the heading
branch's vocabulary (`complete|partial|planned|researched|discussed|empty|no_directory`). The
selectors immediately below match `planned|partial` for `currentPhase` and
`empty|no_directory|discussed|researched` for `nextPhase` — so a table-declared phase that
*has* a directory matches **neither**. This is not cosmetic: `gsd-core/workflows/plan-phase.md`
auto-detects which phase to plan by reading `next_phase` off `roadmap.analyze`. The same
branch also sets `tHasContext` via a bare `fs.existsSync(.../'CONTEXT.md')` (line 501) instead
of the canonical dual-form predicate that handles the padded `NN-CONTEXT.md` shape — despite
the branch claiming the "same enrichment contract as headings."

**2. The recovery path produces an internally inconsistent document.** `cmdRoadmapAnalyze`
introduces `effectiveContent` so the #3165 fallback re-scan is reflected downstream, and
switches it when the recovery fires. But `milestones` is still derived from the original
scoped `content`, while the checklist/`missing_details` scan uses `effectiveContent`. When
the recovery path fires, `phases` and `missing_phase_details` come from the fallback document
and `milestones` does not — one payload assembled from two different views of the file.

**3. Containment is asymmetric inside one command.** `src/planning-inspect.cts` builds an
elaborate symlink-containment layer for its own reads: `isWithinRoot` as *"the ONE
containment comparison every containment check in this module shares"* (equality or
`root + path.sep`, never a bare `startsWith`, which *"would wrongly accept a sibling
directory like `.planning-evil/`"*), `fs.realpathSync` on both sides, and containment gates
placed *before* `readdirSync` because it follows directory symlinks. Then
`buildPlanningInspect` calls `buildPlanningSnapshot(cwd)` first thing — and the snapshot's
builders read with plain `fs.readFileSync`/`platformReadSync`, no containment. Snapshot-sourced
values reach the payload having bypassed the boundary the module documents at length. A
security boundary enforced at a consumer rather than an owner protects only the paths that
consumer reads directly.

**4. A raw re-derivation in the module that forbids them.** `src/planning-inspect.cts` line
1170 derives the phase id with an inline `/^(\d+(?:\.\d+)*)/.exec(phase.dir)` — in a module
whose header declares "COMPOSED, NOT RE-DERIVED", and which already imports the canonical
phase-key helpers. A letter-leading directory yields `phaseId: null`, which routes into a
diagnostic reading "Phase directory name carries no recognizable phase number." The
`worstScope` fold two lines below it is textbook; the line above it is the thing the file
exists to prevent.

**5. The migration tool parses a different milestone model than every read path.**
`roadmap-upgrade.cts`'s `MILESTONE_HEADING_RE` matches `^##` only, while the owner grammar in
`src/roadmap-parser.cts` is `^#{1,3}`. A roadmap using `#` or `###` milestone headings is
understood one way by the migrator and another way by everything else. This is the price of
the deliberate isolation described above — worth naming, because "this tool is exempt from the
shared model" and "this tool disagrees with the shared model about the input" are different
claims, and only the first one was decided.

Three smaller items, recorded for honesty rather than consequence: `getMilestoneInfo`'s large
doc comment in `src/roadmap-parser.cts` sits above a different function than the one it
describes; `getMilestonePhaseFilter`'s catch block claims a variable is "left as-is" when it
was reassigned inside the try (stale, not wrong in effect); and `PlanningSnapshot`'s interface
flags that the design doc's own table and prose disagree on the field count, following the
table and saying so *"rather than silently reconciled, since correcting the design doc's prose
is outside this diff's scope."*

## Conventions worth stealing

Independent of this domain, four habits in this code do most of the work:

**Disclose the limit instead of shrinking the scope.** `countMilestoneHeadings` ends with
*"Honest limit: ... it is recorded here rather than hidden."* `roadmap-disk-consistency.cts`'s
header carries a *"KNOWN GAP, disclosed rather than silently dropped"* about dash-shaped ids
never matching checkbox keys. `planning-snapshot.cts` discloses that one of its builders does
not walk a second root. None of these are fixed. All of them are findable by the next person
who hits the symptom.

**Name the direction of the error.** Not "this might be wrong" but which way. The known gap
above errs in *"the conservative direction (a possible false W006, not a suppressed true
one)."* `getMilestonePhaseFilter`'s degraded path falls back to a pass-all filter, described
as *"a safe, non-corrupting (over-inclusive, never under-inclusive) degrade."* And
`buildPlanningTrackedField` treats an `ENOBUFS` overflow from `git ls-files` as **proof of
tracking** rather than a degraded read, because *"`ls-files` only overflows because it had
non-empty output to begin with"* and misclassifying it would drop the finding *"in precisely
the scenario where it matters most."* Direction is what lets a reader decide whether to act.

**Comments as a rejected-alternatives archive.** `countMilestoneHeadings` carries roughly
eighty lines documenting three prior models that each shipped and each broke a real roadmap
shape, before naming the model that survived. Comment density across this module set runs
from 20% to 67% of lines. The payoff is not readability — it is that the next person cannot
re-propose a model that already failed, and can recognise the shape of input that killed it.

**Distinguish relocation from reinvention, explicitly.** Snapshot builders say things like
*"Relocates ... verbatim from `verify.cts`"* and, on a near-miss, *"Confirmed NOT a fit for
`listArchiveVersionDirs` ... that function scans `milestones/*-phases/` DIRECTORIES, a
different target than this field's `milestones/*-ROADMAP.md` FILES — reusing it here would
silently answer the wrong question."* Recording the reuse you *rejected*, and why, is what
stops the next refactor from making the mistake for you.
