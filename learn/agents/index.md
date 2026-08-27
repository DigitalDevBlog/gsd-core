# Agents

An agent definition in GSD Core is a single markdown file with four YAML keys and a prompt
body. There are exactly 35 of them in `agents/`. That is the whole storage format — no
class, no registration call, no interface to implement.

The interesting part is what surrounds that file. A definition alone is inert; what makes
it *dispatchable* is a set of agreements that live outside it — a roster that three
separate files must agree on, a tool grant that gets rewritten per runtime, and a declared
completion contract in `gsd-core/references/agent-contracts.md` that a CI script enforces
in both directions. This page works through those agreements, because they are the part
you would have to rebuild.

The question to hold throughout: **what must a subagent definition specify to be safe to
spawn?**

## The file

Every one of the 35 files in `agents/` opens the same way:

```yaml
---
name: gsd-plan-checker
description: Verifies plans will achieve phase goal before execution. Goal-backward analysis of plan quality. Spawned by /gsd:plan-phase orchestrator.
tools: Read, Bash, Glob, Grep, Skill
color: green
---
```

Four keys, and the frequency across the roster is total: all 35 declare `name`,
`description`, `tools`, and `color`. Exactly one key appears anywhere else — a single
`model: sonnet` in `agents/gsd-mempalace-curator.md`.

That uniformity is not accidental, and it is worth noticing that the schema is *small*. A
framework of this size could easily have grown per-agent timeouts, retry policies, memory
settings, or permission modes. Instead those live in the orchestration layer, and the
agent file stays a prompt plus a grant.

### What each field actually does

The four fields are not four instances of the same thing. They are read by different
consumers, and they fail differently.

| Field | Who reads it | Failure mode if wrong |
|-------|--------------|-----------------------|
| `name` | The dispatcher, as `subagent_type` | Spawn fails or silently falls back |
| `description` | The dispatcher, for selection; also scanned by CI | Wrong agent picked; no length gate |
| `tools` | Install-time converters, then the host's permission layer | Install aborts, or a body path is dead |
| `color` | The host UI only | Nothing — no GSD logic reads it |

`name` is the dispatch address. Workflows spawn by literal string:
`gsd-core/workflows/secure-phase.md:123` carries `subagent_type="gsd-security-auditor"`,
and `gsd-core/workflows/validate-phase.md:118` carries
`subagent_type="gsd-nyquist-auditor"`. There is no indirection — the filename stem, the
`name:` value, and the dispatch string are the same token in three places.

`color` is the honest outlier — the only one of the four that **no behavioural decision
depends on**. GSD's own logic does touch it: `colorNameToHex`
(`src/runtime-artifact-conversion.cts:128`) translates the name to a hex value at `:1928`
and `:2112` for hosts that want one. But every use is presentational — it is translated,
forwarded or discarded depending on where the agent is being installed, and nothing
branches on its value. `src/runtime-artifact-conversion.cts:1899` and `:2083` strip it outright
for opencode and Kilo agents, and the Cursor, Windsurf, and Augment converters do the same
(`:2332`, `:2351`, `:2370`). Meanwhile the Qwen converter re-emits it verbatim
(`:2466`) and the Copilot converter reads it at `:2283`. If you are designing a comparable
format, `color` is a useful reminder that a field can be worth keeping for
human ergonomics while carrying zero semantic load — provided you know which one it is.

!!! note "Two disagreeing signals in one file"
    `agents/gsd-mempalace-curator.md:5` pins `model: sonnet`. But model selection for the
    roster is data, not prose: `gsd-core/bin/shared/model-catalog.json` assigns that agent
    `routingTier: "light"`, and the catalog's `adaptiveTierMap` maps `light` to haiku. So
    under the adaptive profile the catalog resolves haiku while the file pins sonnet. The
    other 34 agents carry one signal; this one carries two. Which wins depends on the
    install path, but the disagreement is in the source either way.

### The body

