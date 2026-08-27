# Multi-harness portability

GSD Core installs itself into nineteen different agent harnesses. Not "supports" — installs:
converts its command corpus into their file formats, writes their config files (eighteen of
them; `vscode` declares `configHome.kind: "none"` and `localConfigDir: null`, and binds as an
extension instead), registers hooks against their event vocabularies, and uninstalls cleanly
afterwards. Nineteen is the
count of capability packs that declare a `runtime.hostIntegration` block
(`grep -rl '"hostIntegration"' capabilities/` → 19 files, out of 46 packs total; the other
27 split 22 `role: feature` and 5 `role: reviewer`).

That is an unusual amount of portability for a framework with one maintainer. This page is
about how it is constructed, and about the bill. The short version of the bill: the
abstraction that was supposed to make the second harness cheap was extracted from the first
harness, and when the second one arrived it did not fit.

## The axis that does not select anything

Start with the design as advertised. Every runtime pack declares an `embeddingMode`, one of
two values:

```
grep -rh '"embeddingMode"' capabilities/*/capability.json | sort | uniq -c
   8  "embeddingMode": "declarative"
  11  "embeddingMode": "imperative"
```

The framework ships one adapter per mode. `src/adapter-declarative.cts` (42 lines) and
`src/adapter-imperative.cts` (85 lines) are the public embedding adapters — both generic over
`runtime`, both `Object.freeze`d, both fail-closed at construction with a `TypeError` when
`runtime` is not a non-empty string, and both delegating to `install-engine`'s
`installRuntimeArtifacts` / `uninstallRuntimeArtifacts` rather than reimplementing anything.
`docs/how-to/author-a-host-plugin.md:25` publishes the routing table: `programmatic-cli` hosts
take the imperative adapter, `declarative-cli` hosts (its examples: Antigravity, Codex) take
the declarative one.

Now look at what the installer does.

```js
// bin/install.js:695-700
function _runtimeAdapter(runtime) {
  try {
    return createImperativeAdapter({ runtime });
  } catch {
    return null;
  }
}
```

No branch. `_runtimeAdapter` is the single factory both call sites use — install at
`bin/install.js:10693-10706` and uninstall at `bin/install.js:8132-8137`, each using the
adapter when non-null and falling open to the engine directly when the composed registry
cannot load. So `codex`, `copilot`, `antigravity`, `windsurf` and the four other packs that
declare `"embeddingMode": "declarative"` all install through the **imperative** adapter.

!!! warning "State the scope of the claim precisely"
    The axis is inert *for adapter selection inside this repo's installer*. It is not inert.
    Grepping `embeddingMode` across `src/`, `bin/`, `scripts/`, `hooks/` and `vscode/` finds
    exactly three kinds of consumer: negotiation and profile classification in
    `src/host-integration.cts` (the axis is negotiated at `:402`, and `profileOf` at `:278-280`
    turns it into `ide` / `programmatic-cli` / `declarative-cli`), schema validation in
    `scripts/registry-schema.cjs:93`, and one doc comment at `vscode/host-binding.js:25`. The
    axis drives *classification and negotiation*. It does not drive *dispatch*.

That distinction matters because the two adapters are not equivalent.

## The two adapters are not interchangeable

Put the install bodies side by side. Declarative:

```ts
// src/adapter-declarative.cts:30-36
installEngine.installRuntimeArtifacts(
  runtime,
  intent.configDir,
  intent.scope,
  intent.resolvedProfile,
  intent.resolveAttribution,
);
```

Imperative:

```ts
// src/adapter-imperative.cts:72-79
installEngine.installRuntimeArtifacts(
  runtime,
  intent.configDir,
  intent.scope,
  intent.resolvedProfile,
  intent.resolveAttribution,
  registry,
);
```

