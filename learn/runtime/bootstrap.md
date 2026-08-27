# Bootstrap & distribution

A framework that only exists in one repository is a script. A framework that has to arrive
on someone else's disk, next to eighteen different host runtimes' config directories (the
installer's `runtimeMap`, `bin/install.js:12746-12765`), and keep working after the user
edits the files it installed — that is a distribution problem, and it is where most of
GSD Core's genuinely hard engineering ended up.

This page is about the seam between "the repo" and "an install". Four questions organise
it: what the published package contains, how it gets from a tarball into a host's config
directory, what happens when the artifacts a bin entry needs are not there, and how a
20-line shell script encodes one of the deepest constraints in agent-harness authoring.

!!! info "Read `runtime/index.md` first for the build regime"
    The build-at-publish rule (ADR-457: `src/*.cts` is truth, `gsd-core/bin/lib/*.cjs` is
    gitignored `tsc` output) is established in [the runtime overview](index.md), along
    with the nine files that *are* committed under `gsd-core/bin/lib/`. This page assumes
    it and goes after the *distribution* consequences instead — which turn out to be more
    interesting than the build itself.

## Four bin entries, two directories, one rule

`package.json:6-11` declares four executables:

| `bin` name | Path | Size |
|---|---|---|
| `gsd-core` | `bin/install.js` | 13,756 lines / 642,054 bytes |
| `gsd-tools` | `gsd-core/bin/gsd-tools.cjs` | 4,423 lines / 217,227 bytes |
| `gsd_run` | `gsd-core/bin/gsd_run` | 20 lines / 849 bytes |
| `gsd-mcp-server` | `bin/gsd-mcp-server.js` | 31 lines / 1,369 bytes |

(Sizes from `wc -lc` on each path.)

Two of them live in top-level `bin/` and two in `gsd-core/bin/`. That split looks like
historical sediment. It is not — it is a stated rule, and `bin/gsd-mcp-server.js:5-9`
writes it down at the exact place a future contributor would get it wrong:

> Lives at top-level `bin/` (alongside `install.js`) — it is a PACKAGE bin the host spawns
> via `npx gsd-mcp-server` (or the global bin), NOT a per-runtime artifact copied into a
> host's config dir. (Placing it under `gsd-core/bin/` would leak it into every runtime
> install + break golden parity.)

The rule, generalised: **`gsd-core/` is payload, everything else is packaging.** The whole
`gsd-core/` tree gets recursively copied into each host's config directory by
`copyWithPathReplacement` (`bin/install.js:10991`). So anything you put there ships to
every user of every runtime, appears in the install-tree golden fixtures, and has to
survive a per-runtime content rewrite. Anything in top-level `bin/` is invoked from the
npm bin shim and never copied at all.

This is worth stealing. If your framework both *installs itself somewhere* and *exposes
CLIs*, the two sets of executables have completely different lifecycles, and putting them
in one directory means every new file silently joins whichever set the copy loop happens
to cover. Encoding the boundary as a directory — and asserting the reason in the header of
the file most likely to be misplaced — costs nothing and is self-enforcing.

## What actually ships

`package.json:12-33` is a `files[]` whitelist: 21 entries, 8 of which are negations
(`node -e "const f=require('./package.json').files; console.log(f.length, f.filter(x=>x[0]==='!').length)"`).
There is no `.npmignore`, so the whitelist is the whole story.

The positives are `bin`, `commands`, `skills`, `gsd-core`, `assets`, `agents`,
`.claude-plugin`, `.opencode`, `GEMINI.md`, `hooks`, `scripts`, `pi`, `vscode`. The
negations are all under `scripts/` — eight named test/QA/codegen helpers
(`gen-emitted-baseline.cjs`, `qa-smell-ratchet.cjs`, `live-config-guard.cjs`,
`run-tests.cjs`, `affected-tests-lib.cjs`, `run-affected-tests.cjs`,
`lint-no-adhoc-regex-escape.cjs`, `lint-allow-test-rule-refs.cjs`) carved back out of a
directory that is otherwise shipped wholesale.

### `capabilities/` does not ship, and that is the design

Notice what is *absent* from that list: `capabilities/`. The repo has 46 capability packs
(`ls capabilities | wc -l`), and not one of them reaches the tarball.

They do not need to. `scripts/gen-capability-registry.cjs:25` reads `capabilities/` at
build time and emits a single flattened CommonJS module,
`gsd-core/bin/lib/capability-registry.cjs` — 7,638 lines, 315,501 bytes, and one of the
nine `.cjs` files git-tracked under `bin/lib/`. It is *larger than `gsd-tools.cjs`
itself.*

The flattening is total, not just metadata. Each pack may carry prompt fragments alongside
its `capability.json` — `find capabilities -type f` returns 46 `capability.json` and
`find capabilities -name '*.md'` returns 12 fragment files.
`materializeHookFragments` (`scripts/gen-capability-registry.cjs:470`) reads each fragment
off disk and inlines its text into the registry, so an entry ends up carrying *both* keys:

```js
"fragment": {
  "path": "fragments/execute-wave-pre.md",
  "inline": "# Claude orchestration — Workflow execution backend (BETA)\n\n> Injected at …"
}
```

`grep -c '"inline":'` on the generated registry returns 28 — more than the 12 fragment
files, because inlining happens once per *contribution point* that references a fragment,
and a pack can reference the same file from several points. The `path` survives as
provenance; the `inline` is what actually gets injected at runtime.

Who reads the repo's `capabilities/` directory? Scanning every script for a real disk read
rather than a doc-comment mention —
`for f in scripts/*.cjs; do grep -qE "join\([^)]*'capabilities'\)|readdirSync\([^)]*capabilities" "$f" && echo $f; done`
— returns exactly two: `gen-capability-registry.cjs` (line 25, the flattener) and
`sync-manifest-versions.cjs` (`:130-147`, which stamps each pack's `version` field at
release). Even `scripts/gen-capability-matrix.cjs`, which writes the human-facing
capability table, reads the *generated registry* rather than the source packs
(`:31`) — the flattened artifact has become the interface for the repo's own tooling too.

Nothing in an **installed** tree reads it. Grepping `src/` for a `'capabilities'` path
segment returns only *installed* roots — `~/.gsd/capabilities`
(`src/capability-loader.cts:342`), `<projectRoot>/.gsd/capabilities` (`:352`),
`<runtimeDir>/.gsd/capabilities` (`src/capability-lifecycle.cts:236`),
`<gsdHome>/.gsd/capabilities` (`src/capability-source.cts:764`). Those are the
**third-party overlay** roots, a different mechanism entirely.

So first-party capability packs are a **source format, not a distribution format**. The
directory is developer ergonomics — one folder per capability, schema-validated, diffable
— and the shipped artifact is one machine-written file with cross-capability invariants
already resolved. If you are designing an extension system, this is the cleaner half of
the tradeoff: authors get files, the runtime gets a single validated table, and there is
no filesystem walk on the hot path of every install.

## The install-time layout mapping

```kroki-plantuml
@startuml
skinparam componentStyle rectangle
skinparam shadowing false
skinparam defaultTextAlignment center
skinparam nodesep 12
skinparam ranksep 34

package "Repo only — never shipped" {
  component "capabilities/\n46 packs + 12 fragments" as caps
  component "src/*.cts\n194 modules" as srcx
}

package "Published tarball  (package.json files[])" {
  component "bin/\ninstall.js\ngsd-mcp-server.js" as pkgbin
  component "gsd-core/\nbin/ · workflows/\ncontexts/ · templates/\nreferences/" as pkgcore
  component "commands/ · skills/\nagents/ · hooks/\nassets/ · scripts/ (−8)" as pkgassets
}

package "Host config dir  (~/.claude, ~/.codex, …)" {
  component "gsd-core/\n(whole payload,\nrewritten per runtime)" as instcore
  component "commands/gsd/\nskills/ · agents/\n(layout-driven names)" as instcmds
  component "gsd-file-manifest.json\ngsd-pristine/\ngsd-local-patches/" as instupd
}

package "Files GSD does not own" {
  component "~/.codex/config.toml\ncopilot-instructions.md\n~/.agents/AGENTS.md" as foreign
}

caps -[#666]-> pkgcore : gen-capability-registry.cjs\n**flattens to one .cjs**\n(paths + inlined fragments)
srcx -[#666]-> pkgcore : tsc at prepack/prepare\n**gitignored in-repo,\nprebuilt in tarball**

pkgcore -[#0a7]-> instcore : copyWithPathReplacement\nconfinementRoot asserted\n**before** the rmSync
pkgassets -[#0a7]-> instcmds : resolveRuntimeArtifactLayout\n+ RUNTIME_CONTENT_DISPATCH\n**absence = identity**
pkgbin .[#999].> instcore : never copied —\nnpm bin shim only

instcore -[#c60]-> instupd : generateManifest (SHA-256)\npopulatePristineDir\nsaveLocalPatches
instcore -[#a06]-> foreign : marker-delimited block\nspliced in;\nevery writer has a\nstripGsdFrom* inverse
@enduml
```

Four arrow kinds, four different mechanisms. The green arrows are the copy; the grey are
the build; the orange is the update system's write side; the purple is the one place GSD
writes into a file it did not create. The dotted arrow is the rule from the previous
section — `bin/` deliberately does not participate.

## The self-healing build, and the two entrypoints that do not heal

Build-at-publish creates an obvious hazard: the repository ships a tree that cannot run
until `tsc` has run over it. On the npm path that is fine — `build:lib` is wired into
`prepare`, `prepack`, `prepublishOnly` and `pretest` (`package.json:111-115`). But
`gsd-core/bin/ensure-runtime-build.cjs:12-17` names a channel where none of those fire:

