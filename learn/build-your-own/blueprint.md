# Minimal blueprint

The [overview](index.md) took GSD Core's choices apart and asked which of them transfer.
This page does the constructive half: it names the **smallest thing worth building**, shows
the files, and sketches each one concretely enough to start on Monday.

The target is not a toy. It is the smallest system that still has the property that makes
GSD Core work at all — *a stage cannot start until the previous stage has written a named
file, and that file is on disk where a human can read it in a diff.* Everything else in the
194 top-level modules of `src/` is machinery bolted onto that spine after evidence arrived.

!!! info "What kind of page this is"
    The file layout, the four pieces and the growth signals are **my synthesis**, not a
    reconstruction of any GSD Core artefact — no path in this repo describes a "minimal
    version". What *is* checkable is every claim about GSD Core used to justify a cut: those
    cite a path and, where a number appears, the command that produced it. Treat the
    blueprint as an argument and the citations as evidence for its premises.

## The bill you are not paying

Worth stating up front, because it is the reason this page exists. Measured against the
current tree:

```bash
find src -name '*.cts' | wc -l                          # 220
find src -name '*.cts' -exec cat {} + | wc -c           # 5,608,277
find tests -name '*.test.cjs' | wc -l                   # 821
find tests -name '*.test.cjs' -exec cat {} + | wc -c    # 22,054,809
ls scripts/lint-*.cjs | wc -l                           # 40
```

Twenty-two megabytes of hand-written test code and forty bespoke lint scripts is what it
costs to hold 71 commands, 35 agents and 19 harnesses still. None of that is wrong. All of
it is a *response* — and if you build the response before you have the thing it responds
to, you have bought the cost with none of the benefit.

## The cut: five steps down to three

GSD Core's contracted loop is five steps producing a single strand,
`CONTEXT.md → PLAN.md → SUMMARY.md → UAT.md` ([The Loop](../loop/index.md)). Two of the five come
out of a v1 without argument, and the repo supplies the argument for both.

**`ship` goes first**, because it already produces nothing. Its `produces:` marker in
`gsd-core/workflows/ship.md` is empty, so the contracted chain terminates there and
re-entry comes from outside the loop entirely — a `roadmap.analyze` query the orchestrator
runs. A step that consumes an artefact and emits none is a *hook point*, not a stage. You do
not need a stage to run `git push`.

**`discuss` folds into `plan`**, because in the declared contract its output `CONTEXT.md`
has a single consumer — `plan`. (Only in the *declared* contract: the overview's warning
box records that executors additionally receive phase CONTEXT above a context-window
threshold, so the real read-set is wider than the contract's. That is itself an argument
for the fold.) A stage whose contracted purpose is to write the input to the next stage is
a section of that stage until you have a reason to separate them — and the reason, when it
comes, is legible: you want to run the two halves in different sessions, or with different
agents.

That leaves **plan → execute → verify**, which is the smallest chain that still has the
property worth having: a planning step whose output is reviewable *before* anything is
built, and a verification step whose output is reviewable after.

## The file layout

Two trees, and keeping them distinct is the one structural decision here that is expensive
to reverse. The first is what you author and ship; the second is what appears inside a
user's project.

```text
your-framework/                  # the thing you build
├── bin/
│   └── yf.mjs                   # ONE entry point; the only writer of state
├── commands/
│   ├── plan.md                  # authoring surface — one file per command
│   ├── execute.md
│   └── verify.md
├── lib/
│   ├── state.mjs                # read/write STATE.md; nothing else touches it
│   └── paths.mjs                # where .work/ is, resolved one way
├── templates/
│   └── STATE.md                 # the shape of state, scaffolded on init
└── install.mjs                  # copy commands/ into the host's config dir
```

```text
some-project/                    # what a user ends up with
└── .work/
    ├── PROJECT.md               # written once, by a human, read every stage
    ├── STATE.md                 # the single mutable file
    └── 01-auth/
        ├── PLAN.md              # produced by plan,    consumed by execute
        ├── SUMMARY.md           # produced by execute, consumed by verify
        └── UAT.md               # produced by verify,  consumed by a human
```

Two rules make this layout worth more than its size.