Five arguments versus six. The sixth is the composed capability registry from
`loadRegistry({includeInstalled: true})`, and the comment immediately above it
(`src/adapter-imperative.cts:65-71`, issue #2322) says what the omission costs:

> without this the adapter path never staged a third-party capability skill regardless of
> registration.

The declarative adapter never loads a registry at all. It has no `capability-loader` import.

Every in-repo path gets the registry: the primary path because it goes through the imperative
adapter, and the fail-open fallback because `bin/install.js:10705` threads
`_installedCapabilityRegistry` explicitly with its own `#2322` comment. The only consumer that
would ever take the declarative branch is an external host plugin following the documented
routing table — and that plugin silently loses third-party capability-skill staging, with no
error and no warning.

!!! danger "This is a portability defect, not a dead-code curiosity"
    Both functions are re-exported as public, versioned SDK surface at
    `src/host-integration-sdk.cts:47-48`, whose header calls the module "the contract:
    everything it re-exports is public + versioned". And the declarative adapter *is*
    exercised in-repo, as a reference contract: `tests/declarative-reference-codebuddy.test.cjs:80`
    constructs `createDeclarativeAdapter({runtime: 'codebuddy'})` and asserts the
    classification. So the divergence is not two dead functions drifting — it is a live
    contract with a test that checks the shape and not the behaviour.

### The justification for interchangeability was deleted

Why would two adapters with different arities be presented as one interface? Because of a
claim that used to be checked and no longer is. Three separate places in this repo assert
byte-identity, each citing the same gate:

| Site | Claim |
|---|---|
| `src/adapter-declarative.cts:12` | "byte-identical to today's install (the link is gated by `tests/golden-install-parity.test.cjs`)" |
| `src/embedding-adapter.cts:62` | "to today's install is gated by `tests/golden-install-parity.test.cjs`" |
| `bin/install.js:10691` | "byte-identical output (gated by golden-install-parity)" |

`ls tests/golden-install-parity.test.cjs` → *No such file or directory*. So does
`tests/fixtures/golden-install-parity/`.

The deletion was deliberate and recorded — `tests/ci-test-scope.test.cjs:609` and `:643` both
name it: *"deleted by #2724 (ADR-2719 Phase 4)"*, and `tests/emitted-attribution.test.cjs:8`
declares itself the differential-attribution successor. What is interesting is that the repo
already knows the name is historical: `tests/check-glossary-refs.test.cjs:334` registers the
string as `'Historically \`tests/golden-install-parity.test.cjs\`...'` inside a drift-checking
mechanism, and the same file at `:336` treats a *typo* sibling of the name as real drift. The
repo built a machine to catch stale references to this exact filename, and three docstrings
carrying the stale reference sailed past it — because the machine checks glossary entries, not
comment prose.

The lesson generalises past this repo: **when you retire a test that was the stated
justification for an equivalence, the equivalence claims are now unsourced assertions.** Grep
for the test name, not just for the code it covered.

## What varies per harness

Here is every runtime pack with the three negotiated axes that vary most, plus one field that
is not an axis at all — read out of the descriptors:

| Pack | `embeddingMode` | `hookBus` | `hooksSurface` | `commandSurface` |
|---|---|---|---|---|
| antigravity | declarative | host | settings-json | slash-file |
| augment | declarative | host | settings-json | slash-file |
| claude | imperative | host | settings-json | slash-file |
| cline | imperative | host | cline-rules | slash-file |
| codebuddy | declarative | host | settings-json | slash-file |
| codex | declarative | host | codex-hooks-json | slash-file |
| copilot | declarative | host | copilot-inline | slash-file |
| cursor | imperative | host | cursor-hooks-json | slash-file |
| hermes | imperative | host | settings-json | slash-programmatic |
| kilo | imperative | host | none | slash-file |
| kimi | imperative | host | kimi-hooks-toml | slash-file |
| kimi-code | declarative | host | kimi-hooks-toml | slash-file |
| opencode | imperative | host | none | slash-file |
| pi | imperative | host | none | slash-programmatic |
| qwen | imperative | host | settings-json | slash-file |
| trae | imperative | **engine** | none | slash-file |
| vscode | imperative | **engine** | none | palette |
| windsurf | declarative | host | windsurf-hooks-json | slash-file |
| zcode | declarative | host | none | slash-file |

Read the columns rather than the rows. `commandSurface` collapses almost entirely —
16 `slash-file`, 2 `slash-programmatic`, 1 `palette`. `hookBus` collapses to 17 `host` / 2
`engine`. `hooksSurface` does not collapse at all: **eight distinct values across nineteen
packs**, five of them singletons.

!!! warning "The most variable dimension is the one the negotiation schema does not model"
    The first three columns are members of `HOST_INTEGRATION_AXES` (`src/host-integration.cts:20-56`,
    nine axes) and live under `runtime.hostIntegration`. `hooksSurface` is a sibling of that
    block, not a member of it — `'hooksSurface' in capability.runtime.hostIntegration` is
    `false` for `cursor`, and `grep -c "hooksSurface" src/host-integration.cts` returns **0**.

    Everything this page praises about negotiation therefore does not apply to it. No
    `negotiateScalar`, no `SAFE_DEFAULTS` floor, no `undocumented` sentinel, no
    "every effective scalar is in `engine.known[axis]`" post-condition. What guards it instead
    is a descriptor-time validator: `VALID_HOOKS_SURFACES`
    (`gsd-core/bin/lib/capability-validator.cjs:775`) is a closed 8-member set checked at
    `:1218-1224`, with a reserved-name guard against `__proto__`/`constructor`/`prototype` and
    a "GATE A" `installSurface` ↔ `hooksSurface` parity invariant at `:1607-1609`.

    That is a real gate, and it is a *different kind* of gate: it rejects at load time rather
    than degrading at negotiation time. Which is arguably right — you cannot fail-closed your
    way into speaking a TOML dialect — but it means the dimension carrying the most variance
    is also the one with no graceful-degradation story. And its own comment at `:1218` still
    calls it a "closed 7-enum (ADR-1016 Decision 5)"; `windsurf-hooks-json` made it eight.

Three more dimensions vary underneath the table:

**Converters.** Extracting every `"converter"` value from every `capabilities/*/capability.json`
yields 29 distinct names, declared by 17 of the 19 packs (`grep -rl '"converter"' capabilities/*/capability.json`).
Each has a matching `function <name>` definition in `src/runtime-artifact-conversion.cts`. Most
appear twice, because a runtime declares the same converter for its local and global
`artifactLayout` entries. The naming convention is strict —
`convertClaudeCommandTo<Host>Skill` / `convertClaudeAgentTo<Host>Agent` — with exactly two
exceptions, `convertClaudeToKiloFrontmatter` (`src/runtime-artifact-conversion.cts:1993`) and
`convertClaudeToOpencodeFrontmatter` (`:1831`), which take an options bag
`(content, { isAgent, modelOverride })` instead of the positional `(content, skillName)` and
are shared between a runtime's commands-kind and agents-kind.

There is one orphan. Diffing the 29 used names against the `VALID_CONVERTER_NAMES` allowlist in
`gsd-core/bin/lib/capability-validator.cjs:727-767` (a *committed* file, 30 entries) gives zero
used-but-not-allowlisted and one allowlisted-but-unused:
`convertClaudeCommandToWindsurfSkill`. It exists and is exported
(`src/runtime-artifact-conversion.cts:1186`, `:3570`) and no descriptor names it — Windsurf
declares `convertClaudeCommandToWindsurfWorkflow` instead. A converter written for a harness
concept that harness turned out not to have.

**Two event vocabularies, deliberately distinct.** `src/host-integration.cts:710-719` states it
outright:

> hookEvents = the managed-hook dialect (declarative hosts' settings.json); extensionEvents =
> the plugin-owned event subset (imperative hosts' extension API). They are not the same thing.

Ten packs declare `hookEvents`, five declare `extensionEvents`. A third vocabulary,
`extendedHookEvents`, is PascalCase (`SubagentStop`, `Stop`, `PreCompact`) while Cursor's
`managedHookEvents` is camelCase — and both can appear inside one descriptor, read by different
consumers.

**Hook protocol shape.** This is the one that does not abstract. Cursor blocks by writing
`{ block: true, reason }` to stdout; Windsurf's Cascade blocks by exiting 2 with a stderr
reason. Cursor's `hooks.json` entry is `{ type: 'command', command }`; Cascade's is a bare
`command` string with no `type` field and no top-level `version`. Cursor has a context-injection
channel (`additional_context`); Cascade has none. All four differences are documented in the
source at `src/runtime-hooks-surface.cts:1420-1442`, with citations to each vendor's docs.

## What stays shared

The genuinely shared layer is small, and it is worth being precise about what is in it, because
"shared" here means two different things.

**Shared by construction.** `src/host-integration-sdk.cts` is one frozen object and the
declared public contract. Its header:

> External host-plugin authors import ONLY from this entry. It IS the contract: everything it
> re-exports is public + versioned (PROTOCOL_VERSION governs the set); everything else in
> gsd-core is internal.

What it exposes: the closed axis vocabulary and protocol version, `negotiateHostCapabilities` +
`profileOf` + `degradationFor` + the event-surface lookups, five adapters
(`createDeclarativeAdapter`, `createImperativeAdapter`, `createModelAdapter`, `createHookBus`,
`createStateIO`), and a serialized handshake for out-of-process hosts. Underneath, both
embedding adapters call the *same* `installRuntimeArtifacts`, so the per-harness divergence
lives entirely in descriptor data consumed by one engine, not in nineteen install paths.

Negotiation is the strongest part of the design. `SAFE_DEFAULTS` (`src/host-integration.cts:127-137`)
is documented as the most restrictive known value per axis; `negotiateHostCapabilities` never
throws, returns a fresh object per call, and enforces the post-condition that every effective
scalar is a member of the known vocabulary. `subagentToolkit` returns `'full'` only on a literal
`'full'`. `namedDispatch: false` structurally caps `maxDepth`/`nested`/`background` to `0`/`false`.
And `UNDOCUMENTED = 'undocumented'` (`:26-33`) is a fully-plumbed corpus-wide sentinel that
"VALIDATES (accepted by the validator) but NEVER propagates into effective axes", with an
explicit instruction not to add it to `HOST_INTEGRATION_AXES` — so a descriptor author can say
*the vendor's docs are silent here* without that being indistinguishable from a malformed
descriptor. Issue #2603 added a sentinel-specific warning for the numeric `dispatch.maxDepth`
field precisely because the generic path reported it as "missing or not a number".

The vocabularies are also deliberately incomplete. `effortSurface` carries a comment
(`src/host-integration.cts:52-55`) explaining that a config-file-only member is *not* added
because the only host that ever had one was sunset: "Adding a member with no host would be a
guess." Name only what a host actually has.

**Shared by copy.** The same repo contains the counterexample. The `.planning/` write policy is
implemented three times from character-identical regexes:

| Site | Enforcement |
|---|---|
| `src/host-integration-adapters/cline-sdk-binding.cts:45,53,69` | **Blocks** (`{decision:'skip', reason}`) |
| `hooks/gsd-cursor-pre-tool.js:25-27` | Advises only — emits `additional_context`, never the `block`/`reason` its own header documents at line 14 |
| `hooks/gsd-cursor-post-tool.js:25-27` | Advisory by construction; the write already happened |

Three copies, three enforcement strengths, one policy. And the strictest copy is on the host
the framework cannot reach — grepping every export of `cline-sdk-binding.cts`
(`clineGsdPlugin`, `evaluateBeforeTool`, `resolveClineAgentModelParams`, `inferProviderId`,
`WRITE_TOOL_PATTERN`, `PLANNING_PATH_PATTERN`, `PLANNING_GUARD_REASON`,
`DEFAULT_CLINE_PROVIDER_ID`) across `src/`, `bin/`, `scripts/`, `hooks/`, `capabilities/`,
`vscode/` and `package.json` finds only the definitions plus two test files. The module is not
re-exported by the SDK. Its own header is candid about why — `@cline/sdk` is "not linked at
build/test time" — but the shipped effect is a decision-function library nothing calls.

The framework knows how to do this well when it tries. `OPENCODE_EXTENSION_EVENTS`
(`src/host-integration.cts:724-739`) is *aliased* to the `kilo` key rather than copied, with a
comment saying the alias — not a copy-paste — is what keeps the fork "pinned together by
construction". That is the same problem solved correctly, four hundred lines from the copies.

## The maintenance cost, honestly

### The abstraction was cut along the wrong seam

`src/host-integration-adapters/imperative-hook-bus.cts` (155 lines) exists to generalise
Cursor's hook wiring. Its header says so: it reads `hostBehaviors.managedHookEvents` from the
descriptor, "NOT a hardcoded sessionStart/postToolUse pair."

Three pieces of evidence say the generalisation did not happen.

**Every identifier in it is Cursor.** `CURSOR_HOOK_EVENTS`, `CursorHookEvent`,
`CURSOR_EVENT_SCRIPT_MAP` — and the map's six values are `gsd-cursor-session-start.js`,
`gsd-cursor-post-tool.js`, `gsd-cursor-pre-tool.js`, `gsd-cursor-stop.js`,
`gsd-cursor-subagent-start.js`, `gsd-cursor-subagent-stop.js`. There is no non-Cursor string in
the file.

**The descriptor-driven event set is a no-op today.** `resolveManagedHookEvents` reads a
descriptor list so the event set is not hardcoded. `grep -rn "managedHookEvents" capabilities/`
returns exactly one hit: `capabilities/cursor/capability.json:107`. And that list is
`["sessionStart", "postToolUse", "preToolUse", "stop", "subagentStart", "subagentStop"]` — the
six members of the `CURSOR_HOOK_EVENTS` fallback, in order, element for element. The mechanism
is descriptor-driven; the outcome is the hardcoded default for every shipped descriptor.

**When the second `hooks.json` host arrived, it got a copy.** Windsurf declares
`hooksSurface: "windsurf-hooks-json"` and `hookBus: "host"` and does not route through the hook
bus at all. It is wired by a copy-shaped sibling inside `src/runtime-hooks-surface.cts`:
`GSD_WINDSURF_HOOK_MARKER` (`:1445`), `WINDSURF_HOOK_EVENTS` (`:1449`),
`WINDSURF_EVENT_SCRIPT_MAP` (`:1452`, whose own comment says it "mirrors CURSOR_EVENT_SCRIPT_MAP's
convention"), and `buildWindsurfHookEntry`.

!!! note "The reason is good, which is the point"
    `src/runtime-hooks-surface.cts:1420-1442` gives the honest justification: Cascade blocks
    via exit code 2 rather than stdout JSON, its entry shape has a bare `command` with no
    `type` field, and it has no context-injection channel — so only 2 of GSD's 6 Cursor events
    (`pre_write_code`, `pre_run_command`) have a Cascade counterpart, and the 4 advisory events
    are "deliberately NOT ported."

    The abstraction was not lazy. It was extracted from a sample of one, and the second sample
    differed on the axis the abstraction had held constant: the *protocol*, not the *event
    list*. `imperative-hook-bus.cts` parameterises which events to register. Nothing in it
    parameterises how a hook says no.

The concrete cost is visible in `learn/runtime/hooks.md`, which counts the scripts: eight of the
26 hook scripts in `hooks/` are per-harness adapters — six Cursor, two Windsurf — a third of the
directory, none of it shared.

### Nineteen harnesses means sixteen reference-host test files

`ls tests/declarative-reference-*.test.cjs tests/*imperative-reference*.test.cjs` returns 16
files. Mapping them onto the 19 runtime packs: 6 of the 8 declarative packs have one (`codex`
and `kimi-code` do not), and 10 of the 11 imperative packs do (there is no file named
`tests/vscode-imperative-reference.test.cjs`). These are not small: `declarative-reference-antigravity.test.cjs`
is 24.5K, `trae-imperative-reference.test.cjs` 22.1K. Each one classifies the profile,
round-trips a real install, proves negotiation fails closed on a corrupted descriptor, validates
the descriptor, and source-greps the installer for retired `runtime === '<id>'` branches.

That last check is the direction of travel: `hostBehaviors` is an escape hatch that folds former
hardcodes into descriptor data. `src/install-engine.cts:594-598` is the clearest specimen —
`agentFileExtension` "descriptor-driven via hostBehaviors.agentFileExtension (was hardcoded
`runtime === 'copilot'`)", with `capabilities/copilot/capability.json:94` supplying `.agent.md`
and every other descriptor leaving it unset. Likewise `cleanupSkillSidecars` gates the Codex
sidecar cleanup at `bin/install.js:10714`, and `bin/install.js:10684` notes that skills-root
resolution is now descriptor-driven "(no isCodex check)".

The trade-off is that `hostBehaviors` is an open, untyped bag of one-off string keys:
`legacyCommandsGsdUninstall: "global"` and `hyphenNameAgentBody: true`
(`capabilities/claude/capability.json:115-116`), `doneBannerStyle: "kimi-agent-file"`
(`capabilities/kimi/capability.json:98`) against `"kimi-code"` in the neighbouring pack. The
coupling moved from code into data. It did not disappear, and data has no type checker.

### Descriptor-driven dispatch hardened in one place and not its sibling

Since the portability mechanism *is* dynamic dispatch off a descriptor string, the safety of
that dispatch is a portability concern. There are two dispatch sites and only one is guarded.

`src/runtime-artifact-layout.cts:283-302` carries a long comment calling the guard a "security
fix" and explaining the hazard precisely: a hand-edited or malformed descriptor naming an
Object-prototype member (`"constructor"`, `"toString"`, `"hasOwnProperty"`) "resolves to that
member instead of throwing, producing garbage staged content rather than a loud failure."
`_resolveNamedConverter` fixes it with an allowlist check against `VALID_CONVERTER_NAMES`, and
fails closed even if the allowlist module itself cannot load.

`src/install-engine.cts:1273` does not:

```ts
const converterName: string | undefined = skillsKindEntry.converter;
const converter = converterName ? SKILLS_CONVERTER_REGISTRY[converterName] : undefined;
if (!converter) {
  throw new TypeError(`installOpencodeFamilySkills: unknown skills converter ...`);
}
```

`SKILLS_CONVERTER_REGISTRY` (`src/install-engine.cts:156`) is a plain object literal with three
entries and an ordinary `Object.prototype`. A descriptor declaring `"converter": "constructor"`
resolves to the inherited function, passes the truthiness check at `:1274` instead of hitting
the intended `TypeError`, and reaches the call at `:1324`. I verified reachability rather than
assuming it: `skillsKind` resolves its converter *lazily*, inside `stage()`
(`src/runtime-artifact-layout.cts:561`), so the `resolveRuntimeArtifactLayout` call at
`install-engine.cts:1261` does not throw first — and `installOpencodeFamilySkills` never calls
`kind.stage()`; it runs its own convert-and-write loop. The path affects the opencode family
(`opencode`, `kilo`, `kimi-code`) and needs a hand-edited or third-party descriptor. I did not
execute it, so the only claim I will make is the narrow one: **the intended loud failure does
not occur.** Per `runtime-artifact-layout.cts:288-290`, `VALID_CONVERTER_NAMES` is "otherwise
enforced ONLY at lint/build time."

Two dispatch sites, a few hundred lines apart, in files that import the same descriptor field.
One got the fix. That is what a fix looks like when the surface it protects is spread across
nineteen configurations.

### Copy-paste in the descriptors themselves

`capabilities/kimi/capability.json` declares `"localConfigDir": ".kimi-code"` — the same value
as the separate `kimi-code` runtime's own `localConfigDir`. These are different products: Kimi
CLI is Moonshot's (global home `~/.config/agents`); Kimi Code is a distinct product (`~/.kimi-code`,
probed via `config.toml`). The collision is inert today because kimi's `artifactLayout.local` is
`[]` and `hostBehaviors.localInstallDeferred` is true — but two modules derive a project path
from that field: `src/install-scope.cts:210` documents the `local` scope as "joins the registry's
`localConfigDir` onto `cwd`", and `src/agent-install-check.cts:128-133` does
`path.join(projectRoot, localConfigDirName)`. Any future local-scope resolution for `kimi`
points into `kimi-code`'s directory. Two harnesses with similar names, one descriptor field
copied across.

### The shipped SDK is harder to reach than the docs suggest

One more cost that only shows up if you check the packaging. `docs/how-to/author-a-host-plugin.md:8`
tells external authors the contract is `src/host-integration-sdk.cts` →
`gsd-core/bin/lib/host-integration-sdk.cjs`. Two facts about that path:

- `package.json` has **no** `exports` field (`node -e "console.log('exports' in require('./package.json'))"`
  → `false`). `main` is `.opencode/plugins/gsd-core.js`. So an external plugin must deep-require
  the built file by path.
- That built file is **not committed**: `git check-ignore -v` resolves it to `.gitignore:75`.
  It is ADR-457 build-at-publish output compiled from the `.cts` source. Only 8 top-level `.cjs`
  files under `gsd-core/bin/lib/` are tracked; this is not one of them.

Inside the repo, the SDK entry has exactly one consumer: `tests/sdk-smoke.test.cjs:16`, which
the SDK header describes as building a new host plugin "against this surface only — proving an
external author can wire a host without reading gsd-core internals." That test is real and it
is good practice. It is also the entire in-repo evidence that the external path works, for a
surface with no export map.

## Is multi-harness support worth it for a framework you are building alone?

Not as designed here. But the failure is more specific than "too many targets", and the specific
version is the useful one.

Look at where the cost actually landed. It did not land on the engine: one
`installRuntimeArtifacts`, one negotiation module, one closed axis vocabulary, and the
descriptor-driven artifact layout genuinely absorbed nineteen harnesses' worth of file-format
variation into data. Nor did it land on the converters: 29 of them sounds like a lot until you
notice they are mechanical frontmatter transforms with a strict naming convention, one dispatch
table, and a validator allowlist. Those parts scale.

The cost landed on **hooks** — the one dimension where the harnesses differ in *protocol* rather
than in *format*, and, as the section above showed, the one dimension the negotiation schema
never modelled. A skill file is a skill file with different frontmatter; a hook that blocks is
a fundamentally different object depending on whether the host reads your stdout or your exit
code. That is where the parallel implementation appeared, where a third of `hooks/` went, where
the same `.planning/` policy got written three times at three different strengths, and where the
one abstraction built to absorb the variation absorbed the wrong variable.

So the transferable rules, in order of how much they would have saved:

1. **Port the format layer; think hard before porting the protocol layer.** Converting artifacts
   into a new host's file shapes is descriptor work and it compounds cheaply. Adapting to a new
   host's *control* protocol — how it blocks, how it injects context, how it isolates a subagent
   — is bespoke every time and it compounds expensively.

    Scanning every `hooks/gsd-*.js` for the deny emissions each host understands — a
    `permission: "deny"` / `decision: 'block'` payload, a literal `block: true`, or a
    `process.exit(2)` — returns eight scripts, in three harness families: five Claude-family
    guards (`gsd-workflow-guard.js`, `gsd-write-guard.js`, `gsd-worktree-path-guard.js`,
    `gsd-agent-isolation-guard.js`, `gsd-read-injection-scanner.js`), one Cursor
    (`gsd-cursor-subagent-start.js`), and two Windsurf (`gsd-windsurf-pre-write.js`,
    `gsd-windsurf-pre-command.js`). Nineteen harnesses get artifact projection. **Three get
    enforcement**, and the three do not enforce the same rules.
2. **Extract from two, not from one.** `imperative-hook-bus.cts` parameterised the variable
   Cursor happened to expose (which events to register) and hardcoded the one it did not
   (how a hook refuses). Writing the second host concretely and duplicating on purpose, then
   extracting once both shapes are visible, would have cut the seam in the right place.
3. **If a claim is your justification for an abstraction, wire it to a test whose deletion is
   loud.** "Byte-identical, gated by golden-install-parity" was the argument that two adapters
   were one interface. The gate is gone, the claim is repeated in three files, and the arity
   difference between the two adapters is now real and undetected.
4. **Count the tests before you add the target.** 16 reference-host test files, several over
   20K, is the standing tax on this design. Adding harness twenty is not "write a descriptor" —
   it is a descriptor, possibly a converter, possibly a hook dialect, and a 10-to-25K reference
   test that must keep passing forever.

A solo framework can afford portability across a *format* boundary. It probably cannot afford
portability across a *behaviour* boundary. The honest version of this design ships full artifact
projection to all nineteen harnesses, ships enforcement only to the handful whose control
protocol you have actually read the docs for, and *says which is which in the README* — rather
than leaving a reader to discover it by counting the files in `hooks/`.
