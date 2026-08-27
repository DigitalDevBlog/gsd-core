# Runtime

Every AI coding framework is a pile of markdown. GSD Core is a pile of markdown plus
194 TypeScript modules, and the interesting question is not *how much* code there is —
it is **where the code stops**.

This section has no equivalent in a pure prompt framework. There is nothing to compare it
to in PAUL or in any of the prompt-only spec-driven kits. That absence is the point: this
page is where you decide what deserves real code in *your* framework, and GSD Core is a
useful subject precisely because it drew the line somewhere unusual and left evidence of
why.

!!! info "Reading this as construction"
    Nothing below tells you how to run `gsd-tools`. It answers: what did the authors
    choose to make deterministic, what did they leave to the model, what shape did the
    resulting code take, and where did that shape fail them.

## What you can actually see

Before the map, a warning about the terrain — because it changes what "reading the source"
means here.

The runtime's source of truth is `src/*.cts`. It is compiled to `gsd-core/bin/lib/*.cjs`
at **install, pack, publish and test time**, and the output is gitignored. From
`tsconfig.build.json`:

| Setting | Value | Consequence |
|---------|-------|-------------|
| `rootDir` / `outDir` | `src` → `gsd-core/bin/lib` | Flat 1:1 emit, no bundler |
| `module` / `moduleResolution` | `nodenext` | CommonJS semantics with modern resolution |
| `target` | `ES2022` | |
| `strict` | `true` | |
| `noEmitOnError` | `true` | **The type check is a hard build gate** — a type error produces no artifact at all |
| `include` | `src/**/*.cts` | |

The `.cts` extension is not decoration. It is chosen *because* `tsc` then emits `.cjs`
natively, which is what lets the project ship CommonJS to a Node CLI without Rollup,
esbuild or a bundler config. The price is paid in every import statement in the tree:

```ts
// src/roadmap.cts — the file on disk is ./clock.cts, but the specifier names the emit
import { realClock } from './clock.cjs';
```

`package.json` wires `npm run build:lib` (`tsc -p tsconfig.build.json`) into four
lifecycle hooks — `prepare`, `prepack`, `prepublishOnly` and `pretest`. So the artifact is
regenerated whenever anyone installs, packs, publishes or tests, and never committed.

!!! warning "The repo's own architecture doc is inconsistent about this"
    `docs/ARCHITECTURE.md` gets it right in some places — lines 323, 352, 362 and 399 all
    say "generated to `gsd-core/bin/lib/*.cjs` per ADR-457". But its section headings do
    not: line 304 is `### Command Routing Hub (gsd-core/bin/lib/command-routing-hub.cjs)`,
    naming a path that does not exist in a fresh clone. The directory tree at line 659
    lists `bin/lib/*.cjs   # Domain modules` with no such note, and the word "gitignored"
    appears exactly once in the whole document — about screenshots, not about this.

    A reader who clones the repo and greps `bin/lib/` finds eight files. **Cite
    `src/<name>.cts`.** If you must cite a `.cjs` under `gsd-core/bin/lib/`, say out loud
    that it is build output. Inconsistent provenance labelling is worse than none: it
    teaches the reader that an unlabelled path is a real one.

### The nine files that *are* committed there

`git ls-files gsd-core/bin/lib` returns exactly eleven entries. Eight are `.cjs`:
`capability-command-router`, `capability-registry`, `capability-validator`,
`legacy-cleanup`, `loop-host-contract`, `package-identity`,
`profile-pipeline-command-router`, `stale-bake-guard`. The other three are
`gsd-core/bin/lib/vendor/re2js.cjs`, its `.d.cts` and a README.

The obvious reading — "modules not yet migrated to ADR-457" — is wrong. I checked each of
the eight for a `src/*.cts` counterpart: **none exists.** They are not tsc output at all.
Most are *generator* output — `scripts/gen-capability-registry.cjs`,
`scripts/generate-package-identity.cjs`, `scripts/gen-loop-host-contract.cjs` — committed
because they must exist before a build runs. `package.json`'s `version` script even
`git add`s one of them by name:

