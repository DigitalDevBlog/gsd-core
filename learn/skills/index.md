# Skills

A GSD skill is a markdown file with YAML frontmatter. There is no interpreter, no
scheduler, no runtime that steps through it. The only thing that "executes" a `SKILL.md`
is a language model reading it as part of its context window. Everything interesting in
this layer follows from that constraint: if you cannot enforce a rule with code, you have
to encode it as text the model will read, and you have to build a compiler and a linter
around the text to keep it honest.

This page takes the skill layer apart: what the frontmatter contract actually buys you,
how `commands/gsd/` is mechanically projected into `skills/`, how dispatch works, and what
"executable markdown" means when there is nothing to execute it.

## One source, two surfaces

`skills/` is not hand-written. Every file in it is generated.

`scripts/gen-plugin-skills.cjs` reads the directory listing of `commands/gsd/`, and for
each `<stem>.md` produces `skills/gsd-<stem>/SKILL.md` by calling
`convertClaudeCommandToClaudeSkill` from the compiled converter. The mapping is exactly
1:1 and complete: 71 files in `commands/gsd/`, 71 directories in `skills/`, each holding a
single `SKILL.md`, with no orphan in either direction.

The generator is deliberately destructive:

```js
fs.rmSync(SKILLS_DIR, { recursive: true, force: true });
fs.mkdirSync(SKILLS_DIR, { recursive: true });
```

`--write` deletes the whole tree and regenerates from scratch. `--check` compares
byte-for-byte *and* flags any `gsd-*` directory with no source command — so a deleted
command cannot leave a zombie skill behind.

!!! note "Generated, but committed"
    All 71 generated `SKILL.md` files are tracked in git. That looks redundant until you
    notice why: the npm package and plugin-only installs must work without running a
    build. So the repo commits the build output and makes staleness a CI failure instead
    of a merge-time surprise. `package.json` wires `gen:plugin-skills -- --write` into
    `build` (after `build:lib`, on which it hard-depends) and `gen-plugin-skills.cjs
    --check` into `lint:generated-sync`, itself inside `lint:ci`.

The design decision worth stealing: **the command `.md` is the compiler input, and every
runtime artifact is output.** Claude skills are only one target. `src/runtime-artifact-conversion.cts`
exports converters for Kimi, Cursor, Windsurf, Augment, Trae, CodeBuddy, Cline, Codex,
opencode, kilo, Qwen, Hermes and ZCode as well. You write the prompt once; the build
projects it into every host's dialect.

## The frontmatter contract, field by field

The single most important thing to understand about GSD's frontmatter is that it is
**layered by audience**. Some fields are a runtime contract with the host and survive
conversion. Others are a build/install/lint contract and are deliberately destroyed on the
way out.

`convertClaudeCommandToClaudeSkill` (`src/runtime-artifact-conversion.cts`) does not copy
frontmatter. It *reconstructs* it, extracting exactly five fields and dropping everything
else on the floor:

| Field | What it buys | Survives conversion? |
|---|---|---|
| `name` | The token the host registers the skill under and completes on. | Rewritten, not copied — see below. |
| `description` | The one-line summary the host shows in listings, and the text a router matches intent against. | Yes, re-emitted through `yamlQuote()`. |
| `argument-hint` | Autocomplete/usage string. In practice, the flag documentation. | Yes — **unless empty**. |
| `allowed-tools` | The capability grant. Preserved verbatim as a YAML block list. | Yes, byte-for-byte. |
| `context` | `context: fork` runs the skill in an isolated subagent window (#769). | Yes — but **no command in `commands/gsd/` declares it**. Converter capability only. |
| `agent` | Re-emitted verbatim by the converter (`src/runtime-artifact-conversion.cts:466,509`). Its binding semantics are **undocumented anywhere in the repo** — no ADR, no comment, no usage. | Yes — but 0 of 71 commands declare it. |
| `effort` | Sets `output_config.effort` for the invocation. | **No — stripped.** |
| `requires` | Declared dependency graph, consumed by the linter and the install profiles. | **No — stripped.** |
| `type` | Present on five commands. Consumed by nothing. | **No — stripped.** |

Two of these strips are worth dwelling on.

### `effort` is stripped for a cache reason, not a cleanliness reason

Six commands carry `effort:` — `effort: max` on `commands/gsd/autonomous.md`,
`commands/gsd/execute-phase.md` and `commands/gsd/plan-phase.md`; `effort: low` on
`commands/gsd/next.md`, `commands/gsd/progress.md` and `commands/gsd/stats.md`. Zero of
the 71 `SKILL.md` files carry it.

The converter comments explain why, and the reasoning is a performance argument rather
than a style one:

> `effort:` is intentionally NOT emitted into skill frontmatter — a static effort value
> changes `output_config.effort` on invocation and invalidates the caller's prompt cache
> at both scope boundaries.

So the field is live on the command surface and deliberately dead on the skill surface.
This is a good example of a general principle: a frontmatter field is not "metadata," it
is an instruction to a specific host, and a field that is harmless in one delivery
mechanism can be expensive in another.

### `requires` is a build graph, not runtime state

57 of the 71 commands declare `requires:`; 14 omit it entirely (`commands/gsd/help.md` is
one). None of the skills have it. It exists to be linted, and
`scripts/lint-skill-deps.cjs` enforces it two ways:

1. **Frontmatter/body consistency.** Any `gsd:<stem>` reference in a command body must
   appear in that command's `requires:` list.
2. **Profile closure.** For every non-full install profile, every dependency of every
   included skill must also be in the closure — so you cannot ship a partial install where
   a skill routes to something that isn't there.

The self-reference case is handled with one line, `if (ref === stem) continue;`, which is
why `commands/gsd/graphify.md` can legally declare `requires: []` while its body mentions
`gsd:graphify` three times — every one of those references is to itself.

(Picking the right example matters here: `extractBody()` strips frontmatter before
`extractBodyReferences()` runs, so a command whose only `gsd:` occurrence is the
`name:` key never exercises the guard at all. Six commands actually do — `explore`,
`fast`, `graphify`, `progress`, `quick` and `workstreams`.)

!!! tip "The linter only works on the command surface — by construction"
    `extractBodyReferences` in `scripts/lint-skill-deps.cjs` matches `/(?:\/?)gsd:([a-z0-9_-]+)/g`
    — the **colon** form. Since the converter rewrites every skill body to the hyphen
    form, pointing this linter at `skills/` would find exactly zero references. The
    authoring/lint surface and the shipped/registered surface are separated by nothing more
    than a punctuation character, and that separation is load-bearing.

## The colon/hyphen spine

Commands live in the colon namespace (`name: gsd:plan-phase`). Skills live in the hyphen
namespace (`name: gsd-plan-phase`). This is not cosmetic — it is the mechanism that keeps
the two surfaces from contaminating each other.

Two things happen at conversion time:

- `skillFrontmatterName` derives the emitted `name:` **purely from the directory name**
  (`src/runtime-artifact-conversion.cts`), and the generator builds that directory name as
  `PREFIX + stem` — i.e. from the *filename*, never from the declared `name:`.
- `transformContentToHyphen` rewrites every `/gsd:<cmd>` and `gsd:<cmd>` occurrence **in
  the body** to the hyphen form.

The visible effect: `commands/gsd/plan-phase.md` says `produced by /gsd:review`;
`skills/gsd-plan-phase/SKILL.md` says `produced by /gsd-review`. The stated reason is that
the installed body must match the hyphen `name:` that Claude Code, Qwen and Hermes register
under (#2808, #3583).

### The rewrite table is the directory listing

This is the most conceptually surprising piece of the build. `transformContentToHyphen`
needs to know which tokens are legal command names, and it gets that list from
`readGsdCommandNames()` in `src/command-roster.cts`:

```ts
return fs.readdirSync(COMMANDS_DIR)
  .filter((f: string) => f.endsWith('.md'))
  .map((f: string) => f.replace(/\.md$/, ''));
```

**The set of legal namespace tokens is defined by which `.md` files exist on disk.** Adding
a command file changes how every other command's prose is rewritten. The markdown files
are not just the compiler's input; the directory listing is the compiler's symbol table.

### The twin diff is tiny and entirely mechanical

Diff any command against its generated skill and you get only six kinds of change:

1. `name:` colon → hyphen
2. `description:` gains YAML quoting
3. `effort:` dropped
4. `requires:` dropped
5. a blank line inserted after the closing `---`
6. body colon → hyphen rewrites

Across all 71 pairs the largest diff is 23 changed lines (`complete-milestone`), but that
is an outlier: the median is 6, and 46 of the 71 pairs fall between 4 and 6 changed lines. The entire `<objective>` / `<execution_context>` / `<runtime_note>` / `<context>`
/ `<process>` prose is carried through byte-for-byte. **The conversion is a renaming pass,
not a translation.** If you are building something similar, that is the property to aim
for: the more your converter merely renames, the less there is to go wrong per target
host, and you will have a lot of target hosts.

## What makes a markdown file executable

Here is the central question. A model reads a `SKILL.md` and then *does things*. There is
no interpreter. So what is doing the work?

### The four-block skeleton is a calling convention

Command bodies are not free prose. They follow an XML-ish skeleton:
`<objective>` (61 of 71 files), `<execution_context>` (60), `<context>` (45),
`<process>` (57). Each block has a fixed job:

- `<objective>` — what this run is for, stated before anything else is loaded.
- `<execution_context>` — nothing but bare `@`-include lines. This is the *import block*.
- `<context>` — binds runtime values, almost always `Arguments: $ARGUMENTS`.
- `<process>` — the body. Frequently two sentences.

`commands/gsd/help.md` is the purest specimen. Its entire `<process>` is one line:

```
Follow ~/.claude/gsd-core/workflows/help.md with $ARGUMENTS.
```

That is the whole program: an include plus a variable binding. The command holds objective,
flags and a pointer; the actual procedure lives in `gsd-core/workflows/`, which holds 88
top-level `.md` files and 152 including its `steps/`, `modes/` and `templates/`
subdirectories. `commands/gsd/execute-phase.md` states the split as a budget:
`Context budget: ~15% orchestrator, 100% fresh per subagent.`

**Thin orchestrator, fat workflow.** The command is a manifest. The workflow is the code.

### `@` is the only operator

There is no conditional, no loop, no function call, no variable other than `$ARGUMENTS`.
The entire composition mechanism is textual inclusion at load time. The most-included file
is `@~/.claude/gsd-core/references/ui-brand.md` — which lives in the repo at
`gsd-core/references/ui-brand.md` — pulled into 20 commands. It is a shared output contract
distributed by inclusion rather than inheritance, because inclusion is the only mechanism
available.

And the includes address themselves through *install* paths, not repo paths:
`@~/.claude/gsd-core/workflows/<stem>.md` is where the file will live after installation,
not where it lives in the source tree. Which means the build needs a linker.
`scripts/lint-command-contract.cjs` is it:

- **Rule 4** resolves every `@`-ref back onto the repo layout and fails if the target is
  missing. (Symbol resolution.)
- **Rule 5** requires each `@`-ref be alone on its line, with no trailing prose. (The parse
  is line-oriented, so this is a grammar constraint.)
- **Rule 6** requires every `gsd-core/workflows/*.md` be transitively reachable from at
  least one loader under `commands/`, `agents/` or `skills/`. (Dead-code elimination.)

A markdown prompt system with no interpreter still has link errors, and it still has
unreachable code. GSD's answer is to build the compiler passes anyway.

### `allowed-tools` is the capability grant

This is the field that most clearly makes frontmatter an interface rather than
documentation. `commands/gsd/plan-phase.md` grants Read, Write, Bash, Glob, Grep, Agent,
AskUserQuestion, WebFetch and `mcp__context7__*`. The namespace routers get exactly
`[Read, Skill]` — because their entire program is "read a table, dispatch." The tool list
*is* the sandbox, and it is expressed in the same YAML the host reads for autocomplete.

### Semantics you cannot enforce get asserted redundantly

This is the thesis. When you have no interpreter to check anything, the only enforcement
mechanism left is repeating yourself at the model.

`commands/gsd/execute-phase.md` devotes a block to insisting its own documentation is not
self-activating:

```
Flag handling rule:
- The optional flags documented below are available behaviors, not implied active behaviors
- A flag is active only when its literal token appears in `$ARGUMENTS`
- If a documented flag is absent from `$ARGUMENTS`, treat it as inactive
```

Then, 27 lines further down — *after* the flag list the rule is warning about — it restates
the same instruction a fourth time, now enumerated per flag:

```
- `--gaps-only` is active only if the literal `--gaps-only` token is present in `$ARGUMENTS`
- `--interactive` is active only if the literal `--interactive` token is present in `$ARGUMENTS`
- If none of these tokens appear, run the standard full-phase execution flow with no flag-specific filtering
- Do not infer that a flag is active just because it is documented in this prompt
```

This exists because a model that reads a list of flags is tempted to use them. There is no
argument parser to say no, so the guard is a natural-language assertion — a generic
three-bullet rule stated before the flags, and a per-flag restatement stated after them.

Similarly, ten commands carry a `<runtime_note>` block telling the model that VS Code
Copilot's `vscode_askquestions` is the equivalent of `AskUserQuestion`. Cross-host
portability is handled by *instructing the model*, not by capability negotiation.
`commands/gsd/plan-phase.md`'s version adds a defensive clause `execute-phase`'s does not:
`Do not skip questioning steps because AskUserQuestion appears unavailable.`

!!! warning "The guard is duplicated, and the include mechanism cannot absorb it"
    This guard appears in exactly two files — `commands/gsd/execute-phase.md` and
    `commands/gsd/docs-update.md` — and comparing them exposes a real limitation.
    `commands/gsd/docs-update.md` carries the same shape in its `<context>` block, headed
    `**Available optional flags (documentation only — not automatically active):**`, with
    its own per-flag enumeration for `--force` and `--verify-only` and the closing line
    **byte-identical** to `execute-phase`'s.

    So the invariant part is genuinely shared text, and this repo has a mechanism for
    shared text — `@`-inclusion, which 20 commands use for `ui-brand.md`. But the
    surrounding enumeration is parameterized by flag name, and `@`-inclusion has no
    parameters. It splices a file in verbatim; it cannot take arguments. Extracting the
    one invariant line while leaving the parameterized bulk duplicated would arguably make
    things worse.

    This is the sharpest limit of the whole design. The composition primitive is textual
    inclusion, which means **anything that varies per call site cannot be factored out at
    all** — it has to be copy-pasted and kept in sync by hand, in the one layer where no
    linter is watching. A prompt system with no parameterized include has no functions,
    only headers.

### Defaults encoded as negative flags

One more prompt-layer technique worth naming. `commands/gsd/plan-phase.md` carries 22
documented flags in a single `argument-hint` string (`execute-phase` carries 4), and the
list shows behaviour migrating from opt-in to default by *inverting the flag rather than
removing it*:

- `--no-tracer` opts **out** of tracer-first decomposition; horizontal layering is
  described in-file as "the legacy default."
- `--no-reversibility-gates` suppresses a checkpoint that one-way-door decisions get by
  default.
- `--mvp` "no longer *turns it on*" — it is now enrichment on top of a default it used to
  enable.

The `--no-reversibility-gates` entry contains the sharpest sentence in the flag docs:
*"the flag changes what stops the run, not what the plan remembers."* Safe by default,
escapable by explicit negative flag, with the record kept either way.

## Dispatch: the `ns-*` namespace routers

Six files in `commands/gsd/` are a different artifact kind sharing the directory:
`ns-context`, `ns-ideate`, `ns-manage`, `ns-project`, `ns-review`, `ns-workflow`.

They have no `<objective>`, no `<execution_context>`, no `<process>`. Their entire body is
a markdown table:

```markdown
| User wants | Invoke |
|---|---|
| Review code for quality and correctness | gsd-code-review |
| Auto-fix code review findings | gsd-code-review --fix |
| Debug a failing feature or error | gsd-debug |
...

Invoke the matched skill directly using the Skill tool.
```

Their `description` is not a sentence but a pipe-delimited keyword line —
`"quality gates | code review debug audit security eval ui"` for `commands/gsd/ns-review.md`,
`"workflow | discuss plan execute verify phase progress"` for `commands/gsd/ns-workflow.md`.
Their `allowed-tools` is exactly `[Read, Skill]`.

That is the whole dispatch mechanism: **the routing table is a markdown table, and the
matcher is the model.** The `description` keywords are the index; the table is the
dispatch map; `Skill` is the only capability needed to act on a match. These six files are
part of why the four-block skeleton counts above fall short of 71 — they are a second
artifact kind that happens to share the directory, and the build treats them identically to
every other command because nothing in the frontmatter distinguishes them.

Routers also carry their own migration history in user-facing prose.
`commands/gsd/ns-review.md` states `gsd-code-review-fix was absorbed by gsd-code-review --fix in #2790`,
and `commands/gsd/ns-workflow.md` spends five lines explaining that "Sub-skill names below
are post-#2790 consolidated targets" and that "The reclaimed `gsd-next` target is the
state-aware smart-entry launcher, not the retired workflow-advance command." Issue numbers
and retirement notices ship *inside the runtime prompt*, because the model reading it needs
to not route to a command that no longer exists.

!!! note "'Router' is overloaded in this repo"
    The `ns-*` files are prompt-level routers. Separately, `src/planning-command-router.cts`
    and `src/agent-command-router.cts` are CLI subcommand dispatchers for `gsd-tools` — a
    different layer entirely, dispatching argv rather than intent. They are worth a glance
    for two conventions: strictness argued from consequence (`src/planning-command-router.cts`
    rejects extra args because "silently ignoring an argument a caller believed was scoping
    the query would return a full-project snapshot presented as a scoped one — a confidently
    wrong answer"), and TypeScript unions mirrored as frozen runtime objects
    (`AGENT_FAILURE_CLASSES` in `src/agent-command-router.cts`, consumed for real by
    `src/commands.cts`) because union literals erase at runtime and a second surface would
    otherwise re-declare them and drift.

    They also disagree with each other. `src/planning-command-router.cts` routes through the
    shared `routeCjsCommandFamily` adapter (`src/cjs-command-router-adapter.cts`) and calls
    `error(...)` then explicitly `return`s; `src/agent-command-router.cts` hand-rolls the
    same behaviour with a bare `if (subcommand !== 'classify-failure')` and falls straight
    through after `error(...)`. Same codebase, same `io.error`, two different assumptions
    about whether it terminates. And `src/planning-command-router.cts` claims the router
    signature is "`{ args, cwd, raw, error }` — identical to the other host routers," which
    is not true: `gsd-core/bin/gsd-tools.cjs` registers planning directly in its table but
    needs a wrapper for agent that silently discards `cwd` and the injected `error`.

## Surprises and tensions

These are the places where the design leaks, and they are more instructive than the parts
that work.

### The filename wins over the declared `name:`

`commands/gsd/ns-review.md` declares `name: gsd-quality`. But
`skills/gsd-ns-review/SKILL.md` ships `name: gsd-ns-review`, because the generator keys off
the filename stem (`PREFIX + stem`) and `skillFrontmatterName` returns the directory name
unchanged. Every `ns-*` file has this mismatch (`ns-context`→`gsd-context`,
`ns-ideate`→`gsd-ideate`, `ns-manage`→`gsd-manage`, `ns-project`→`gsd-project`,
`ns-workflow`→`gsd-workflow`), but `ns-review` is the only one where the declared name is
not even a rename of the stem.

The skill surface provably ignores `name:`. Whether the *command* surface honours the
declared `name:` or its filename is not resolved by the source — which means six files
declare an identity that at least one of their two surfaces does not use.

### `argument-hint: ""` is declared six times and never survives

All six `ns-*` commands declare `argument-hint: ""`. The converter emits the field only
under `if (argumentHint)` — and the empty string is falsy, so the field silently vanishes.
Confirmed against `skills/gsd-ns-review/SKILL.md`, which has no `argument-hint` line at
all. Six files declare a field that the build cannot distinguish from absence.

### The converter exists twice

`bin/install.js` line 64 already does
`require('../gsd-core/bin/lib/runtime-artifact-conversion.cjs')`. Yet `bin/install.js` also
defines its **own** `convertClaudeCommandToClaudeSkill`, with the same signature and a
byte-identical doc comment, and re-exports it.

This is precisely the drift class the repo has a dedicated linter for.
`scripts/lint-compiled-artifact-sync.cjs` opens by narrating a real incident:

> #2653: `gsd-core/bin/lib/api-coverage.cjs` sat four days behind its `.cts` after PR #2551
> changed the source without regenerating the artifact, shipping a module that silently
> lacked the entire #2366 fix while CI stayed green.

`src/runtime-artifact-conversion.cts` itself calls out the same pattern around another
export, describing it as "the exact drift class this PR exists to reduce." The awareness is
documented; the duplicate survives.

### An arity mismatch TypeScript structurally cannot see

`scripts/gen-plugin-skills.cjs` calls the converter with **five** arguments:

```js
const converted = conversion.convertClaudeCommandToClaudeSkill(src, skillName, RUNTIME, cmdNames, true);
```

The function is declared with **four** parameters —
`function convertClaudeCommandToClaudeSkill(content, skillName, runtime = null, cmdNames = null)`
— identically in `src/runtime-artifact-conversion.cts` and in `bin/install.js`. The trailing
`true` is dropped on the floor.

The reason no tooling catches it is architectural, not accidental. The call goes through a
`require()` of `gsd-core/bin/lib/runtime-artifact-conversion.cjs`, which is a build output
and is gitignored — it does not exist in a clean checkout. TypeScript never sees the call
site, because the call site is a `.cjs` script reaching across a `require()` boundary into
an emitted artifact. **Build-at-publish buys you a single source of truth and costs you
cross-boundary type checking.** That is the trade, stated plainly.

### A contract linter's own header is stale

`scripts/lint-command-contract.cjs` announces that it "Enforces the `commands/gsd/*.md`
contract across all 65 command files." There are 71. The linter globs the directory, so it
is functionally correct — its documentation is six commands behind. A small thing, but a
pointed one on a page about machine-checked conventions: the file that exists to prevent
drift has drifted in the one part of itself no machine reads.

### A command contradicts its own tool, in writing

`commands/gsd/plan-phase.md`'s `<context>` says the phase number is optional — "when
omitted, the orchestrating workflow reads ROADMAP.md and selects the next unplanned phase"
— and then immediately admits that "`gsd-tools.cjs` itself has no auto-detect feature and
requires an explicit phase number."

The optionality lives entirely in the prompt layer. What is notable is that the file *says
so* rather than papering over it. When your program is prose, you can document the seam
between what the prompt promises and what the tool delivers, in the same file, and the
model reading it gets both facts.

## What to take from this layer

If you are building a framework like this one:

1. **Pick one authoring surface and compile everything else from it.** GSD authors commands
   and generates 15+ host formats. The converter's job should be renaming, not translating.
2. **Layer your frontmatter by audience.** Runtime contract fields survive; build, install
   and lint fields get stripped. Decide which is which explicitly, and comment *why* — the
   `effort` strip is a prompt-cache argument that nobody would reconstruct from the code.
3. **Separate your surfaces with a token you can grep.** The colon/hyphen split means a
   linter aimed at the wrong surface finds nothing rather than finding something wrong.
4. **Build the compiler passes even though it's markdown.** Link resolution, grammar
   constraints and reachability are all real failure modes in a prompt system. GSD has a
   linter for each: `scripts/lint-command-contract.cjs`, `scripts/lint-skill-deps.cjs`,
   `scripts/lint-compiled-artifact-sync.cjs`, and `--check` modes on every generator. The
   repo's consistent preference is a script over a code-review rule.
5. **Commit your generated artifacts and gate on staleness** if consumers must work without
   a build. Then make `--check` part of CI, not part of code review.
6. **Accept that some semantics can only be asserted.** When there is no parser, redundant
   natural-language guards are a legitimate engineering technique — but route them through
   your inclusion mechanism so they stay in one place.