**The numbered directory is the unit of work.** `01-auth/` is created by `plan` and is
complete when `UAT.md` lands in it. Sequence lives in the directory name, so a resuming
agent can reconstruct position from a `readdir` alone, without parsing anything. GSD Core
keeps this property and it is why its state file can be a *cache* of position rather than
the authority on it.

**`.work/` is flat at depth two.** Resist nesting milestones above phases in v1. Physical
layout that doubles as taxonomy is very cheap to add and very expensive to change, because
every path in every prompt encodes the taxonomy.

## The minimal architecture

```kroki-plantuml
@startuml
skinparam componentStyle rectangle
skinparam shadowing false
skinparam defaultTextAlignment center
skinparam backgroundColor transparent

actor "Human" as H

package "Agent host (Claude Code, Cursor, ...)" {
  component "loaded command prompt\nplan.md / execute.md / verify.md" as PROMPT
}

package "your-framework (shipped)" {
  component "commands/*.md\nauthoring surface" as CMD
  component "bin/yf\nthe only deterministic code\nthe only writer of STATE.md" as BIN
}

package ".work/ (per project)" {
  component "STATE.md\nYAML frontmatter = machine\nmarkdown body = agent" as ST
  component "01-auth/\nPLAN.md -> SUMMARY.md -> UAT.md" as ART
}

CMD --> PROMPT : installed once,\ncopied verbatim
H --> PROMPT : /plan auth
PROMPT --> BIN : shells out for anything\nthat must be exact
PROMPT --> ART : writes the stage artefact\n(the model's own prose)
BIN --> ST : read-modify-write
ST --> PROMPT : read back as context
ART --> PROMPT : previous stage's output\nis this stage's input
ART --> H : reviewable in a diff

note bottom of BIN
  v1 rule: if the model can write it,
  the model writes it. Code exists only
  for things that must be EXACT --
  position arithmetic and STATE.md writes.
end note
@enduml
```

## Piece 1 — one command surface

One directory of markdown files, one file per command, one naming prefix. That is the whole
mechanism.

The prefix earns its place immediately, and for a reason that is easy to miss:
the [overview](index.md) records that GSD Core separates its two surfaces with a
greppable token (`gsd:` for commands, `gsd-` for skills) so that *a linter aimed at the
wrong surface finds nothing rather than finding something wrong.* Pick your token on day
one; renaming it later means touching every prompt that mentions a command by name.

The rule that keeps this surface cheap: **`commands/` is the sole authoring surface, and
everything else in the shipped tree is output.** In v1 there is no "everything else" — you
ship the directory as-is. That is fine, and it is exactly why v1 needs no generator. The
moment a second surface appears, you are at growth stage 1 below.

## Piece 2 — one spec format

A spec is frontmatter plus a body. The frontmatter is machine-read, the body is prompt text
handed to the model close to verbatim.

```markdown
---
name: plan
description: "Turn a phase description into a reviewable PLAN.md"
argument-hint: "<phase-slug>"
allowed-tools: [Read, Write, Bash, Glob, Grep]
---

<objective>
Produce .work/<NN>-<slug>/PLAN.md for the named phase. Do not write code.
</objective>

<inputs>
- `.work/PROJECT.md` — required. Stop if missing.
- `.work/STATE.md` — required. Stop if missing.
</inputs>

<procedure>
1. Run `yf state read` and confirm status is `ready-to-plan`. If not, stop and say why.
2. Read PROJECT.md.
3. Write PLAN.md using the section order below. Every task gets a verifiable
   done-condition — something a later stage can check without asking you.
4. Run `yf state set status=planned plan=<NN>-<slug>`.
</procedure>

<output_contract>
PLAN.md MUST contain, in this order: ## Goal, ## Tasks, ## Done when, ## Out of scope.
On success end with `## PLAN WRITTEN`.
If you cannot finish, end with `## CONTINUE_REQUIRED` and say what is blocking.
</output_contract>
```

Three things in that sketch are worth more than the rest, and all three are free.

**Reconstruct frontmatter from a whitelist, never filter it.** Decide which keys ship. The
default for any key you add later is "does not ship", which means an authoring-only field
(a `requires:` you lint against, a note to yourself) can never leak into a runtime contract
by accident.

**Type the non-terminal outcome.** `## CONTINUE_REQUIRED` is the highest-value line in the
whole file. A stage that can only say "done" will say "done" when it is not. Naming the
in-between state is what lets a caller loop on progress instead of on hope.