```
"version": "… && node scripts/gen-capability-registry.cjs --write && git add gsd-core/bin/lib/capability-registry.cjs"
```

`src/package-identity.d.cts` is the tell: a hand-written type declaration in `src/` for a
generated artifact that has no implementation in `src/`.

!!! danger "Defect: the CI check that guards this guards nothing"
    `scripts/lint-compiled-artifact-sync.cjs` exists to catch a committed `.cjs` drifting
    from its `.cts` source — the incident in its own docstring is #2653, where
    `api-coverage.cjs` sat four days behind its `.cts` with CI green. It only compares
    *tracked* `gsd-core/bin/lib/*.cjs` files **that have a matching `src/*.cts`**. Since
    all eight tracked artifacts have no `src/` counterpart, the pair set is empty. Running
    it prints:

    ```
    ok compiled-artifact-sync: no tracked compiled artifacts (ADR-457 end state)
    ```

    It is wired into `lint:ci` via `lint:generated-sync`, so CI executes a vacuous check on
    every run. Its own test suite asserts the empty-set path
    (`tests/lint-compiled-artifact-sync.test.cjs`). The docstring anticipates this
    ("passes trivially"), which makes it a *deliberate* no-op — but a green check named
    after a defect class it can no longer detect is worse than no check, because it reads
    as coverage.

## The module map

`src/` is not a package tree. It is a flat bag of 194 top-level `.cts` files (one of which,
`package-identity.d.cts`, is a declaration, leaving 193 implementations) plus five
subdirectories holding 26 more: `health-diagnostic-rules/` (10), `installer-migrations/`
(10), `observability/` (3), `host-integration-adapters/` (2), `vendor/` (1). There are no
`index` barrels, no directory-per-feature, no import boundaries. **The only grouping
mechanism in the repo is the filename prefix.**

The nine families below are therefore *my* overlay, not the repo's. Every module is
assigned to exactly one; the counts sum to 194.

| Family | n | Representative modules | What it owns |
|--------|---|------------------------|--------------|
| Loop state machine | 34 | `state.cts`, `state-transition.cts`, `phase.cts`, `phase-lifecycle.cts`, `roadmap-parser.cts`, `planning-workspace.cts` | Reading and writing `.planning/`, phase/milestone/workstream identity, locking |
| Quality gates & verification | 33 | `verify.cts`, `uat-predicate.cts`, `audit.cts`, `review-lane-runner.cts`, `broken-windows.cts`, `gate-predicate-evaluator.cts` | Deciding whether an artifact the model produced actually satisfies a gate |
| Install & host adaptation | 32 | `init.cts`, `install-engine.cts`, `runtime-artifact-conversion.cts`, `runtime-hooks-surface.cts`, `host-integration.cts`, `codex-agent-toml.cts` | Projecting the same framework onto Claude Code, Cursor, Codex, Gemini, Kilo, … |
| Dispatch & process contract | 27 | `io.cts`, `command-routing-hub.cts`, `cjs-command-router-adapter.cts`, `commands.cts`, 18 `*-command-router.cts` | Turning argv into a handler and a handler into an exit code |
| Knowledge, research & side workflows | 17 | `graphify.cts`, `intel.cts`, `research-store.cts`, `decisions.cts`, `learnings.cts` | Everything orbiting the loop rather than driving it |
| Capability & MCP system | 16 | `capability-loader.cts`, `capability-trust.cts`, `capability-consent.cts`, `mcp-catalog.cts` | Third-party extension packs and their trust boundary |
| Text & document primitives | 13 | `markdown-sectionizer.cts`, `frontmatter.cts`, `markdown-table.cts`, `text-lines.cts`, `pattern.cts` | The one implementation of each parsing concern |
| Config & model resolution | 11 | `config-loader.cts`, `model-resolver.cts`, `model-catalog.cts`, `federated-config.cts` | Layered config and which model runs which role |
| OS, process & VCS seams | 11 | `shell-command-projection.cts`, `clock.cts`, `project-root.cts`, `worktree-safety.cts` | Everything platform-dependent, isolated so it can be lint-enforced |