Below the frontmatter, the bodies are markdown but structured with XML pseudo-tags rather
than headings. All 35 use a `<role>` block, and 34 open with it directly
(`agents/gsd-intel-updater.md` leads with `<required_reading>` instead). From there the
common sections are `<success_criteria>` (30 of 35), `<execution_flow>` (19), `<structured_returns>` (14), `<project_context>` (13), and `<critical_rules>`
(11 of 35). One agent breaks the convention entirely: `agents/gsd-mempalace-curator.md`
switches to markdown `##` headings after `<role>`/`<inputs>` — which is harmless only
because the contract checker's marker detection is in-fence only, so its `## Gate` and
`## Report` headings cannot be mistaken for completion markers.

Shared prose is meant to be `@`-included from `gsd-core/references/`, and this is where the
convention is thinnest. Only five agents — executor, planner, verifier, debugger, and
phase-researcher — actually `@`-include
`~/.claude/gsd-core/references/mandatory-initial-read.md`. Three others
(`gsd-doc-writer`, `gsd-doc-verifier`, `gsd-mempalace-curator`) **inline a prose paraphrase**
of the same instruction instead, and six agents reference `~/.claude/gsd-core` nowhere at
all. A shared reference that five of 35 files use is not yet a convention; it is a habit
three authors happened to share, with the other copies already drifting into restatement.

!!! note "Those `@~/.claude/` paths are not a portability bug"
    It looks Claude-specific, but it is an authoring convention resolved at install time.
    `_applyRuntimeRewrites` — defined in `src/runtime-artifact-conversion.cts` and
    re-exported through `bin/install.js:7408` — rewrites `~/.claude/`, `$HOME/.claude/`,
    and `./.claude/` to the target runtime's config location. The Copilot converter's own
    header (`bin/install.js:1753-1757`) spells the mapping out: global installs send
    `~/.claude/ → ~/.copilot/` and `./.claude/ → ./.github/`, local installs send both to
    `./.github/`. Authors write one canonical path; the installer localises it.

## The roster is a registry, not a directory listing

The most transferable structural decision on this page: **GSD Core does not discover agents
by listing a directory.** Three independent files must name the same 35 agents, and a
script enforces the agreement.

| File | What it holds | Count |
|------|---------------|-------|
| `agents/*.md` | The definitions themselves | 35 |
| `gsd-core/bin/shared/model-catalog.json` | Keys under `agents`, each `{golden, balanced, budget, phaseType, routingTier}` | 35 |
| `gsd-core/references/agent-contracts.md` | Rows in the `## Agent Registry` table | 35 |

All three sets are identical — no entry appears in one and not the others — and
`node scripts/check-contract-drift.cjs` reports `35 agents, 38 markers, 0 violations`.
The script is wired at `package.json:92` and runs inside `lint:ci` at `:122`.

The consequence is worth stating plainly, because it inverts the intuition that the
filesystem is the source of truth. `src/agent-install-check.cts:159` derives the expected
roster from `Object.keys(MODEL_PROFILES)` — re-exported by `src/model-profiles.cts` from
`src/model-catalog.cts`, which reads `model-catalog.json`. **An agent `.md` dropped into
`agents/` with no catalog entry is invisible to the install checker.** The catalog is the
roster; the directory is just where the prompts happen to live.

!!! tip "Path provenance"
    `gsd-core/bin/shared/model-catalog.json` is committed (`git check-ignore` exits 1 on
    it). Do not confuse it with `gsd-core/bin/lib/*.cjs`, which is mostly build output
    compiled from `src/*.cts` and gitignored. Only eight `.cjs` files are committed under
    `bin/lib/`; anything else you find there is generated.

### Seven of the 35 are generated

A subset goes further than validation. `scripts/gen-research-agents.cjs` treats the seven
researcher agents' frontmatter as *derived* from a hand-authored profile table in
`scripts/research-profiles.cjs`. Each profile declares `name`, `description`, `color`,
`tools`, plus three body-content assertions: `requiredIncludes` (the
`@~/.claude/gsd-core/references/*.md` files the body must `@`-include), `requiredSeamCalls`
(the `gsd_run query …` calls it must make), and `outputContract` (its output path and
return marker).