> The Claude Code plugin-marketplace channel does NOT go through `npm publish` or
> `bin/install.js`. Claude Code materializes the git tag tree into its plugin cache and at
> most runs `npm install --ignore-scripts`, so neither `prepare` nor `build:lib` ever
> fires.

The response is a 246-line module that compiles the runtime's own dependencies on first
use. Four decisions in it are worth copying:

**It depends on nothing it might be missing.** Its own docstring: *"Deliberately depends on
nothing under `./lib` (that tree is precisely what may be missing) — only on Node
built-ins."* A bootstrap check that imports from the thing it is checking is not a check.

**One sentinel stands for the whole tree.** `SENTINEL = 'cli-exit.cjs'`
(`ensure-runtime-build.cjs:37`), justified by a compiler setting rather than by hope:
`tsconfig.build.json` sets `noEmitOnError: true`, so `tsc` is all-or-nothing and one file's
existence implies all of them. That reasoning is written into the constant's comment,
which is what makes the shortcut auditable instead of superstitious.

**The lock is an `mkdir`, and it expires.** `acquireLock` on
`gsd-core/bin/lib/.build.lock`; a loser polls for the sentinel with a 120 s deadline, then
*takes over the stale lock* rather than failing (`:140-165`). The motivating load is
stated: a workflow shells out to `gsd-tools` many times in parallel, so the very first
GSD-driven session on a marketplace install is exactly when N processes race.

**It distrusts its own incremental cache.** `forceFullEmit` (`:196-215`) deletes the
`tsbuildinfo` before invoking `tsc`, because heal only runs when the output is already
broken, and a stale incremental cache from a partial build would let `tsc` exit 0 without
re-emitting the missing sentinel. A build tool reporting success while producing nothing
is the failure mode this exists to prevent.

`gsd-core/bin/gsd-tools.cjs:245-256` wires it in correctly: `ensureRuntimeBuild()` runs
*before* the first `./lib` require, and its catch prints only `bootErr.message` — the
actionable `RuntimeBuildError` text — then exits 1, with a comment explaining that the
normal `ExitError`/`runMain` machinery cannot be used because it lives in `./lib`.

!!! danger "Defect: `bin/install.js` and `bin/gsd-mcp-server.js` have the seam and not the guard"
    Both top-level bins require gitignored `gsd-core/bin/lib/*.cjs` modules directly.
    `grep -c "require('\.\./gsd-core/bin/lib/" bin/install.js` returns **15**
    (lines 22, 31, 38, 43, 51, 55, 56, 57, 60, 63, 64, 65, 69, 74, 90). Neither file calls
    `ensureRuntimeBuild` first.

    This checkout is unbuilt — `ls gsd-core/bin/lib/*.cjs | wc -l` returns 8, i.e. only the
    committed files. On it:

    ```
    $ GSD_TEST_MODE=1 node bin/install.js --help
    node:internal/modules/cjs/loader:1252
      throw err;
    Error: Cannot find module '../gsd-core/bin/lib/shell-command-projection.cjs'

    $ echo '' | node bin/gsd-mcp-server.js
    node:internal/modules/cjs/loader:1252
      throw err;
    ```

    A raw V8 stack, where the seam has a written, actionable message ready to print.

    **Reachability matters for how you rank this.** npm installs build via
    `prepare`/`prepack`; the marketplace channel, per `ensure-runtime-build.cjs`'s own
    docstring, never invokes `install.js` at all. So the live path is a developer cloning
    the repo and running a bin before `npm install` — a DX defect, not user breakage. The
    interesting part is not severity.

!!! warning "The interesting part: the guard for this seam excludes these files by one constant"
    `scripts/lint-hooks-runtime-build-seam.cjs` (wired into `lint:ci`) enforces *exactly*
    this rule — call `ensureRuntimeBuild` before requiring a compiled `lib` module. Its
    scope is one line: `SCAN_ROOT = 'hooks'` (`:101`). Its own docstring says so under a
    bolded limitation heading — *"**`hooks/` only.** `scanRepo`'s `walk()` only descends
    `SCAN_ROOT`"* (`:80`).

    So the rule and the violation coexist in the same repository, separated by a directory
    constant. This is the recurring shape of structural guards: **a lint's blast radius is
    a design decision, and it silently ages.** `hooks/` was where the incident happened, so
    `hooks/` is what got scanned. Nothing revisits that when a second directory grows the
    same seam. If you write guards like this, the scope constant deserves the same review
    scrutiny as the rule — and ideally a comment naming what is *deliberately* excluded, so
    the exclusion is a decision rather than an oversight.

### Why the ignore list is enumerated instead of globbed

`grep -c '^/gsd-core/bin/lib/' .gitignore` returns **218** — one line per emitted `.cjs`,
not a `*.cjs` glob. That is deliberate, and it inverts the default: anything under
`gsd-core/bin/lib/` that is *not* on the list is presumed hand-authored and gets committed.
Adding a `src/*.cts` module means hand-adding its emitted path. The cost is a chore; the
benefit is that the nine tracked `.cjs` files under `gsd-core/bin/lib/` — eight at the top level plus the vendored `vendor/re2js.cjs` — are tracked *on purpose* rather than by
accidental non-matching.

