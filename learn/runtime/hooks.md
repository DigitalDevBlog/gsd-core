# The hook system

Every framework built on top of a language model eventually discovers the same thing: an
instruction is a request, not a constraint. GSD Core's `hooks/` directory is where that
discovery got written down as code. It holds 26 hook scripts — 22 `.js` and 4 `.sh` — that
sit between the model and the host runtime and, in a handful of cases, refuse to let a tool
call through.

This page is about how that enforcement layer is constructed: where hooks are registered
(there is more than one answer, and that is the single most important fact here), what
contract every script obeys, and — the argument that justifies the whole layer — which
rules cannot survive as prose.

## What is actually in `hooks/`

```
26 hook scripts        gsd-*.js (22) + gsd-*.sh (4)
 1 plugin manifest     hooks/hooks.json
 1 registry module     hooks/managed-hooks-registry.cjs
 6 shared helpers      hooks/lib/
──
34 files, all committed
```

`git ls-files hooks/` returns exactly those 34 paths. This matters more than it sounds,
because it is the opposite of the situation in `gsd-core/bin/lib/`, where most `.cjs` files
are ADR-457 build output — compiled from `src/*.cts` at publish time and gitignored. Nothing
in `hooks/` is generated. `hooks/gsd-workflow-guard.js` is source of truth, quotable as-is.

!!! warning "The hooks that run on a user's machine are never these files"
    `.gitignore` excludes `hooks/dist/`, and `bin/install.js` stages from exactly there:
    `const hooksSrc = path.join(src, 'hooks', 'dist')` (`bin/install.js:11218`).
    `scripts/build-hooks.js` produces that tree. So `hooks/` is the committed *input* and
    `hooks/dist/` is the gitignored *output* — the same build-at-publish seam as
    `gsd-core/bin/lib/*.cjs`, applied a second time. Cite `hooks/*.js`; never cite a staged
    path as though it were source.

## Registration: the part that makes the system incomprehensible if you miss it

Read `hooks/hooks.json` and you will form a materially wrong picture of this framework.

That file is the Claude Code plugin manifest. It is auto-discovered — `.claude-plugin/plugin.json`
declares `commands` and `skills` but **no** `hooks` field, so the host finds `hooks/hooks.json`
by convention. It uses `${CLAUDE_PLUGIN_ROOT}` interpolation, and it registers:

| | count |
|---|---|
| lifecycle events | 7 (`SessionStart`, `PreToolUse`, `PostToolUse`, `SubagentStop`, `Stop`, `PreCompact`, `FileChanged`) |
| command invocations | 13 |
| distinct scripts referenced | **10 of 26** |

The other sixteen are registered somewhere else entirely — by `src/runtime-hooks-surface.cts`
and `bin/install.js`, which write each host's own settings file with absolute paths projected
through `src/shell-command-projection.cts`. That module is where the framework's most
security-relevant guard actually lives.

### The two surfaces do not nest

The obvious reading is that the installer surface is a superset of the manifest. It is not.
Enumerating both gives a genuine Venn diagram:

- **9 scripts** are on both surfaces.
- **1 script is manifest-only**: `hooks/gsd-ensure-canonical-path.js`. The repo's own test
  says why, at `tests/workflow-guard-registration.test.cjs:56-60` — *"registered ONLY in the
  marketplace-plugin manifest … the plugin install mode never runs `bin/install.js` at all."*
- **14 scripts are installer-only**, including all four `.sh` hooks, both Windsurf adapters,
  all six Cursor adapters, and `gsd-workflow-guard.js`.
- **2 scripts are on no hook-event surface at all**: `hooks/gsd-statusline.js`, wired as
  `settings.statusLine` (`bin/install.js:12564`), and `hooks/gsd-check-update-worker.js`,
  which `gsd-check-update.js` spawns internally.

And "the installer surface" is itself plural. `src/runtime-hooks-surface.cts` contains at
least three distinct emitters: the settings.json path, a Kimi TOML block
(`buildKimiHooksTomlBlock`, line 2429), and descriptor-driven adapters for Cursor and
Windsurf/Cascade that go through `CURSOR_EVENT_SCRIPT_MAP`
(`src/host-integration-adapters/imperative-hook-bus.cts:54`) and `WINDSURF_EVENT_SCRIPT_MAP`
(`src/runtime-hooks-surface.cts:1451`) rather than any literal call a source scan could find.