`--write` regenerates only the frontmatter block, leaving the body byte-identical; `--check`
asserts all four properties. The generator preserves the existing commented-hooks block
byte-for-byte rather than re-emitting it — a small detail that tells you the authors wanted
`--write` to produce an empty `git diff` on a clean tree.

Note where the enforcement lives: `gen-research-agents.cjs` is **not** in `lint:ci` and not
in `lint:generated-sync`. It is enforced by `tests/research-agent-profiles.test.cjs`, which
imports `checkAgent` directly. Same guarantee, different pipeline — worth knowing if you
go looking for it in the lint chain and conclude it is unguarded.

## Least-privilege tool scoping

`tools:` is the security-relevant field, and GSD Core uses it with real discrimination
rather than granting every agent everything.

| Tool | Agents granting it (of 35) |
|------|---------------------------|
| `Read` | 35 |
| `Grep` / `Glob` | 33 |
| `Bash` | 32 |
| `Write` | 26 |
| `Skill` | 22 |
| `Edit` | 12 |
| `WebSearch` | 8 |
| `WebFetch` | 7 |
| `AskUserQuestion` | 3 |
| `Agent` | 1 |

The spread runs from `agents/gsd-user-profiler.md`, which grants exactly one tool
(`tools: Read`), to `agents/gsd-phase-researcher.md` with 17. `Agent` — the ability to
spawn further subagents — is granted to exactly one agent,
`agents/gsd-debug-session-manager.md`, whose whole job is to run a multi-cycle spawn loop
in an isolated context. That is a defensible use of a dangerous capability: one holder,
named, with a stated reason.

### What the scoping buys

Beyond the obvious blast-radius reduction, the grant list is **legible structure**. You can
read a great deal about an agent's role from its tools without opening the body: three
agents lack `Bash` entirely (`gsd-doc-classifier`, `gsd-dom-verifier`, `gsd-user-profiler`),
and nine lack `Write`.

But be careful how much you infer. The contract registry classifies agents by how their
caller detects completion, and cross-tabulating that against `Write` is instructive:

| Registry `Kind` | Write-less agents in that class |
|-----------------|--------------------------------|
| `structured-return` | 5 — advisor-researcher, assumptions-analyzer, framework-selector, integration-checker, user-profiler |
| `sentinel-match` | 3 — plan-checker, security-auditor, ui-checker |
| `artifact+query` | 1 — mempalace-curator |

The nine Write-less agents scatter across **all three** completion classes. Absence of
`Write` is necessary for `structured-return` but nowhere near sufficient — which matters,
because the contract document argues otherwise. More on that below.

### What the scoping costs

Three real costs, all visible in the source.