Two negations carve out vendored code (`.gitignore:300-308`), and the comment gives the
constraint that forces vendoring at all:

> `bin/**` must have zero external requires (installed trees ship with no `node_modules`),
> so this one directory is a deliberate, tracked exception to the blanket `vendor/` ignore
> above.

Hold on to that sentence — it explains the dependency findings at the bottom of this page.

## `gsd_run`: authoring for a harness with no shell continuity

The whole file is 20 lines: a shebang, seven lines of comment justifying it (lines 2-8), and twelve lines of code
(`gsd-core/bin/gsd_run:2-8`):

> `gsd_run` — standalone launcher so workflow bash blocks can invoke the GSD tools in a
> fresh shell. Claude Code (and similar runtimes) runs each fenced bash block in a separate
> process, so an inline `gsd_run()` function defined in an earlier block is undefined in
> later ones (issue #381).

That is a deep constraint and it generalises past this repo. When your framework's
"program" is a markdown document full of fenced bash blocks executed by a host, **you do
not have a shell session.** You have N unrelated processes that happen to share a working
directory. Every abstraction shell programmers reach for by reflex — a function, an
exported variable, a `cd`, an activated virtualenv, a trap — has a lifetime of exactly one
block. The only things that persist are the filesystem, the environment the host chooses
to rebuild, and executables on `PATH`.

GSD's answer is to make the abstraction a *file*: 20 lines of POSIX `sh` with `set -e`,
resolving its own real location through symlinks before `exec node "$dir/gsd-tools.cjs"
"$@"`. A file survives process boundaries; a function does not.

### The same snippet carries two mechanisms, and it is easy to misread

`gsd-core/workflows/_runtime-launcher.snippet.sh` is the single source of truth,
propagated into every workflow and agent file by `scripts/sync-runtime-launcher.cjs`. It is
4,514 bytes on **one line** — a long `if`/`elif` chain probing runtime homes in order:
`$RUNTIME_DIR` or the git toplevel, then `.claude/`, `.codex/`, then `command -v
gsd-tools`, then `$CLAUDE_CONFIG_DIR`, `$HERMES_HOME`, `$CURSOR_CONFIG_DIR`,
`$CODEX_HOME`, `$GEMINI_CONFIG_DIR`, `$COPILOT_CONFIG_DIR`, `$WINDSURF_CONFIG_DIR`,
`$AUGMENT_CONFIG_DIR`, `$TRAE_CONFIG_DIR`, `$QWEN_CONFIG_DIR`, `$CODEBUDDY_CONFIG_DIR`,
`$CLINE_CONFIG_DIR`, `$GROK_AGENTS_HOME`, `$ANTIGRAVITY_CONFIG_DIR`,
`$OPENCODE_CONFIG_DIR`, `$KILO_CONFIG_DIR`, else an error telling the user to reinstall.
Counting `grep -o 'gsd_run() {'` gives **20** definitions — one per resolution branch,
nineteen of them file probes plus the `command -v` PATH branch.

Now the apparent contradiction. `scripts/sync-runtime-launcher.cjs:9-22` inserts that
preamble into **only the first** bash block in a file that calls `gsd_run`, with the
rationale *"Define once per file, use across blocks."* But `gsd_run`'s own header says a
definition in block 1 is gone by block 2. Both cannot be true of the same mechanism.

They are not the same mechanism. The resolution is the **last statement of the snippet**:

```sh
if [ -n "${CLAUDE_ENV_FILE:-}" ] && [ -n "${GSD_TOOLS:-}" ]; then
  printf "export PATH='%s':\"\$PATH\"\n" "${GSD_TOOLS%/*}" >> "$CLAUDE_ENV_FILE" 2>/dev/null || true
fi
```

The first block defines a shell function *and* appends a `PATH` export to a file the host
re-sources for later blocks. `CONTEXT.md:95` states the split explicitly: on runtimes that
run each block in a fresh shell, the `CLAUDE_ENV_FILE` export is what makes `gsd_run`
resolve later, *"the inline function definition remains the fallback for all other
runtimes."*

So: **function for hosts with session continuity, real executable on `PATH` for hosts
without.** One snippet, two audiences, and the difference is invisible unless you read the
tail.

That is elegant, but it is also a chain, and it is worth naming each link rather than
assuming it holds. On a fresh-shell host, `gsd_run` resolves in block 2 only if
`$CLAUDE_ENV_FILE` was set by the host, *and* the append succeeded — `|| true` swallows a
read-only or missing path silently — *and* the host actually sources that file before the
next block. If any link fails, the fallback is the npm `bin` entry: `gsd_run` on the global
`PATH` from `npm i -g @opengsd/gsd-core`. A local-scope install with no `CLAUDE_ENV_FILE`
has neither. I have not run that configuration, so I state the chain rather than claiming
it breaks — but a design whose correctness depends on a silently-swallowed append is worth
an explicit probe in the workflow's first block.

The generalisable lesson, though, is the one in the header comment. **Find out what your
harness resets between steps, and write it down in the artifact that exists because of
it.** `gsd_run`'s comment is eight lines for twelve lines of code, and it is the highest
comment-to-code ratio in the repository for good reason: nothing about the file explains
itself, and the constraint it encodes is invisible from the code.

## The MCP server: one protocol instead of N plugins

`src/mcp-server.cts:1-27` states the goal plainly — expose GSD over stdio JSON-RPC 2.0 so
*"any MCP-consuming host (Claude/Codex/OpenCode/VS Code/Gemini/Cursor/Cline/Hermes) can
drive GSD with NO bespoke plugin."*

The surface is deliberately narrow: three tools (`src/mcp-server.cts:115-149`), covering
two of GSD's six declared interface points.

| Tool | Interface point | Backing seam |
|---|---|---|
| `gsd_invoke_command` | 1 — command | `dispatchGsdCommand` (`src/shell-command-projection.cts`) |
| `gsd_read_state` | 5 — state IO | `stateIO` (`src/state-io.cts`) |
| `gsd_write_state` | 5 — state IO | `stateIO` |

Three packaging decisions are visible here.

**The protocol loop is hand-rolled to avoid a dependency.** From the same docstring: *"No
new runtime dependency — the JSON-RPC stdio loop is hand-rolled (the repo ships only
claude-agent-sdk + ws; adding an MCP SDK is a separate packaging decision)."* Given the
`.gitignore` rule that installed trees have no `node_modules`, an SDK dependency would not
merely be weight — it would be unresolvable in the shipped tree. The constraint at the
bottom of the packaging stack propagates all the way up into protocol implementation
choices.

**Protocol logic is split from process lifecycle.** `handleMessage` is a pure function over
a request object; `runServer` is a thin line-delimited-JSON loop over *injectable* streams.
`bin/gsd-mcp-server.js` is then 31 lines that pass `process.stdin`/`process.stdout` in. The
payoff is stated in the shim's own header: `tests/gsd-mcp-server.test.cjs` covers the
protocol, `tests/gsd-mcp-server-bin.test.cjs` covers spawn → JSON-RPC → clean exit. The
same testability pattern shows up in both large entrypoints (see below) — this repo
consistently treats "can this be tested without a subprocess?" as a design input.

**`initialize` refuses to advertise what it does not implement.** `src/mcp-server.cts:255`
returns `capabilities: { tools: {}, resources: {}, prompts: {} }`, with the comment:

> Deliberately NO `subscribe`/`listChanged` — nothing ever sends those notifications
> (design row 2 / Hyrum's Law: advertising an unimplemented notification is a lie a host
> would act on).

In a protocol whose entire point is that unknown clients integrate without coordination,
the capability handshake is the only contract you have. Declaring an unimplemented
capability is not optimism, it is a bug you cannot observe locally.

## What `install.js` does to files it does not own

`install()` spans `bin/install.js:10089-12538` (~2,450 lines) and `uninstall()` spans
`8024-9024` (~1,000). Earlier in the same file, at `bin/install.js:4440-6260`, sits the hand-written TOML layer — roughly 1,130 lines across the `*Toml*` functions: value
and object parsers (`5527`, `5696`), multiline-basic-string state tracking (`4552`,
`4569`), line records (`4851`), assignment-block-end detection (`6082`), key-line rewriting
(`6186`), schema validation (`5926`) and the merge itself (`6260`).

Nobody writes a TOML lexer for fun. The reason is a requirement: GSD merges its block into
a user's existing `~/.codex/config.toml` **by splicing line ranges**, not by parsing to an
object and re-serialising. A round-trip through any generic TOML serialiser would silently
destroy the user's comments, key ordering and whitespace. Choosing not to do that is what
costs about 1,130 lines — 8% of `bin/install.js`, rising to roughly 20% if every Codex-specific function is counted alongside the TOML ones.

The same principle generalises into a uniform ownership model. Every third-party file GSD
touches gets a marker-delimited block, and **every writer has a named inverse**:

| Writer | Inverse | Marker |
|---|---|---|
| `mergeCodexConfig` (`6260`) | `stripGsdFromCodexConfig` (`4377`) | `GSD_CODEX_MARKER` (`130`) |
| `mergeCopilotInstructions` (`6560`) | `stripGsdFromCopilotInstructions` (`6599`) | `GSD_COPILOT_INSTRUCTIONS_MARKER` (`275`) |
| AGENTS.md writer | `stripGsdFromAgentsMd` (`6640`) | `GSD_AGENTS_MD_MARKER` (`6624`) |

Codex goes further with `GSD_CODEX_HOOKS_OWNERSHIP_PREFIX` (`131`), which records *who*
owns a hooks block so a foreign-owned one is not clobbered — plus targeted strippers for
sections that leaked in past versions (`stripCodexGsdAgentSections` `4337`,
`stripLeakedGsdCodexSections` `5086`, `stripStaleGsdHookBlocks` `5154`). The existence of
those three is its own lesson: **once you write into someone else's file, your uninstaller
inherits every bug your installer ever shipped.**

!!! warning "Latent: two marker constants are byte-identical with different close markers"
    `GSD_COPILOT_INSTRUCTIONS_MARKER` (`bin/install.js:275`) and `GSD_AGENTS_MD_MARKER`
    (`:6624`) are the **same 58-character string**, written two different ways in source:

    ```js
    // :275 — em dash written as a JS escape
    const GSD_COPILOT_INSTRUCTIONS_MARKER = '<!-- GSD Configuration \u2014 managed by gsd-core installer -->';
    // :6624 — identical value, written with a literal em dash character
    const GSD_AGENTS_MD_MARKER = '<!-- GSD Configuration — managed by gsd-core installer -->';
    ```

    Confirmed by evaluating both literals and comparing (`a === b` → `true`, length 58),
    not by reading. Their close markers differ:
    `<!-- /GSD Configuration -->` versus `<!-- End GSD Configuration -->`. Both strippers
    do a bare `content.indexOf(OPEN)` followed by `indexOf(CLOSE)`.

    I am calling this latent, not live: today the two target different files
    (`copilot-instructions.md` versus `~/.agents/AGENTS.md`), and I did not enumerate every
    call site. It also degrades safely — a lone open marker leaves the other stripper's
    close `indexOf` at `-1` and the content returns unchanged.

    The transferable point is about *review*, not about this bug. Two constants written in
    visibly different styles, on lines 6,349 apart, are equal strings. No side-by-side
    reading catches that. If marker identity is load-bearing for a destructive operation,
    the invariant "all open markers are distinct" wants to be a test, because it is not
    something a human can check by looking.

### Path confinement is centralised and fails closed

`normalizeInstallRelativePath` (`:9513`) rejects absolute paths in both posix and win32
form, any empty / `.` / `..` segment, and NUL bytes. `resolveInstallRelativePath` (`:9528`)
re-resolves against the base and rejects anything landing outside the root, including any
symlink crossing. And `copyWithPathReplacement` (`:7563`) **throws if `confinementRoot` is
undefined** — with the check running before its `rmSync`, so a crafted `destDir` cannot
delete outside the declared install root.

The ordering is the whole point. A confinement check after a destructive call is
decoration. Making the parameter mandatory-by-throw rather than defaulted also means a new
call site cannot opt out by omission — the same "absence is an error, not a default" stance
that shows up in the enumerated gitignore.

### Per-runtime differences are a table where absence means identity

`RUNTIME_CONTENT_DISPATCH` (`bin/install.js:7428-7562`) is keyed by runtime id, and each
entry declares **only its delta** — `md`/`js` transforms, `mdSkipGenericRewrite`,
`mdReattributeAfter`, `mdTomlRenameOnCommand`. No entry means identity after the uniform
steps, which is how `claude`, `augment`, `codebuddy` and `kimi` are handled: by not being
mentioned.

Two details make this better than the usual switch-to-table refactor. First, `qwen` and
`hermes` take their brand *values* from `_hostBehaviors(runtime).brandingRewrites` —
descriptor-driven (ADR-1239/#2092) — and degrade closed to a no-op if the registry fails to
load, so a missing descriptor produces unbranded output rather than corrupted output.
Adding a runtime is meant to be a descriptor edit, not a new branch.

Second, one key is documented as dead-but-kept: `mdTomlRenameOnCommand` is *"(unused since
the gemini runtime was removed, #1928; kept as generic dispatch infra for a future
TOML-command runtime)"*. That is the honest version of a pattern usually left to inference.
A reader who greps for the key and finds no consumer would otherwise have to guess whether
it is scaffolding or a bug; the source answers before they ask. **If your converter has a
field no input exercises, say so in the field's own comment** — the alternative is that
someone eventually documents inferred semantics as fact.

### The update path treats user edits as data

Because GSD copies files into a directory the user also edits, upgrade is a merge, not a
copy. The write side lives in `install.js`:

- `generateManifest` (`:9496`) — SHA-256 per installed file → `gsd-file-manifest.json` via
  `writeManifest` (`:9548`)
- `populatePristineDir` (`:9800`) — the transformed-but-unmodified baseline in
  `gsd-pristine/`
- `saveLocalPatches` (`:9874`) — diffs installed files against the manifest and backs up
  user-modified ones into `gsd-local-patches/` (`PATCHES_DIR_NAME`, `:9482`)
- `reportLocalPatches` (`:10054`) — surfaces them to the user

The read side is a **separate executable**: `gsd-core/bin/verify-reapply-patches.cjs`, a
tracked 17 KB script implementing a deterministic "Hunk Verification Gate" that asserts
every user-added line survived the merge, diffing against the pristine baseline. Its
docstring names #2969 as the motivating bug — the gate used to be the model asserting
`verified: yes` in prose.

That move is the single most transferable decision on this page. **A verification step
performed by the thing being verified is not a verification step.** When the merge is done
by an LLM, the check that the merge preserved user content has to be code, run
out-of-band, with a non-zero exit.

!!! bug "Broken usage path in a shipped script"
    `gsd-core/bin/verify-reapply-patches.cjs:11` documents its own invocation as
    `node scripts/verify-reapply-patches.cjs`. That path does not exist — `test -e` fails.
    The file was *deliberately moved* out of `scripts/` for exactly the payload/packaging
    reason at the top of this page: `scripts/` ships in the npm tarball but is not among
    the trees the installer copies into a host's config directory. The changeset says so
    (`.changeset/archived/jolly-newts-roam.md:5`, and `happy-jays-greet.md:5`, #2994):
    *"moved `scripts/verify-reapply-patches.cjs` to `get-shit-done/bin/` which is shipped by
    the installer."* The move was right; the docstring inside the moved file still names the
    old path, so a user copy-pasting the documented command gets `Cannot find module`.
    `scripts/gen-adr-index.cjs:64` carries the same error in a different shape, referring to
    *"the worked example in `bin/verify-reapply-patches.cjs`"*.

    It is part of a pattern. Three shipped comments point at line ranges that have drifted
    by thousands of lines: `gsd-core/bin/lib/stale-bake-guard.cjs:6-9` cites
    `bin/install.js ~5667-5767` for the codex model bake, but those lines are now TOML
    number-literal parsing and table-header validation — the real bake sites are
    `generateCodexAgentToml` (`:3931`, overrides read at `:3964`) called from
    `installCodexConfig` (`:6882-6891`), `convertClaudeToOpencodeFrontmatter` (`:7075-7079`)
    and `convertClaudeToKiloFrontmatter` (`:7261-7266`).
    `gsd-core/bin/lib/profile-pipeline-command-router.cjs:13-14` cites
    `gsd-tools.cjs:366-371`; the function is at `:394` and its fail-fast errors at `:473`
    and `:587`. `gsd-core/bin/gsd-tools.cjs:314` says a helper is *"invoked from `case
    'init'` below"* — there is no `case 'init'`; it is `HOST_COMMAND_ROUTERS.init` at
    `:3676-3680`.

    **Cross-file line-number references are write-once, wrong-forever.** Nothing in `lint:ci`
    checks them, and nothing can cheaply. Name the symbol, not the line — a symbol reference
    is greppable and self-correcting; `~5667-5767` is a lie with a timestamp.

### Both big entrypoints are built to be imported

`bin/install.js` gates its entire main block on `require.main === module &&
!process.env.GSD_TEST_MODE` (`:13679`, closing at `:13756` with the comment
`// end of !GSD_TEST_MODE main logic block`) and exports **127** internals from a
`module.exports` object spanning `:13538-13676` (counting bare identifier entries in that
range: `awk 'NR>=13539 && NR<=13676' bin/install.js | grep -cE '^    [A-Za-z_][A-Za-z0-9_]*,?$'`).
`gsd-tools.cjs` uses `if (require.main === module) runMain(main)`
(`:4401-4403`) and exports its dispatch seams from `:4410` so tests can inject synthetic
registries and module loaders. Several prompt builders (`buildRuntimePromptText`,
`buildUpdateBannerPromptText`) were made pure specifically so tests assert on rendered
output instead of grepping source — their docstrings say exactly that.

The `GSD_TEST_MODE` escape hatch is worth noting as a *smell with a reason*. An env var
that suppresses `main` is the kind of thing that leaks into production behaviour. But it is
the pragmatic answer when the alternative is splitting a 642 KB file. The honest reading:
this is the cost of a file that grew past the point where it could be restructured cheaply,
and the env gate is the repair, not the design.

!!! note "Comment drift in the interactive prompt"
    `bin/install.js:12811` reads: *"Tokenize first so the all-runtimes shortcut also fires
    for inputs the prompt encourages — `\"16,\"`, `\"16 1\"`, etc. — not just the bare
    `\"16\"`."* `ALL_RUNTIMES_OPTION` is `'19'` (`:12767`); option 16 is `trae`
    (`runtimeMap`, `:12762`). The code is correct — it uses the constant — and only the
    comment drifted.

    The underlying fragility is three hand-maintained parallel lists that must stay in
    sync: `runtimeMap` keys 1..18 (`:12746-12765`), the `allRuntimes` array of 18 ids
    (`:12766`), and hand-written menu text in `buildRuntimePromptText` (`:12774+`). Adding a
    runtime means editing three places and a magic number. Given that this repo already
    derives per-runtime *behaviour* from descriptors, the prompt is the one place the
    data-driven approach stopped at the UI boundary.

## Two declared dependencies, neither of them `require`d

`package.json:63-64` declares exactly two runtime dependencies. Both are unusual.

**`ws@^8.21.0` has no code references in the shipped tree.** (It is named in prose in `docs/adr/3625-vetted-spawn-library-evaluation.md` and in the changelog, but never imported.) A repo-wide grep for
`require('ws')` / `require("ws")` and for `'ws'|"ws"|WebSocket` — excluding
`node_modules`, `.git`, `.venv` — finds no source hit in `src/`, `gsd-core/`, `hooks/`,
`scripts/`, `bin/` or `.opencode/`. The only matches are test fixtures using `ws` as a
*workstream* abbreviation, plus `package.json` and the lockfile themselves. And yet
`src/mcp-server.cts:22` names it in prose — *"the repo ships only claude-agent-sdk + ws"* —
inside the paragraph explaining why a third dependency was not added. A dependency cited by
name as a reason not to take on more, which nothing imports.

**`@anthropic-ai/claude-agent-sdk@^0.2.84` is present to be *read*, not imported.** There is
no `require`/`import` of it outside `node_modules`. It appears once, as a path string:
`src/claude-orchestration-command-router.cts:122` builds
`AGENT_SDK_PKG = path.join('@anthropic-ai','claude-agent-sdk','package.json')`, and
`resolveInstalledAgentSdkVersion` (`:124-146`) walks `node_modules` upward reading that
file. The comment explains why not `require.resolve`: the SDK's `exports` map does not
expose `./package.json`, so resolution throws `ERR_PACKAGE_PATH_NOT_EXPORTED`. Reading the
file directly is exports-map independent and *cannot execute package code*.

That is entirely consistent with the packaging rule — `bin/**` must have zero external
requires — and it is a genuinely nice trick: declare a dependency so the *file* is on disk,
then read it as data.

!!! danger "Defect: the declared range cannot satisfy the repo's own floor"
    The walk is `for (const start of [cwd, __dirname])`
    (`src/claude-orchestration-command-router.cts:128`). When the user's project tree has no
    SDK, it falls through to `__dirname` — **GSD's own `node_modules`**.

    `package.json:63` pins `^0.2.84`. Caret on a `0.x` version pins the *minor*: npm treats
    `0.y.z` as having a breaking change at every minor bump, so `^0.2.84` resolves only
    within `0.2.x` and can never reach `0.3.0`.

    The claude-orchestration floor is `0.3.149` —
    `src/claude-orchestration.cts:53` (`WORKFLOW_TOOL_FLOOR_VERSION = '0.3.149'`), with the
    same default in `capabilities/claude-orchestration/capability.json:50-53`.

    So on the `__dirname` fallback path, gate 5 (`src/claude-orchestration.cts:243-247`)
    always returns `agent_sdk_version_below_floor`, and the Workflow backend can never
    activate from GSD's own dependency tree. The fallback that exists to make the feature
    work in more places is the one that guarantees it cannot.

    The general form: **a version floor and a dependency range are the same fact expressed
    twice, in files that no check compares.** This repo has generators and `--check` lints
    for exactly this class of duplication — `sync-manifest-versions.cjs`, the `gen-*.cjs`
    `--check` family. This pair just never got one.

## What I would do differently

Five things, in rough order of how cheaply they transfer.

**Put the payload/packaging boundary in a directory, and say why in the file most likely to
be misfiled.** `bin/` versus `gsd-core/bin/` costs nothing and is self-enforcing, and
`bin/gsd-mcp-server.js:5-9` is a model of how to document a rule *at the point of
temptation* rather than in an ADR nobody rereads.

**Make the source format and the distribution format different on purpose.** 46 pack
directories in, one 315 KB validated table out. Authors get files; the runtime gets a
single artifact with invariants already resolved and no filesystem walk.

**Give the lint's scope the same review as the lint's rule.** `SCAN_ROOT = 'hooks'` is the
whole defect here — the rule was right, the blast radius aged. A scope constant with a
comment naming what is *deliberately* out of scope turns an oversight into a decision.

**Name symbols, never line numbers, across files.** Four stale pointers in shipped code,
one of them off by ~5,000 lines and pointing at unrelated TOML parsing. A symbol name is
greppable and repairs itself under refactoring.

**Assert cross-file duplicated facts, or expect them to drift.** The `^0.2.84` versus
`0.3.149` mismatch, the two byte-identical marker constants, and `ws` being cited in prose
while imported nowhere are three instances of the same failure: a fact recorded in two
places with nothing comparing them. This repo already has the machinery — a `--check` mode
on a generator, a parity test — it just was not pointed at these pairs.

And one that is less a recommendation than an observation. `bin/install.js` is 642 KB in a
single file, roughly a quarter of it a bespoke TOML implementation for one runtime, and the
repair for its untestability is an environment variable that suppresses `main`. Every
individual decision inside it is defensible — splice-don't-reserialise, marker-delimited
ownership, mandatory confinement roots, SHA-256 manifests. The file is still what happens
when a hundred defensible decisions accumulate in one place with no forcing function to
split it. The runtime next door is 194 modules with lint-enforced seams. The difference is
not skill; it is that `src/*.cts` had a build step and a set of guards that made the module
boundary the path of least resistance, and `install.js` never did.