```kroki-plantuml
@startuml
skinparam componentStyle rectangle
skinparam shadowing false
skinparam defaultTextAlignment center

package "Boundary: the code-to-model interface" {
  component "Dispatch & process contract\n27 modules\nio.cts · command-routing-hub.cts\n18 *-command-router.cts" as famDispatch
}

package "Drivers" {
  component "Loop state machine\n34\nstate.cts · phase.cts\nplanning-workspace.cts" as famLoop
  component "Quality gates\n33\nverify.cts · uat-predicate.cts\ngate-predicate-evaluator.cts" as famGates
  component "Install & host adaptation\n32\ninit.cts · install-engine.cts\nruntime-artifact-conversion.cts" as famInstall
  component "Capability & MCP\n16\ncapability-loader.cts\ncapability-trust.cts" as famCaps
  component "Knowledge & side workflows\n17\ngraphify.cts · intel.cts" as famIntel
}

package "Shared seams (one definition per invariant)" {
  component "Text & document primitives\n13\nmarkdown-sectionizer.cts\nfrontmatter.cts · pattern.cts" as famText
  component "Config & model resolution\n11\nconfig-loader.cts\nmodel-resolver.cts" as famConfig
  component "OS, process & VCS seams\n11\nshell-command-projection.cts\nclock.cts · project-root.cts" as famOs
}

famDispatch --> famLoop
famDispatch --> famGates
famDispatch --> famInstall
famDispatch --> famCaps
famDispatch --> famIntel

famLoop --> famText
famGates --> famText
famCaps --> famText
famIntel --> famText

famLoop --> famConfig
famInstall --> famConfig
famCaps --> famConfig

famLoop --> famOs
famInstall --> famOs
famGates --> famOs

note bottom of famDispatch
  Every family returns through io.cts:
  output() writes fd 1, error() writes
  fd 2 and calls process.exit(1).
  The envelope is the return channel.
end note

note right of famText
  Analytical overlay, not repo structure.
  src/ is flat; the only real grouping
  is the filename prefix. Nothing
  enforces these boundaries.
end note
@enduml
```

### Size is brutally skewed

Eight modules exceed 100 KB in that flat directory:

| Module | Size |
|--------|------|
| `src/state.cts` | 261 KB |
| `src/init.cts` | 168 KB |
| `src/runtime-artifact-conversion.cts` | 168 KB |
| `src/phase.cts` | 160 KB |
| `src/runtime-hooks-surface.cts` | 119 KB |
| `src/commands.cts` | 117 KB |
| `src/state-transition.cts` | 109 KB |

Against dozens of sub-2 KB leaves: `src/model-profiles.cts` (965 B),
`src/eval-command-router.cts` (996 B), `src/secrets.cts` (1.1 KB),
`src/planning-scope.cts` (2.1 KB). There is no intermediate structure to break the large
ones against — no package boundary a `state/` split could hide behind, no barrel to
re-export through. This is the cost of the flat-bag choice, paid at exactly the modules
where it hurts most.

## The central argument: what got real code

Read the family table sideways and the rule falls out. GSD Core wrote deterministic code
for five things, and only five:

**1. Parsing and serializing the model's own output.** The model writes markdown; the
runtime must read it back without ambiguity. `src/markdown-sectionizer.cts` (48 KB),
`src/frontmatter.cts` (47 KB) and `src/markdown-table.cts` (39 KB) exist because "just
regex the file" fails. The clearest statement of the principle is in
`src/uat-predicate.cts`'s docstring — it exists to harden against

> the naive whole-file regex in `cmdPhaseComplete` which false-matches `result:` lines
> inside frontmatter, fenced code blocks, blockquotes, and HTML comments.

The model's *judgement* about whether UAT passed is trusted. The runtime's *reading* of
that judgement is not left to a regex. That asymmetry is the whole design.

**2. State transitions.** `src/state-transition.cts` and `src/phase-lifecycle.cts` make
"which stage am I in, and may I move to the next one" a total function over typed inputs.
An LLM asked to reason about its own position in a workflow will drift; a state machine
will not.