**1. The grant list is not portable, and a bad translation aborts the load.** `tools:` is
authored in Claude's vocabulary and must be rewritten for every other runtime.
`convertGeminiToolName` (`bin/install.js:1583`) excludes `Task`, `Agent`, `AskUserQuestion`,
`Skill`, and `SlashCommand` from Gemini agents, and the comments record why in incident
terms: emitting
`AskUserQuestion` "causes frontmatter validation errors (#3362)", and the lowercase
fallback for `Skill` would emit an invalid name "that fails frontmatter validation
(tools.N: Invalid tool name) and **aborts the entire agent load** (#1394)". A
least-privilege list is a per-runtime translation surface, and the failure is loud and
total rather than degraded.

**2. The parser must handle every YAML spelling.** 33 agents write `tools:` as a
comma-separated inline string; two (`gsd-nyquist-auditor`, `gsd-security-auditor`) use a
block sequence. `parseFrontmatterTools` in `bin/install.js:2014` carries an explicit
`collecting` state machine to accept both, plus `allowed-tools:` as an alias.

**3. The grant can silently drift from the body.** This is the expensive one, and GSD Core
has both the bug and the guard.

### The dead-branch bug class

Issue #2526 records the shape: `gsd-ui-auditor` declared
`tools: Read, Write, Bash, Grep, Glob, Skill` — no MCP grant at all — while its body
presented a `<playwright_mcp_approach>` block as the *preferred* capture path. As the
regression test puts it, "that branch was unreachable by construction: the availability
check had a fixed answer, the three `mcp__playwright__*` calls could never dispatch, and
the 'when Playwright-MCP is NOT available' fallback was the only branch that ever ran."

The guard is `tests/mcp-tool-inheritance.test.cjs`, and it is a model of how to write this
kind of check. It reads frontmatter through the canonical parser rather than a hand-rolled
scan (the test requires `gsd-core/bin/lib/frontmatter.cjs` — a **generated, gitignored**
build artifact compiled from `src/frontmatter.cts`, which is the file to cite). It then
documents five deliberate scope boundaries, each with a stated reason and each exercised by
a negative control:

- **Server-level, not exact-tool** — the bug class is a server with zero grants.
- **Trailing `__` required** — bare prose naming a server is not an invocation.
- **Single-character server ids are prose metavariables** — `mcp__X__*` in an example is
  not a reference; the sanctioned multi-character placeholder is `mcp__<SERVER>__*`, exempt
  because `<` lies outside the regex character class. The test explicitly refuses to exempt
  `mcp__SERVER__*`, reasoning that an all-caps escape hatch "would be a false NEGATIVE for
  any real server that happens to be spelled in caps, and a guard that misses a dead
  reference fails in the direction this whole check exists to prevent."
- **`description` is scanned, `tools:` is not** — because `tools:` "legitimately contains
  the grants themselves and would self-reference into a guaranteed pass."

That last one generalises: **a consistency check must never scan the field it is checking
against.** It is the kind of thing you only learn by writing the vacuous version first.

!!! warning "The guard is spelling-dependent, and one instance escapes it"
    `agents/gsd-mempalace-curator.md` reproduces the exact #2526 shape. Its frontmatter is
    `tools: Read, Bash, Grep, Glob` — no MCP wildcard. Its body instructs "prefer the
    `mempalace_*` MCP tools interactively" and names seven concrete calls
    (`mempalace_diary_write`, `mempalace_diary_read`, `mempalace_kg_query`,
    `mempalace_kg_add`, `mempalace_kg_invalidate`, `mempalace_find_tunnels`,
    `mempalace_create_tunnel`). Only the Bash/CLI fallback is reachable.

    It survives CI because the guard's `REFERENCE_RE` is `/mcp__([A-Za-z0-9_-]+?)__/g` — it
    only sees tools spelled with the `mcp__<server>__` prefix. The curator names its MCP
    tools in their bare form, so the regex never fires. Nine other agents *do* declare MCP
    wildcards (`mcp__context7__*`, `mcp__chrome-devtools__*`, the firecrawl/exa/tavily set),
    which is what makes the curator's omission look like an oversight rather than a
    convention.

    The transferable lesson is uncomfortable: the guard checks a *spelling*, not a
    *capability*. It catches the bug in the notation the original bug happened to use.

!!! danger "A missing `tools:` key is unchecked — and, per the test's authors, ungranted"
    Two separate things go quiet when `tools:` is absent, and they belong to different
    layers.

    **The check goes vacuous.** In `tests/mcp-tool-inheritance.test.cjs`, `toolTokens`
    returns `null` when there is no `tools:` key, and `ungrantedServers` then returns `[]`
    before examining the body at all — the agent passes without being checked. That is
    visible directly in the test source.

    **The harness is said to grant everything.** The test's own comment glosses a missing
    key as "no `tools:` key at all (inherits everything)". That is the authors' statement
    about host-harness semantics, not something GSD Core implements or that I traced into
    GSD's code — but it is the assumption the guard was written against.

    All 35 agents declare `tools:`, so the hole is unoccupied today. Still, the combination
    is the sharpest answer to the page's framing question: **a definition is safe to spawn
    because its grant is stated, not inherited.** If the permissive case is also the
    unchecked case, the two failure modes compound — you lose the restriction and the
    warning at the same moment. If you build this, make an absent `tools:` a hard error
    rather than a default, so the schema cannot be quietly forgotten into grant-all.

## The contract an agent fulfils for its caller

Spawning is only half the problem. The caller also has to know **when the agent is done and
where the result is** — and in a prompt-driven system neither is guaranteed by a type
signature. `gsd-core/references/agent-contracts.md` is where GSD Core writes that down.

Its `## Agent Registry` table gives every agent a `Completion Markers` list, a `Consumed by`
list, and a `Kind`. Marker Rule 4 defines `Kind` as exactly one of three values, and the 35
rows split evenly enough to show all three are load-bearing:

| `Kind` | How the caller detects completion | Rows |
|--------|-----------------------------------|------|
| `sentinel-match` | Exact-case string match on a declared `## MARKER` H2 | 15 |
| `artifact+query` | Agent writes a file; caller reads or queries it | 15 |
| `structured-return` | Agent returns parseable text inline; caller reads the return | 5 |

Naming the mechanism is the design move. "Which string do I match?" becomes a table lookup
instead of a read-35-prompts exercise, and — more importantly — it becomes checkable.

### The filesystem is the primary handoff medium

Notice that `artifact+query` and `sentinel-match` tie at 15 each, and that the two largest-by-file-size
agents are both artifact-based. Control in GSD Core passes through the filesystem more often
than through the return channel, and the agent prompts say so in unusually blunt terms.
`agents/gsd-planner.md` and `agents/gsd-executor.md` carry an identical hard `Write contract`
block: the orchestrator reads the artifact from disk and "does NOT read your return message
for the file content"; "Do NOT return the PLAN.md/SUMMARY.md content in your response"; and
"If writing still fails, surface the actual error — Do NOT silently fall back to returning
content."

That last clause is the interesting one. The obvious fallback — if the write fails, just
return the content — is explicitly forbidden, because a caller that reads from disk would
see a missing file and a success message. The design prefers a loud failure to a
half-honoured contract.

The contract document is willing to record its own asymmetries. Its closing note observes
that `## PLAN COMPLETE` is gsd-executor's declared marker, but `execute-phase.md` does not
regex-match it — completion is detected via SUMMARY.md existence and git commit state — and
labels this "intentional behavior, not a mismatch." Documenting the gap beats pretending
the abstraction is uniform.

### Bidirectional enforcement

`scripts/check-contract-drift.cjs` (297 lines) is what keeps the table from becoming
stale prose. Its header enumerates seven arms, and two of them are worth calling out as
patterns:

- **Arm 3 checks agreement in both directions.** Every marker an agent emits in-fence and
  every marker the registry declares must match. If a row's `Kind` is `artifact+query` or
  `structured-return`, then emitting *any* marker is itself a violation
  (`vestigial_marker`) — the opt-out is not a free pass.
- **Arm 7 runs the check backwards.** A workflow or command matching a quoted `## TOKEN`
  that no agent declares is `unmatched_consumer_token` — dispatch-on-phantom. Most drift
  checkers only ask "is every declaration used?"; this one also asks "is every use
  declared?"

Marker detection is deliberately **in-fence only**, which is why
`gsd-mempalace-curator`'s prose H2 headings (`## Gate`, `## Tasks`, `## Report`) do not
register as phantom markers.

There is also a small piece of vocabulary design here. A marker that is emitted but matched
by nobody is annotated `(unconsumed: <reason>)` rather than deleted — Marker Rule 7 restricts
the exemption to display formats and warns "use it for display formats, never to silence a
real orphan." The check still verifies the marker is declared *and* emitted, and still counts
it for case-collision purposes; only the consumer requirement is waived. Contrast with
gsd-doc-synthesizer's `## Synthesis Complete`, which #3565 **deleted** because it
case-collided with gsd-research-synthesizer's `## SYNTHESIS COMPLETE`. Annotate the
deliberate exception; delete the actual collision.

### The registry's stated rationale is wrong

The `Kind` column is correct. The prose justifying two of its cells is not.

The gsd-integration-checker row explains its `structured-return` classification with
"(agent has no Write tool -- it cannot write an artifact)". That agent's grant is
`Read, Bash, Glob, Grep, Skill`. **It has `Bash`** — it can write a file trivially. And
`gsd-mempalace-curator` holds a near-identical Write-less, Bash-having grant
(`Read, Bash, Grep, Glob`) yet the same table classifies it `artifact+query` precisely
because it writes a session diary.

So the stated premise is false, and it yields opposite conclusions on two rows with the same
tool grant. The broader cross-tabulation above makes the point with a count rather than an
anecdote: of the nine Write-less agents, five are `structured-return`, three are
`sentinel-match`, and one is `artifact+query`. Lacking `Write` predicts nothing.

This is a narrow documentation defect with a wide lesson. `Kind` is a genuine property of
the caller relationship — a *choice* about where the result lands — and the moment it is
re-derived from a tool grant, it stops being a contract and becomes an inference. The
column is machine-checked; the parenthetical is not, and it rotted.

### Two more that CI does not see

The three-way roster alignment is perfect and the drift checker is clean, which makes the
un-linted defects stand out by contrast:

- `agents/gsd-doc-verifier.md` has an **unbalanced XML tag**. `<role>` opens at line 14 and
  closes at line 25; a second, stray `</role>` sits at line 217 as the final line of the
  file, after `</success_criteria>`. Every other section in that file balances. The drift
  checker validates fenced-code-block closure in agent files (arm 2) but not XML tag
  balance, so it passes.
- `agents/gsd-doc-writer.md:597-605` numbers its `<critical_rules>` **1, 2, 3, 4, 8, 9, 5,
  6, 7**. Rules 8 and 9 were spliced in after rule 4 without renumbering. Rule 3's
  forward-reference to "see rule 7" still resolves, but any future "rule N" reference is
  ambiguous.

Both are the same failure: the agent bodies are prompts, and prompt structure has no
compiler. GSD Core built checkers for the two structural properties that break dispatch
(fence closure, marker agreement) and none for the properties that only break comprehension.

## The gap in the contract: `agent_hint`

The clearest example of a contract with a consumer but no producer.

A PLAN.md can declare `agent_hint: <name>` in its frontmatter to opt into a specialist
executor. The consumer chain is complete and careful:

- `src/config.cts:96` gates it on `workflow.agent_hint_routing`, default-on.
- `src/plan-document.cts:293` and `src/phase.cts:600,932` parse the frontmatter field.
- `gsd-core/workflows/execute-phase/steps/per-plan-executor-routing.md` resolves it into
  `EXECUTOR_TYPE`, which becomes `subagent_type` at dispatch.
- `resolveAgentHint` (`src/agent-install-check.cts:387`) does the resolution.

`resolveAgentHint` is worth reading as a piece of defensive design. It answers a different
question from the roster check — "does an agent file for this **arbitrary** name exist?" —
so a plan can name a domain specialist that shares the gsd-executor contract without being
in the built-in 35. It is fail-closed on every axis: empty or whitespace names return
`null`; any name containing `/`, `\`, or `..` is rejected, with the comment naming the
attack outright ("a value like `../../README` cannot path-traverse out of the agents dir via
`path.join` and match an unrelated file — that would echo an invalid `subagent_type` and
block the wave"); and a resolution miss returns `null` so the caller falls back to
`gsd-executor`. A misspelled hint degrades to the default rather than failing the wave.

And yet: **`grep -rn agent_hint agents/` returns zero hits across all 35 agent files.**
`gsd-planner.md`'s `## Frontmatter Fields` table (lines 397-410) does not list it. The agent
that authors PLAN.md has never been told the field exists.

The routing step is also candid that it is only half-wired. On the `orchestrator-worktree`
isolation backend a resolved hint "cannot be honored yet" — that backend spawns via a
process path with no `subagent_type` — so the step emits a note to stderr rather than
silently dropping it:

```
note: plan ${plan_id} agent_hint='${PLAN_HINT}' resolved, but orchestrator-worktree
dispatch does not route subagent types in this release — using the default executor.
```

Announcing a partial implementation at the seam is the right call. Not telling the producer
its field exists is not. A frontmatter field is a two-sided contract, and only one side was
written.

## Loose conventions around the tight core

The enforced parts of the agent format are genuinely tight. The unenforced parts are not,
and the contrast is informative about where the authors chose to spend their linting budget.

**Descriptions have no budget.** `scripts/lint-descriptions.cjs` enforces a 100-character
ceiling — on `commands/gsd/*.md` only, hard-coded to `COMMANDS_DIR`. Agent descriptions run
from 72 to 362 characters with no gate at all, despite being the surface a dispatcher selects
on and the surface the #2526 MCP check scans. Worse, the linter is unwired: it appears in no
`lint:ci` chain and nothing in `.github/` references it. It is exercised only against
synthetic fixtures by `tests/skill-frontmatter-contract.test.cjs`, which runs the script as
a subprocess — the script is tested, but never actually run over the real tree. Meanwhile
`docs/research/2026-05-12-skill-surface-budget.md:34` states the budget is "enforced in CI
by `scripts/lint-descriptions.cjs`." It is not. (The repo's `docs/` tree is unreliable in
exactly this way; prefer source.)