!!! note "If you are building this yourself"
    One hook, one registration. GSD Core has a plugin manifest and an installer that evolved
    independently, and the cost is visible below in three separate defects — a flag with no
    effect, a matcher that silently narrowed, and a hook that spawns a process to do nothing.
    None of those are logic bugs in a hook. All three are drift between registration surfaces
    that no single file reconciles.

## Where hooks sit in a session

```kroki-plantuml
@startuml
skinparam defaultFontSize 12
skinparam sequenceMessageAlign left
skinparam ParticipantPadding 14

participant "Model" as M
participant "Host\n(Claude Code)" as H
participant "hooks/*.js\nhooks/*.sh" as K
database ".planning/config.json\n.gsd/dispatch-isolation-sentinel.json" as S

== SessionStart ==
H -> K : gsd-ensure-canonical-path.js <i>manifest only</i>
H -> K : gsd-check-update.js
H -> K : gsd-update-banner.js <i>installer only, opt-in</i>
H -> K : gsd-session-state.sh <i>installer only, hooks.community</i>

== PreToolUse — only the matching subset fires ==
M -> H : one tool call
H -> K : gsd-prompt-guard.js <i>Write|Edit</i> — advisory
H -> K : gsd-read-guard.js <i>Write|Edit</i> — exits 0 on Claude Code
H -> K : gsd-write-guard.js <i>Write</i> — <b>BLOCK</b> shrink of curated .planning/
H -> K : gsd-worktree-path-guard.js <i>Write|Edit|MultiEdit</i> — <b>BLOCK</b> write outside worktree
H -> K : gsd-agent-isolation-guard.js <i>Agent|Task</i> — <b>BLOCK</b> unisolated dispatch
H -> K : gsd-workflow-guard.js <i>Bash|Edit|Write|MultiEdit — installer only</i>
H -> K : gsd-validate-commit.sh <i>Bash — installer only</i>
K -> S : read (fresh, never cached)
K --> H : exit 0\nor exit 2 + {"decision":"block"} on stdout + reason on stderr
H --> M : tool runs, or the block reason comes back

== PostToolUse ==
H -> K : gsd-read-injection-scanner.js <i>Read|WebFetch|WebSearch</i> manifest / <i>Read</i> installer
H -> K : gsd-context-monitor.js <i>Bash|Edit|Write|MultiEdit|Agent|Task</i>
H -> K : gsd-phase-boundary.sh <i>Write|Edit — installer only</i>

== FileChanged ==
H -> K : gsd-config-reload.js <i>config.json</i>

== SubagentStop / Stop / PreCompact ==
H -> K : gsd-context-monitor.js <i>unmatched</i>
@enduml
```

Read the matchers, not the column. A single `PreToolUse` fires only the hooks whose
matcher accepts that tool — an `Agent`/`Task` dispatch reaches `gsd-agent-isolation-guard.js`
and nothing else in that block; a `Bash` call reaches the workflow guard and the commit
validator and none of the write guards. And every arrow marked *installer only* is simply
absent on a plugin-marketplace install, because that mode never runs `bin/install.js`.

The `FileChanged` event with matcher `config.json` looks malformed next to the tool-name
regexes around it. It is not — it is a real, Claude-Code-only event, and it is what makes
`.planning/config.json` hot-reloadable mid-session.

## The 26, grouped by purpose

| Purpose | Scripts | n |
|---|---|---|
| **Guarding** — inspect a pending call, advise or block | `hooks/gsd-prompt-guard.js`, `hooks/gsd-read-guard.js`, `hooks/gsd-write-guard.js`, `hooks/gsd-worktree-path-guard.js`, `hooks/gsd-agent-isolation-guard.js`, `hooks/gsd-workflow-guard.js`, `hooks/gsd-read-injection-scanner.js`, `hooks/gsd-validate-commit.sh` | 8 |
| **State** — write, read or re-inject project state | `hooks/gsd-ensure-canonical-path.js`, `hooks/gsd-config-reload.js`, `hooks/gsd-session-state.sh`, `hooks/gsd-phase-boundary.sh`, `hooks/gsd-graphify-update.sh` | 5 |
| **UX / telemetry** — tell the human or the model something | `hooks/gsd-statusline.js`, `hooks/gsd-context-monitor.js`, `hooks/gsd-check-update.js`, `hooks/gsd-check-update-worker.js`, `hooks/gsd-update-banner.js` | 5 |
| **Host adapters** — translate a non-Claude host's hook bus | `hooks/gsd-cursor-{session-start,pre-tool,post-tool,stop,subagent-start,subagent-stop}.js`, `hooks/gsd-windsurf-{pre-write,pre-command}.js` | 8 |