**3. Concurrency.** `src/planning-workspace.cts` owns `withPlanningLock`, backed by a
`.planning/.lock` file whose body is JSON `{ pid, cwd, acquired }`. Stealing a lock is
gated on `process.kill(pid, 0)` liveness (treating `EPERM` as alive), so a dead holder is
stolen promptly and a live one waited on. `PLANNING_LOCK_RETRY_ERRNOS` enumerates eight
transient codes — `EPERM`, `EBUSY`, `EAGAIN`, `EINTR`, `EINVAL`, `EIO`, `ENOENT`,
`ESTALE` — each with a per-platform rationale (Docker overlayfs, NFS, Windows AV), and
deliberately *excludes* `EMFILE`, `ENOSPC`, `EROFS`, `EACCES`. No prompt can encode that.

**4. Platform portability.** `src/shell-command-projection.cts` (53 KB) centralises
`platformWriteSync`, `platformReadSync`, `platformEnsureDir`, `retryRenameSync` and
Windows binary resolution.

**5. Trust and security boundaries.** `src/capability-trust.cts` (92 KB),
`src/capability-consent.cts` (54 KB) and `src/external-descriptor-trust.cts` decide whether
third-party code may run. Consent binds to a *recomputed* full-bundle content hash rather
than the ledger's own integrity field (#1459), so a planted ledger entry does not grant
consent. `destSubpaths` outside `configHome` reject the descriptor outright, fail-closed
(#1681).

Everything else — what to build, whether a plan is sound, what the risks are, how to phrase
a review finding — lives in `commands/gsd/*.md` (71 files) and `agents/*.md` (35 files), not
in `src/`.

!!! quote "The transferable rule"
    **Predicates are code; the content they gate is prose.**
    `src/gate-predicate-evaluator.cts`, `src/uat-predicate.cts` and
    `src/context-predicates.cts` implement decision *mechanics*. The decision *substance*
    is workflow markdown. If you are drawing this line in your own framework, that is the
    cut to copy: write code for anything whose answer must be identical on two runs, and
    leave everything whose answer is a judgement to the model.

### `src/io.cts` is the boundary artifact

If you read one file to understand the whole position, read this one. It defines the
CLI envelope, and the CLI envelope *is* the interface between the deterministic layer and
the model.

`output(result, raw, rawValue)` writes fd 1. `error(message, reason, extra): never` writes
fd 2 and calls `process.exit(1)`. Routers such as `src/agent-command-router.cts` return
`void` and terminate — **the envelope, not a return value, is the return channel for most
of the system.**

Three details carry the argument:

- **`ERROR_REASON` is a frozen string enum**, and its stated purpose is that tests assert
  on a typed field instead of grepping stderr (#2974). `--json-errors` flips `error()` into
  emitting `{ ok, reason, message, ...extra }`, and the `extra` bag exists specifically so
  a caller can assert a structured field rather than regex a human sentence. The same
  pattern recurs in `src/command-routing-hub.cts` (`ERROR_KINDS`) and
  `src/agent-command-router.cts` (`AGENT_FAILURE_CLASSES`) — `Object.freeze({` appears in
  41 of the 194 modules. **Never make phrasing load-bearing** is the position, and it is
  stated four separate times in four separate files.
- **`writeAllSync` treats stdout as a possibly-non-blocking pipe.** It loops on short write
  counts and retries `EAGAIN`/`EINTR` with a bounded ~1 s `Atomics.wait` backoff. A bare
  `fs.writeSync` silently truncated output under the parallel `node --test` runner (#1008).
  `output()` deliberately does *not* call `process.exit()`, so the event loop drains.
- **The `@file:` escape hatch encodes an assumption about the host, not the data.** At
  `src/io.cts:153`, a JSON payload over 50 000 characters is written to
  `$TMPDIR/gsd/gsd-<ts>.json` and the string `'@file:' + tmpPath` emitted instead.
  `gsd-core/bin/gsd-tools.cjs` — the hand-written, tracked CLI entrypoint, not a build
  artifact — intercepts stdout at lines 4275–4329 and resolves it transparently. The comment says why: without it, "every workflow" would need its own
  `if [[ "$INIT" == @file:* ]]` branch (#1891). The number 50 000 is Claude Code's Bash tool
  buffer — a *host* constraint leaking into a *library* — and centralising the leak in one
  function is the right call.

## What the code looks like once you're inside it

Four conventions recur often enough to be the house dialect.

### Two export styles, split by role

110 of the 194 top-level modules use `export =` (103 of them the exact `export = { … }`
literal form). The remaining 84 do not: 82 use ESM named exports (`export const` /
`export function` / `export class`), `src/configuration.cts` uses an `export { … }` list,
and `src/package-identity.d.cts` is a declaration file. The two sets are **disjoint** — no
module mixes them.

The tempting hypothesis is that `export =` marks pre-ADR-457 legacy. The crosstab
falsifies it: 107 modules carry an `ADR-457` reference and 61 of those use `export =` — a
57 % rate against a 57 % base rate across all 194. That is independence, not correlation.
What *does* track is role — as a strong tendency, not an absolute. **Thirty of the 35 modules that
define a `cmd*` CLI entry point use `export =`. Zero use ESM named exports.** So do all 19
files matching `src/*command-router*.cts`, without exception. (`src/planning-inspect.cts`
is one of seven that write `export =` against a named object rather than
an inline literal — a formatting difference, not a different style.) Pure leaves —
`src/clock.cts`, `src/pattern.cts`, `src/text-lines.cts`, `src/artifacts.cts`,
`src/markdown-table.cts` — use ESM.

The reason is mechanical: `export =` exports one namespace object, which is what lets a
module ship mutable test-seam setters and frozen enums through the same handle.

The cost propagates into consumers. An `export =` module cannot be ESM-destructured, so
callers must write `import x = require('./x.cjs'); const { … } = x;`, each occurrence
preceded by `// eslint-disable-next-line @typescript-eslint/no-require-imports`. **115 of
194 modules carry that disable.** `src/roadmap.cts` lines 9–51 shows both styles
interleaved in a single import block — ESM for `clock.cjs`, `pattern.cjs`,
`text-lines.cjs`, `markdown-sectionizer.cjs`, `markdown-table.cjs`, `phase-lifecycle.cjs`,
`shell-command-projection.cjs`; `import … = require()` for `io.cjs`, `phase-id.cjs`,
`planning-scope.cjs`, `roadmap-parser.cjs`, `config-loader.cjs`, `core-utils.cjs`,
`frontmatter.cjs`, `verification.cjs` and more.

### Test seams shipped in production

Underscore-prefixed setters on the module's export object: `_setLockProbes`,
`_resetLockProbes`, `_setPlanningLockTestHooks`, `_setStateLockTestHooks`,
`_setValidatorForTest`, `_setGeneratorForTest`, `_setHttpGet`,
`_setCapabilitySourceHttpGet`, `_setBundleMaxFilesForTest`, plus seven `_reset*ForTests`
cache clearers — **22 distinct `_set*`/`_reset*` identifiers** across `src/*.cts`. It is an explicit convention;
`src/intel-command-router.cts` lines 25–29 says the `_`-prefix "follows the repo's
established seam convention (see `audit-command-router.cts`)".

This buys deterministic tests for pid-liveness, lock-steal races and HTTP without a DI
container. It costs mutable module-level singletons in shipped code. For a CLI that runs as
a short-lived process, that trade is defensible; for a long-lived server it would not be.

### Clock injection

`src/clock.cts` is imported by 14 modules, and twelve signatures across seven modules take
an optional `clock?` parameter (`withPlanningLock(cwd, fn, clock?)`). Time is treated as a
dependency, not an ambient.

### Inline issue provenance

Nearly every non-obvious branch carries `#NNNN` or `ADR-NNNN` in a comment naming the
failure it was written against. This is the single most imitable habit on the page: it turns
the codebase into a searchable incident log, and it is why this write-up can cite a cause
for almost every design decision.

## Lint is the architecture, not the type system

`noEmitOnError: true` makes TypeScript a hard gate, but the invariants that actually hold
this system together are not expressible in types. So they were built as lint rules.

`eslint-rules/` contains **22 bespoke rules**. The `src/**/*.cts` block of
`eslint.config.mjs` (line 341) promotes **nine** to `'error'`:

Each carries its own provenance comment in the config. Quoting them, because the comments
*are* the argument:

| Rule | Provenance | What the config comment says it catches |
|------|-----------|------------------------------------------|
| `local/no-adhoc-markdown-parsing` | ADR-1372 T7 | "enforce use of the markdown-sectionizer seam; grandfather pre-migration sites with `// allow-adhoc-markdown: <reason>`" |
| `local/no-adhoc-regex-escape` | ADR-3212 Phase 1, #3412 | "flags a re-inlined escape-all-metachars `.replace()` helper or an unrouted `new RegExp()` from a runtime value" |
| `local/no-crlf-fragile-split` | ADR-3212 Phase 2, #3413 | "widen the CRLF-fragile-split prohibition from `tests/` to `src/`" |
| `local/no-unbounded-quantifier` | ADR-3212 Phase 4, #3415 | "bound quantifiers over document content (CWE-1333, #2128 class)" |
| `local/normalize-path-in-content` | ADR-1703 Phase 5 | "flag path-returning calls interpolated into content … without POSIX normalization"; promoted to error "after precision review (`path.basename` excluded)" |
| `local/require-fs-op-fallback` | ADR-1703 Phase 6 | "an unguarded `fs.rename`/`fs.renameSync` … that lacks a transient-errno fallback (EPERM/EBUSY/EACCES retry or a Windows platform guard). See DEFECT.WINDOWS-FS-OPS" |
| `local/require-subprocess-timeout` | DEFECT.UNBOUNDED-SUBPROCESS | "`execSync`/`execFileSync`/`spawnSync` without a `timeout` option — an unbounded sync subprocess hangs indefinitely on a stuck remote/large repo/missing network. The 8 pre-existing call sites this surfaced were migrated in #2896" |
| `local/no-external-require-in-bin` | #3477 follow-up | "the emitted mirror is almost always eslint-ignored as a generated artifact … so this is the ONLY place a bad external import in an already-migrated module is still visible to lint" |
| `local/no-private-binary-resolution` | #3619, epic #3411 Phase 3 | "a re-implemented Windows binary resolver — a PATHEXT read or a hardcoded exe-extension list — outside the platform seam (`src/shell-command-projection.cts`, exempt by path)" |

Two things to notice. First, each of the first five names a *seam module* and makes
re-inlining its job a build failure rather than a review catch. Second — read the
`no-external-require-in-bin` entry again. It exists **because** the emitted `.cjs` mirrors
are eslint-ignored, so `src/**/*.cts` is the only surface where the rule can still see
anything. The build model and the lint model are load-bearing on each other, which is
exactly why the three missing ignore entries above matter more than they look. That is the mechanism that keeps "one
definition per invariant" true over years. On top of it, `lint:ci` chains **28** separate
`node scripts/lint-*.cjs` drift checkers (of 40 such scripts in `scripts/`) —
`lint-state-field-drift`, `lint-completion-predicate-drift`,
`lint-planning-snapshot-bypass-drift`, `lint-phase-enumeration-drift`,
`lint-health-diagnostic-rule-table` and more.

!!! tip "The lesson for your framework"
    Every one of these rules cites the incident that motivated it. That is the discipline
    worth stealing: when a class of bug recurs, do not write a style guide entry — write a
    lint rule and name the issue in its comment. GSD Core's 22 rules are its actual
    architecture document.

### But the biggest structural risk has no checker at all

Twenty-two bespoke ESLint rules, 28 chained drift checkers — and **zero import-cycle
tooling**. No `import/no-cycle`, no `madge`, no `dependency-cruiser` anywhere in
`eslint.config.mjs` or `package.json` (verified: zero occurrences in either file).
Acyclicity across 220 `.cts` files with two interleaved import styles is maintained by prose.
`src/roadmap.cts:37-38`:

```
#3641: milestone-scope's convention resolution reads the project config
(no cycle — config-loader does not import this module)
```

That invariant is asserted in a comment and checked by nothing. In a flat directory with no
package boundaries, the *only* thing preventing a cycle is a developer remembering to write
that comment — and the only thing keeping the comment true is that nobody edits
`config-loader.cts`. Given how thoroughly everything else here is mechanised, this gap is
striking. **If you build a flat module bag, buy the cycle checker on day one.**

## The dispatch refactor, caught mid-flight

`src/command-routing-hub.cts` is the newest and cleanest thing in the tree. Its docstring
declares hard invariants: it never prints, never calls `process.exit`, never throws. Errors
are a closed taxonomy — `UnknownCommand`, `InvalidArgs`, `HandlerRefusal`, `HandlerFailure`
— every variant is `Object.freeze`d, and a runtime `_VARIANT_SCHEMA` rejects handler results
carrying fields outside their variant's allowed set. It is a pure dispatch core, fully
testable without spawning a process.

Then it is undone at the boundary. `src/cjs-command-router-adapter.cts` takes `error` as an
**injected callback** and translates the hub's typed `InvalidArgs.exitReason` straight back
into `error(msg, reason)` — which exits the process. The `error` parameter is declared on
`RouteCjsCommandFamilyOptions` at `src/cjs-command-router-adapter.cts:49`, widened by
amendment #1642 to carry an `ERROR_REASON` value. Legacy callers are routed through a
synthetic family literal `'__legacy_cjs_family__'` (line 80).

So the closed taxonomy and the no-exit invariant hold *inside the core* and evaporate at the
process edge. That is a defensible strangler-fig position — you get typed, testable
internals now and change the process contract never — but it means the hub's guarantees do
not describe what the CLI actually does.

**The migration is roughly a fifth done.** Of the 19 files matching `*command-router*` in
`src/` (18 concrete routers plus the adapter), exactly four routers import the hub:
`graphify-`, `intel-`, `phase-`, `refactor-trigger-`. Nine of the concrete routers still
import `io.cjs`, and thirteen of the eighteen contain a bare `error(` or `output(` call.
And `gsd-core/bin/gsd-tools.cjs` lines 304–326 `require()`s **14 routers directly by
path**, bypassing the hub entirely.

The adapter's own #2620 comment (`src/cjs-command-router-adapter.cts:16-19`) records the cost of a half-migration: observability was
**inert in the live CLI for an unknown period**, because the hub defaults to a no-op logger
and the dispatch path never injected the real one. `GSD_AUDIT` and `config.audit.enabled`
did nothing. A refactor that lands the abstraction but not the wiring produces exactly this
failure mode, and it is invisible — nothing errors, telemetry is simply absent.

## Three defects worth reading the source for

The passages above are design tensions. These three are plain bugs, and each is instructive
about a different failure mode.

!!! danger "A comment that contradicts the code, with a real consequence"
    `src/audit.cts:1669`:

    ```ts
    ioError(`unknown --category "${category}". Available: …`);
    return; // unreachable — ioError throws — satisfies TS control-flow analysis
    ```

    `ioError` is bound at `src/audit.cts:36` as `const { output, error: ioError } = io;` —
    it is `io.cts`'s `error()`, which does **not** throw. At `src/io.cts:253` it calls
    `process.exit(1)`.

    The rest of the repo knows this. `src/state.cts:241-242` documents the correct
    behaviour precisely — "`process.on('exit')` fires even on `process.exit(1)`, unlike
    `try/finally` which is skipped when `error()` calls `process.exit(1)` inside a locked
    region (#1916)" — and `src/intel-command-router.cts:31-33` says the same. So the
    consequence is concrete: a developer trusting the `audit.cts` comment would wrap the
    call in `try/catch` expecting to intercept the failure. It will not fire, and any
    `finally` cleanup in that frame is skipped.

!!! danger "A hand-maintained ignore list that is already missing entries"
    `eslint.config.mjs` carries an ignores block listing `gsd-core/bin/lib/*.cjs` paths
    **individually** — there is no `gsd-core/bin/lib/**` glob. Three emitted mirrors have
    no entry anywhere in the file:
    `host-integration-adapters/cline-sdk-binding`, `host-integration-adapters/imperative-hook-bus`,
    and `installer-migrations/006-pi-extension-cjs-to-js` — while sibling migrations 000–005
    and 007–009 all do (lines 132, 152–153, 195–198, 202, 205).

    Their emitted `.cjs` therefore falls under the `gsd-core/bin/**/*.cjs` block at line 435
    and gets linted as if hand-written CommonJS. There are zero *stale* entries, which
    proves the list is otherwise carefully curated — and that is the point. A
    hand-maintained mirror of a generated file set will drift; the design guarantees it. A
    glob would have cost one line.

!!! danger "Dead code masquerading as a narrowing"
    `src/artifacts.cts:34-36`:

    ```ts
    export const CANONICAL_PATTERNS: ReadonlyArray<RegExp> = [
      /^v\d+\.\d+(?:\.\d+)?-MILESTONE-AUDIT\.md$/i, // gsd-complete-milestone (pre-archive)
      /^v\d+\.\d+(?:\.\d+)?-.*\.md$/i,               // other version-stamped planning docs
    ];
    ```

    The second pattern fully subsumes the first — both match `v1.2-MILESTONE-AUDIT.md` — so
    entry 0 can never be the sole matcher. It documents provenance rather than narrowing
    anything, which is a reasonable intent, but a reader auditing `isCanonicalPlanningFile`
    would assume a real distinction exists.

## One more: the stated rationale that does not apply

`src/cli-exit.cts` provides `ExitError` and `runMain()`, which set `process.exitCode`
instead of calling `process.exit`. Lines 8 and 24 justify the whole apparatus with: exit is
"banned by `n/no-process-exit`".

It is not banned here. `n/no-process-exit` is registered at `eslint.config.mjs:470`, inside
the **CommonJS-files block**. The `src/**/*.cts` block at line 341 loads
`tseslint.configs.recommendedTypeChecked` plus nine `local/*` rules and **never loads the
`n` plugin at all**. There are 26 live `process.exit(` call sites across 15 `src/*.cts`
modules — `api-coverage`, `teams-status`, `io`, `ui-safety-gate`, `state`,
`assumption-delta` and others — carrying **zero** eslint-disable comments. They are
lint-legal. The rule also cannot reach the emitted mirrors, which sit in the ignores list.

The discipline `cli-exit.cts` exists to satisfy is not enforced on the code it lives among.
The module is still good practice; its stated reason is scope-wrong, and a reader who
believes it will draw a false conclusion about what the linter guarantees.

## What to take into your own framework

| Decision | GSD Core's answer | Worth copying? |
|----------|-------------------|----------------|
| What gets deterministic code | Parsing, state transitions, concurrency, portability, trust | **Yes.** All five are places where two runs must agree. |
| What stays prose | Judgement: what to build, is this plan sound, what are the risks | **Yes.** Code cannot do it and shouldn't fake it. |
| Errors as data | Frozen `reason` enums + `--json-errors`; tests assert fields, never stderr text | **Yes**, unambiguously. Cheapest high-leverage decision here. |
| Build-at-publish, gitignored output | `.cts` → `.cjs` via `tsc`, no bundler; `noEmitOnError` gates | **Yes for shipping**, but document loudly that `bin/lib/` is generated — this repo didn't, and its own docs mislead. |
| Lint rules over style guides | 22 bespoke rules, each naming its incident | **Yes.** This is the actual architecture doc. |
| Flat module bag | 194 top-level files, prefix-only grouping, 8 modules over 100 KB | **No.** Eight six-figure files with nowhere to split them is the direct cost. |
| No cycle checking | Acyclicity asserted in comments | **No.** If you go flat, buy the checker on day one. |
| Test seams in shipped code | 22 `_`-prefixed setters on export objects | **Situationally.** Fine for a short-lived CLI; a hazard in a long-lived process. |