**Frontmatter form varies.** 33 agents use inline comma-separated `tools:`; two
(`gsd-nyquist-auditor`, `gsd-security-auditor`) use a block sequence. Nothing enforces one
spelling, so the parser carries the cost — see `parseFrontmatterTools` above. The body-level
variation is covered in [The body](#the-body).

**Two-thirds of the roster carries dead config.** 25 of 35 agent frontmatters contain
commented-out `# hooks:` blocks, and **not one agent has a live `hooks:` key**. Most are
`PostToolUse` matchers on `Write|Edit` running `npx eslint --fix $FILE`. Several are pure
placebo: `gsd-doc-classifier` and `gsd-doc-synthesizer` would run `command: "true"`;
`gsd-ai-researcher` would `echo 'AI-SPEC written'`. `gsd-intel-updater` has a bare
`# hooks:` with no body. `gsd-code-fixer` and `gsd-code-reviewer` use an entirely different,
non-schema shape (`#   - before_write`).

That last cluster deserves a beat. Beyond being dead weight that every reader must decide
to ignore, the eslint variant would — if uncommented — run `npx eslint --fix` inside
arbitrary user projects that may not use eslint. Commented-out code in a *prompt* is a
peculiar hazard: the file is read by a language model as well as by a parser, and a block
that looks like configuration is context the agent pays for on every spawn.

## Building your own

What must a subagent definition specify to be safe to spawn? Reading GSD Core's answer
backwards, the definition needs four things, and the surrounding system needs two more.

**In the definition:**

1. **A dispatch identity** that is the same token everywhere — filename, `name:`, and the
   caller's spawn string. GSD Core keeps all three literally identical, which removes an
   entire class of mapping bug.
2. **An explicit tool grant, where absent means deny.** GSD Core's guard treats a missing
   `tools:` as grant-all; all 35 files happen to declare it, so the hole is theoretical
   today. Make it an error, and the schema will not betray you later.
3. **A declared completion mechanism**, not just a declared marker. The `sentinel-match` /
   `artifact+query` / `structured-return` split is the reusable idea: the caller needs to
   know *where to look*, and that is a different fact from *what to look for*.
4. **A stated behaviour on partial failure.** The planner and executor's "do not silently
   fall back to returning content" is the pattern — pick the loud failure over the
   half-honoured contract.

**Around the definition:**

5. **A registry, not a directory scan.** Make some other file the roster and check both
   directions. GSD Core's three-way alignment across `agents/`, `model-catalog.json`, and
   `agent-contracts.md` costs one CI script and makes "I added a file and nothing happened"
   impossible to ship silently.
6. **Guards that check the *capability*, not the *spelling*.** This is where GSD Core's
   own machinery falls short, and it is the most useful thing on this page. The #2526 check
   is thoughtfully built — canonical parser, documented scope boundaries, negative controls
   to prove it still fires — and it still misses `gsd-mempalace-curator` because that agent
   writes `mempalace_diary_write` instead of `mcp__mempalace__diary_write`. A guard written
   against the notation of the bug that prompted it will catch that bug again and little
   else.

The pattern underneath all six: in a prompt-driven system, every contract is *prose* until
something machine-checks it. GSD Core's agent layer is a fairly precise map of which
contracts got checked. The roster, the markers, and the MCP grants did — and they hold. The
`Kind` rationale, the XML balance, the rule numbering, and `agent_hint`'s producer side did
not — and every one of them has drifted.