**Declare an output contract, not a style preference.** "These sections, in this order" is
checkable by the next stage with a `grep`. "Be thorough" is not.

## Piece 3 — one state file

One file, two audiences. Frontmatter carries scalars a program reads; the body carries prose
an agent reads. GSD Core's version lives at `gsd-core/templates/state.md` and describes
`.planning/STATE.md` as "the project's living memory". Reduced to what v1 needs:

```markdown
---
status: ready-to-plan        # ready-to-plan | planned | executing | ready-to-verify | done
phase: 01-auth               # or null
updated: 2026-08-21
---

# State

## Current position

Phase: 01-auth
Status: ready-to-plan
Last activity: 2026-08-21 — planned auth, awaiting execute

## Decisions

- Sessions are server-side; no JWT. (01-auth, 2026-08-21)

## Do not retry

- Passwordless via magic link — rejected, needs an email provider we do not have.
```

The frontmatter/body split is the load-bearing part. Scalars in frontmatter can be read and
rewritten by a program without understanding the file; prose in the body can be grown by an
agent without a schema migration. You get a machine contract and an open-ended log in one
file that still diffs cleanly.

Two cheap habits to adopt now, both of which GSD Core arrived at the long way:

- **Label every artefact IMMUTABLE, OVERWRITE or APPEND, and publish the read-order** a
  resuming agent should follow. In v1 that is four lines in your README. It is the entire
  content of `## Do not retry` above — negative knowledge is the highest-value thing state
  carries, because a rejected approach that is not written down is one an agent will
  cheerfully propose again.
- **One writer.** `bin/yf` writes `STATE.md`. Prompts *ask* it to, by shelling out to
  `yf state set …` — they never open `STATE.md` with an editing tool themselves. The
  converse also holds: prompts write the stage artefacts (`PLAN.md`, `SUMMARY.md`,
  `UAT.md`) and `bin/yf` never touches those. Two file classes, one writer each, and the
  boundary stated in both directions. The rule costs nothing while you dispatch one agent
  at a time, and it is the thing you cannot retrofit cheaply once you fan out.

You do not need locking in v1. The concurrency machinery in `src/state.cts` — 261 KB, the
largest module in the repo — is a response to parallel dispatch, not to state files.

## Piece 4 — one loop

Three steps, single strand, each consuming exactly what its predecessor produced:

| Step | Reads | Writes | Terminal marker |
|---|---|---|---|
| `plan` | `PROJECT.md`, `STATE.md` | `PLAN.md` | `## PLAN WRITTEN` |
| `execute` | `PLAN.md` | `SUMMARY.md` | `## EXECUTION COMPLETE` |
| `verify` | `SUMMARY.md` | `UAT.md` | `## VERIFIED` / `## GAPS FOUND` |

Those marker names are mine and deliberately do not reuse GSD Core's. Note in particular
that GSD Core's `## PLAN COMPLETE` is the **executor's** completion marker, not the
planner's (`gsd-core/references/agent-contracts.md:97`) — a good illustration of why marker
vocabulary is worth fixing before you have three commands rather than after.

Keep the strand single. GSD Core's contract declares no fan-in and no step reading two
upstream artefacts, and [The Loop](../loop/index.md) is explicit that the narrowness is the
feature: *it is what makes the chain checkable at all.* A step that reads two upstream files
has an ordering constraint you now have to enforce somewhere.

The one closure worth building in v1 is the gap edge — `verify` writing `## GAPS FOUND` and
`plan` accepting a gaps file as input. Without it your loop is a pipeline that runs once.
With it, it is a loop.

And believe the filesystem over the transcript. GSD Core detects executor completion from
SUMMARY existence and git state rather than matching the agent's declared success string,
and `gsd-core/references/agent-contracts.md:97` records that as deliberate — "intentional
behavior, not a mismatch". Your terminal markers are for the human reading the file; your
control flow should key on whether the file is there.