A third of the directory is host-adapter code. That is the real cost of being runtime-agnostic,
and it is worth seeing before you commit to supporting more than one host.

## The guards: what each prevents, and why a sentence would not have

This is the argument the layer exists to make, and the most persuasive evidence is the
authors indicting their own prose. Three headers, quoted verbatim.

### `hooks/gsd-worktree-path-guard.js` — writes escaping an isolated worktree

> *"The prose guard in `agents/gsd-executor.md` step 0b is never enforced because the model
> under load skips it."* — `hooks/gsd-worktree-path-guard.js:8-9`

**Prevents:** an `Edit`/`Write`/`MultiEdit` with an absolute path resolving outside the
current worktree root — an executor running in `isolation="worktree"` writing into the user's
primary checkout.

**Why prose fails:** the instruction existed first, in the agent definition, and was observed
being skipped. Note also the implementation restraint: the guard does not reimplement path
comparison. It compares two raw `git rev-parse --show-toplevel` outputs, on the reasoning that
both values come from the same git binary in the same format by definition — which sidesteps
Windows 8.3 short names, case-insensitive volumes and slash direction for free.

### `hooks/gsd-agent-isolation-guard.js` — dispatching an executor with isolation silently dropped

> *"A prose backstop cannot fix a prose defect — it is the same class of artifact the model
> may equally skip. This hook enforces the invariant at the tooling layer instead:
> HARD-BLOCKING."*

**Prevents:** an `Agent(subagent_type="gsd-executor", …)` call in a project configured for
`harness-worktree` isolation where the model did not copy the isolation flag through. The
failure mode is not a crash — it is the executor committing directly into the user's primary
checkout, with no consent and no warning.

**Why prose fails:** the *delivery mechanism* was prose. `executor-isolation-dispatch.md`
resolved the correct value in shell and then asked the model, in English, to substitute it
into a tool call. Nothing verified it did. Writing a second English instruction to check the
first is the defect, recursively.

### `hooks/gsd-write-guard.js` — a whole-file `Write` that destroys curated history

The most honest header in the repository, and worth reading twice:

> *"Fixes 1 and 2 (PR #989) are instructions to a model: they lower the probability of a
> clobber but cannot prevent one, and they protect only the agents that were audited. This
> hook is enforced by code rather than by instruction … An advisory will not do — #973 records
> an agent reading the advisory, classifying it as non-binding, and reasoning past it while
> holding a false model of what `Write` does."*

**Prevents:** a `Write` that catastrophically shrinks an existing curated `.planning/` artifact
— the incident was a planner reading a ~16-line window of `ROADMAP.md` and overwriting all 292
lines with it.

Then the same header states its own bound, which is the part most frameworks omit:

> *"this stops accidental and single-shot collapse, not a determined agent. The sentinel hatch
> below is a plain file, so an agent that would reason past an advisory can arm one with a
> single Bash call it is already permitted to make. What ships is the conversion of 'ignore a
> sentence' into 'take one deliberate, path-bound, single-use, auditable action'."*

That is the correct claim for a hook layer to make. Hooks do not make a model safe. They
change the cost of non-compliance from *zero* to *one deliberate auditable act* — and they
convert an instruction the model is free to reinterpret into a state transition you can log.

### The rest

- **`hooks/gsd-workflow-guard.js`** — advisory when a file edit happens outside a GSD workflow
  context; one hard block, `WORKTREE_AGENT_FORCE_ADD_FORBIDDEN`, on `git add -f` over
  `agent-*` / `worktree-agent-*` branches. Its header advertises itself as a SOFT guard and
  then carries a block with the opposite failure direction from everything else in the file.
  Two contradictory postures in one script, deliberately.
- **`hooks/gsd-prompt-guard.js` / `hooks/gsd-read-injection-scanner.js`** — prompt-injection screening on
  the way in (content written to `.planning/`) and on the way back (content returned by tool
  calls). Both share `hooks/lib/injection-patterns.js` specifically so the two copies cannot
  drift. The scanner's own header is explicit about what it is not: *"a static pattern match —
  NOT a semantic guard … It does NOT understand context, intent, or novel phrasing."*