## Deliberately not in v1

The [overview's Part 2](index.md#part-2-reuse-adapt-drop) answers *should you ever build
this*, scored across three team sizes. This table answers a narrower question — **what does
v1 omit, and what observable event says the omission has expired?** The point of the third
column is that it is an event you will notice happening, not a size you have to estimate.

| Feature | Why not in v1 | The event that changes the answer |
|---|---|---|
| A generator + `--check` | With one authoring surface there is nothing to generate | Two committed files must agree on one list of names |
| Subagents | A subagent is a context boundary; you do not have a context problem yet | One stage's inputs stop fitting, or you want a bounded job to fail without poisoning the session |
| An agent roster / model catalogue | Three agents fit in your head | You add an agent and cannot say from memory which commands can spawn it |
| Hooks | A hook is enforcement, and you have not yet been disobeyed | A rule written in a prompt is violated after the model has read it |
| Capability packs | An extension contract only pays if third parties exist. Solo, you are the third party | Someone who cannot edit your repo needs to add a stage |
| An attachment-point contract | Three steps have an obvious order; you can read it off the directory | A step needs to run between two existing steps and you cannot say where |
| A second harness | Portability is a product decision, not an architecture one | A user who will not switch hosts asks for it |
| Locking / pid-liveness gates | Driven by parallel dispatch, not headcount | You dispatch two agents that can write in the same window |
| A milestone layer above phases | Physical nesting encodes taxonomy into every path | Your phase list stops fitting on one screen *and* phases start grouping naturally |
| A plugin/registry format | A registry describes a system; in v1 the directory *is* the description | Your distribution channel requires a manifest |

Every row above is something GSD Core has and v1 does not. That is not a criticism of GSD
Core — each one is traceable to a pressure it actually met. It is a warning about copying a
mature framework's *shape* before meeting its pressures.

## The growth path

Each stage below is triggered by a condition, not by a schedule. Build in this order,
because each stage assumes the previous one exists.

### First — a generator with a `--check` mode

**Signal:** two committed files must agree on one list of names, and you have just updated
one and forgotten the other.

In my judgement this is the highest-return addition on the list, and the cost is the thing
to notice: it is
**constant**. One generator plus one `--check` step does not grow with team size or corpus
size, while the drift it prevents starts accruing on day one — because your framework's
second maintainer is you in six months, and you will not remember that two files were
supposed to agree.

GSD Core's own behaviour is the argument. `scripts/check-contract-drift.cjs` guards the
agent roster across several files, and running it right now prints
`ok check-contract-drift: 35 agents, 38 markers, 0 violations`. That is one of many: 14
`--check` invocations are chained into a single npm script
(`node -e "…split('&&').filter(s => s.includes('--check')).length"` over
`package.json`'s `lint:generated-sync` → 14), alongside 40 bespoke `scripts/lint-*.cjs`. A
maintainer does not write forty guards for a problem that is not recurring.

The shape to copy is a three-mode CLI, and `scripts/gen-plugin-skills.cjs` is the cleanest
example in the repo at 117 lines including its docstring:

```javascript
// node gen.mjs           -> print what it WOULD write   (safe default)
// node gen.mjs --write   -> write the generated tree
// node gen.mjs --check   -> exit 1 if the committed tree is stale
```

Three details in that file are what make the pattern hold, and each is one line of thought:

- The **default mode is a dry run.** `--write` is opt-in, so a generator invoked by mistake
  cannot destroy the tree.
- `--check` compares **byte-for-byte** (`gen-plugin-skills.cjs:90`), not semantically. A
  fuzzy comparison is a comparison you will eventually have to debug.
- `--check` also walks the **output** directory for entries with no source
  (`gen-plugin-skills.cjs:95–102`). A generator that only checks the forward direction
  passes happily while a deleted source leaves an orphan behind.

Then commit the generated tree. It feels wrong and it is correct: npm and plugin installs
run no build step, so an uncommitted artefact is a missing artefact — and staleness becomes
a CI failure rather than a merge surprise.

!!! warning "A generator relocates drift, it does not remove it"
    `--check` proves the artefact matches its source. It says nothing about whether the
    source is *true*. `gen-plugin-skills.cjs` byte-compares 71 skills against 71 commands
    and is silent on whether the CLI syntax those commands quote is correct. Know which of
    the two problems you just solved.

### Second — deterministic code behind a CLI

**Signal:** you have fixed the same parsing or arithmetic bug in a prompt twice.

That is the whole test. A model asked to compute "plan 3 of 7, so 43%" will get it right
most times, and the times it does not are not reproducible. Move it into `bin/yf` and it is
wrong *reproducibly* — and a reproducible bug is one you can fix once.

GSD Core wrote deterministic code for five categories, and the list transfers: parsing and
serialising the model's own markdown output, state transitions, concurrency, platform
portability, and trust boundaries. In v1 you need the first two. Note what is *not* on that
list — what to build, whether a plan is sound, how to phrase a review. Those stay in prompts
permanently.

Copy the categories; do not copy the shape. 194 top-level modules in one flat `src/` is
what happens when nothing pushes back, and eight of them now exceed 100 KB
(`ls -S src/*.cts | head -8`).

### Third — subagents with typed returns

**Signal:** a stage's inputs stop fitting in one context, *or* you want a job whose failure
does not contaminate the session that dispatched it.

Both are real reasons and neither is "subagents are good architecture". A subagent is a
context boundary with a cost: you now have a return channel that can lie.

The contract pieces are free and worth taking whole on the first subagent you write: a
typed non-terminal return, loops bounded on *progress* rather than iteration count, named
terminal markers instead of silent give-up, and completion inferred from side effects. The
35-agent roster with its three-way alignment checker is not free — that exists because at 35
agents nobody can hold the roster in their head, and it is pure overhead at three.

One idea from that machinery is worth copying at any size because it inverts an intuition:
**make something other than the directory listing your roster.** GSD Core derives the
expected roster from a model catalogue rather than from `agents/`
(`src/agent-install-check.cts:159`), so a file dropped in with no catalogue entry is
invisible. That reads as a bug until you see what it buys — "I added a file and nothing
happened" becomes impossible to ship silently.

### Fourth — a hook, and only for a rule that has been broken

**Signal:** a rule stated in a prompt was violated by a model that had read it.

[The hook system](../runtime/hooks.md) (line 397) states the test better than I can:

> Does this rule survive a model that has read it, understood it, and decided it does not
> apply?

If it does, prose is enough. If it does not, you need something outside the model's
judgement. The evidence from this repo is that hooks arrive as *retrofits* — the site traces
several to specific incident numbers — and that is the healthy pattern, not a failure of
foresight. A hook written before the violation is a guess about which rule will be broken.

Two things to decide when you write the first one, because both age silently:

- **Name the enforcement host in your README.** If you ever support more than one host,
  projection of artefacts scales and enforcement does not. Decide which host gets teeth and
  say so, rather than discovering it later from a support thread.
- **Audit the hook's scope as hard as its rule.** The failure mode of a mature guard is not
  a false alarm, it is a **vacuous pass** — a directory constant that no longer covers the
  tree, an empty set that iterates zero times, a regex class that stopped matching. A guard
  that reports zero because it looked nowhere is worse than no guard, because it is
  believed. Print what you scanned, not just what you found.

### Fifth — packaging, extension contracts, more harnesses

**Signal:** somebody who cannot edit your repository needs to change its behaviour.

Until then you are the third party, and you can read the ordering off the files. The
overview's Part 2 covers the trade in full; the only thing to add here is a sequencing
warning: when you do build an abstraction over host differences, **extract it from two
implementations rather than one.** GSD Core's hook bus parameterised what its first harness
varied and hardcoded what it did not, so the second harness produced a sibling rather than a
second caller. One data point does not define an axis.

## Where this leaves you

The blueprint is four files of framework and three files of state, and the honest claim for
it is not that it is sufficient — it is that every piece it omits has a *signal*, and you
will recognise the signal when it arrives.

The order matters more than the contents. Generators before code, code before subagents,
subagents before hooks, hooks before packaging — because each stage produces the thing the
next one guards. Build them in the other direction and you get GSD Core's shape without its
reasons.