- **`hooks/gsd-read-guard.js`** — the thesis and its failure mode in a single file. See below.

## The invocation contract

Every one of the 26 obeys the same conventions. They are worth copying wholesale.

**A version token on line 2.** Every script carries `gsd-hook-version: {{GSD_VERSION}}`,
stamped with `pkg.version` at install time. `grep -L` across `hooks/*.js hooks/*.sh` returns
empty; no file in `hooks/lib/` carries it. That one token is what lets an installed tree
detect staleness, and it is also what distinguishes a hook from a helper.

**A self-imposed stdin budget.** Every stdin-consuming hook opens with the same shape — a self-timeout that exits clean rather than hanging the host. The *shape* is universal; the value is not. Seven of the 22 `.js` hooks use 3000 ms; others range up to 10000 ms (`gsd-context-monitor.js:36`, `gsd-windsurf-pre-write.js:66`), with `gsd-read-injection-scanner.js:216` at 5000 and `gsd-config-reload.js:27` at 8000. The 3-second guards open:

```js
let input = '';
const stdinTimeout = setTimeout(() => process.exit(0), 3000);
```

cleared on `'end'`. That is nested *inside* the host's own timeout (5s for guards, 10s for
`gsd-context-monitor.js`). The hook would rather kill itself than be killed — because a hook
killed by the host has undefined verdict semantics, whereas `exit(0)` means "allow".

**Two verdict channels, strictly separated.** Advisory hooks write
`{hookSpecificOutput: {hookEventName, additionalContext}}` to stdout and never block. Blocking
hooks write `{decision: 'block', reason, …}` to stdout, then write *the same reason again* to
stderr, then `exit 2`. The duplication is commented identically in three files — `gsd-agent-isolation-guard.js:490`, `gsd-worktree-path-guard.js:275` and `:303`, and `gsd-write-guard.js:169`. (`gsd-workflow-guard.js:118` makes the same point in different words, citing #2304.) The identical set is:

> *"Kimi feeds stderr (not stdout) back to the model on exit 2."*
> — `hooks/gsd-agent-isolation-guard.js:490`, and the same note in `gsd-worktree-path-guard.js`
> and `gsd-workflow-guard.js`

One script, two incompatible host protocols, no abstraction layer. That is the pragmatic
answer, and it is paired with matcher translation at registration time: the same guard is
registered as `Bash|Edit|Write|MultiEdit` for Claude Code
(`src/runtime-hooks-surface.cts:1969`) and `Shell|WriteFile|StrReplaceFile` for Kimi
(`src/runtime-hooks-surface.cts:2446`). Both layers are required — a translated matcher that
fires into an untranslated guard just reaches a script that exits immediately.

**Fail open, with two named exceptions.** Every guard terminates in `catch { process.exit(0) }`,
on the principle that a broken guard must never wedge a session. Exactly two legs invert it:

1. `gsd-workflow-guard.js`'s force-add block *re-derives its blocking context inside the catch*
   and exits 2 rather than allowing (#3504). It tags the verdict `origin: 'fail-closed'` so the
   model is never told a violation was detected when the call was merely unanalysed, and there
   is a `GSD_TEST_MODE`-gated fault-injection seam so the posture is testable.
2. `gsd-agent-isolation-guard.js` denies when it cannot resolve isolation, explicitly refusing
   to reuse `routeDispatchIsolation`'s fail-open default — *"that existing query is fail-OPEN
   by design (sequential execution is always safe for the SCHEDULER); this guard's job is the
   opposite invariant."*

Two different fail-open defaults in one codebase, each correct for its own invariant, with the
reasoning written where the divergence lives. That is the pattern to copy.

**Shared logic deliberately not shared.** Six guards each inline an identical ~90-line
`normalizeKimiPayload()`. The comment explains the choice:

> *"Inlined per guard (not `hooks/lib/`): hook scripts are staged as standalone files, and a
> sibling require is a staging dependency that can fail silently."*
> — `hooks/gsd-prompt-guard.js:33-35`

The function maps Kimi's tool names (`WriteFile`→`Write`, `StrReplaceFile`→`Edit`,
`ReadFile`→`Read`, `Shell`→`Bash`) through a `Map` rather than an object literal, because bare
bracket lookup on an object resolves `constructor` / `__proto__` / `toString` to truthy values
and the fall-through for unknown names would never fire. The exceptions that *are* shared —
`hooks/lib/injection-patterns.js`, `git-cmd.js`, `cursor-workspace.js` — are precisely the files on the
installer's `GSD_HOOK_LIB_FILES` allowlist. The allowlist is what licenses a sibling require.

**Shell hooks use Node as a JSON codec, never `jq`.** Stated as a dependency decision:
*"Uses Node.js for JSON parsing (always available in GSD projects, no jq dependency)"*
(`hooks/gsd-phase-boundary.sh:5`). They emit the same `hookSpecificOutput` envelope as the JS
hooks, with extra typed fields (`planning_modified`, `file_path`, `state_present`,
`config_mode`) added so tests assert on a contract instead of substring-matching prose.

**Comment density is inverted.** `hooks/gsd-prompt-guard.js` is 197 lines, roughly 120 of them
commentary, with individual comments running 20+ lines to explain why a `??` became a `?.` —
because each such change closed a crash-to-allow bypass where a malformed payload threw into
the fail-open catch and silently downgraded a block into an allow. The comments are an embedded
incident log. In an enforcement layer that is the right ratio: the reason a line exists *is*
the specification.

## State channels

Hooks are stateless subprocesses, so shared state has to go through the filesystem.

The main one is `.gsd/dispatch-isolation-sentinel.json`, defined as `SENTINEL_RELATIVE_PATH`
in `hooks/lib/isolation-sentinel.js` with a 10-minute staleness window (`SENTINEL_STALE_MS`).
A workflow step writes the resolved isolation decision after
`gsd-core/workflows/execute-phase/steps/executor-isolation-dispatch.md` computes it in shell;
`gsd-agent-isolation-guard.js` and `gsd-cursor-subagent-start.js` then read it. A fresh
sentinel is authoritative; an absent or stale one degrades to a conservative registry + config
check rather than guessing. This is the enforcement thesis made concrete — a workflow computes
a value in shell, persists it, and a hook enforces it, with the model never in the delivery path.

Secondary channels: `gsd-context-monitor.js` dedupes headroom warnings through
`os.tmpdir()/claude-ctx-<sessionId>-warned.json`, which is what lets one script be registered
on four different events without repeating itself. And `.planning/config.json` is the opt-in
gate, re-read fresh on every invocation rather than cached — hot-reloaded mid-session by
`gsd-config-reload.js` on `FileChanged`.

Note the deliberate trade: guards default to **off** in config but are **registered
unconditionally** at install time. The gate lives in the script, not the registration. That
costs a wasted subprocess per matched event and buys the ability to toggle behaviour without
reinstalling. Reasonable — but it is what makes the next defect possible.

## Where it breaks

These are the passages worth studying hardest, because each is a consequence of a structural
decision rather than a typo.

**A config flag with no effect on plugin installs.** `hooks/gsd-workflow-guard.js` is opt-in via
`hooks.workflow_guard: true` in `.planning/config.json`, and it is registered *only* by
`src/runtime-hooks-surface.cts`. It is absent from `hooks/hooks.json` entirely. Per the repo's
own test, plugin installs never run `bin/install.js`. So a user on a plugin-marketplace install
who sets that flag gets silence: neither the advisory nor the `git add -f` hard block can ever
fire, and nothing diagnoses the mismatch. A flag the user can set that does nothing is worse
than no flag.

**A matcher that narrowed on one surface only.** `hooks/gsd-read-injection-scanner.js` is registered
on `Read|WebFetch|WebSearch` in `hooks/hooks.json` but on `'Read'` alone at
`src/runtime-hooks-surface.cts:1943`. The script's own header says it *"scans content returned
by Read, WebFetch, and WebSearch."* Injection scanning of fetched web content therefore exists
on the Claude plugin surface and nowhere else — including on classic Claude Code installs.

**A guaranteed no-op that still costs a subprocess.** `hooks/hooks.json` registers
`hooks/gsd-read-guard.js` on `PreToolUse` / `Write|Edit`. The script exits 0 the moment it detects
Claude Code (`hooks/gsd-read-guard.js:147-155`, via `data.session_id` or `CLAUDE_CODE_*`).
Its header states plainly that it exists for non-Claude models — *"MiniMax M2.5 on OpenCode"*
looping forever on read-before-edit. Since `hooks/hooks.json` is consumed exclusively by Claude
Code, that registration spawns a Node process on every Write and Edit purely to exit. It is
also a small irony worth naming: the one guard written because a model *will not comply* is
registered on the surface where it cannot run.

**Two load-bearing helpers escape lifecycle management.** `GSD_HOOK_LIB_FILES`
(`bin/install.js:383`) lists four files. `hooks/lib/isolation-sentinel.js` and `hooks/lib/isolation-deny-reason.js`
are not among them, yet both are hard top-level `require`s of `hooks/gsd-agent-isolation-guard.js`
(lines 66-67) and `hooks/gsd-cursor-subagent-start.js` (lines 64-65). They reach the target anyway —
via an unfiltered one-level recursion into `hooks/dist` subdirectories added by #3579 for an
unrelated graphify helper. But `GSD_HOOK_LIB_FILES` governs two *other* things: manifest
tracking (`bin/install.js:9723`) and uninstall removal (`bin/install.js:8609`). So both files
are invisible to drift detection, and are left behind on uninstall — which then makes the
following `fs.rmdirSync(hooksLibDir)` fail, so the directory is never pruned either. An
allowlist used for three different purposes will eventually be right for one of them.

**A registry that conflates "files we ship" with "hooks".** `hooks/managed-hooks-registry.cjs`
exports a 26-entry `MANAGED_HOOKS` array matching disk 1:1 — including `hooks/gsd-statusline.js`,
which is not a hook-event handler at all. At 1065 lines it is the largest file in `hooks/` by a
wide margin, and it is wired as `settings.statusLine.command`. It nonetheless implements the
full hook convention: stdin JSON, 3-second self-timeout, fail-open. The conflation is
conventional rather than accidental — but the array's name promises something it does not
deliver.

**Non-uniform opt-in gates on the community hooks.** The installer groups the four `.sh` files
into a single `expectedShHooks` array under the comment *"Warn if expected community `.sh`
hooks are missing"* (`bin/install.js:11287-11288`), implying one switch. Three gate on
`hooks.community === true`; `hooks/gsd-graphify-update.sh` requires its own separate pair,
`graphify.enabled && graphify.auto_update`. A user who sets `hooks.community` and expects four
hooks gets three.

**Hooks that self-heal the build seam.** Six hooks plus `hooks/lib/isolation-sentinel.js`
`require` gitignored ADR-457 artifacts under `gsd-core/bin/lib/*.cjs`, which do not exist on a
raw plugin-marketplace or git-clone install that never ran `npm run build:lib`. Each routes
that require through `ensureRuntimeBuild()` from `gsd-core/bin/ensure-runtime-build.cjs`.
`gsd-statusline.js` scopes the seam to `require.main === module` and degrades to printing
nothing, *"because the statusline renders on EVERY prompt, so a build failure here must DEGRADE
… rather than crash Claude Code's per-render statusline hook."* Correct handling — of a problem
created by choosing to ship build output.

## The transferable lesson

The rule for deciding what belongs in a hook is not "what is important." It is:

> **Does this rule survive a model that has read it, understood it, and decided it does not
> apply?**

If yes, prose is fine — put it in the agent definition and move on. If no, it must be code,
and the evidence in this repository is that you will usually find out which category a rule is
in the expensive way. All three hard-blocking guards here were written *after* the prose
version demonstrably failed: `#260` for the worktree escape, `#973` for the ROADMAP clobber,
`#3045` for the dropped isolation flag. Each header names the incident.

Four corollaries the implementation earns:

1. **Fail open by default; fail closed only where you can name the invariant.** A broken guard
   that wedges every tool call is a worse outcome than the violation it was preventing. But
   when you do invert it, engineer the inversion separately — re-derive the context inside the
   catch, tag the verdict so the model knows the difference between *detected* and
   *unanalysable*, and build a fault-injection seam so the posture stays tested.
2. **State the bound.** `hooks/gsd-write-guard.js` says out loud that it does not stop a determined
   agent, only accidental collapse. A guard whose limits are documented is a guard people use
   correctly.
3. **One registration surface.** Every defect above is drift between two mechanisms that
   describe the same thing.
4. **Enforcement is a state transition, not a sentence.** The value hooks add is not
   prevention — it is converting "ignore a sentence" into a deliberate, path-bound, auditable
   act. That is achievable. Prevention is not.
