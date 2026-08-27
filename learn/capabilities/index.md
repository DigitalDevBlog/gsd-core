# Capabilities

GSD Core ships 46 capability packs. Each one is a directory under `capabilities/` containing
a single `capability.json` and, for nine of them, a `fragments/` subdirectory of markdown.
That is the whole on-disk surface: `find capabilities -type f | wc -l` returns **58** — 46
manifests plus 12 fragment files.

Which sounds like a configuration format, and is not. The interesting property of this
subsystem is that the *same* manifest schema is read on two completely different trust
footings. First-party packs are compiled at build time into a committed registry by a script
that can afford to trust its input. Third-party packs arrive from npm, git, or a local path
at install time, and everything after that point — bounded reads, disclosure, consent,
locking, ledgers, rollback — exists because the framework has decided to execute somebody
else's hook scripts on your machine.

This page is about the shape of a capability and the two lifecycles it moves through. The
transferable question underneath is the one every plugin system eventually answers badly:
how do you let a third party extend your framework without handing them the keys?

## What is actually in `capabilities/`

Tallying `(role, tier)` across all 46 manifests:

| role / tier | count |
|---|---|
| `feature` / `full` | 20 |
| `runtime` / `core` | 19 |
| `reviewer` / `full` | 5 |
| `feature` / `standard` | 2 |

Twelve fragment files spread over **nine** packs — `ai-integration`, `assumption-delta`,
`claude-orchestration` (2), `external-job` (2), `live-dom-uat`, `mempalace` (2),
`pattern-mapper`, `research`, `schema-gate` — counted by
`find capabilities -path '*/fragments/*.md' | cut -d/ -f2 | sort | uniq -c`.

!!! warning "These files never reach a user's machine"
    `capabilities` is absent from `package.json`'s `files[]` array. The packs are build-time
    *inputs*. `scripts/gen-capability-registry.cjs` compiles them into the committed
    `gsd-core/bin/lib/capability-registry.cjs`, and that generated file is what ships. Cite
    `capabilities/<id>/capability.json` when you mean the source of a first-party capability,
    and never cite it as something the runtime reads — for first-party packs, it does not.

A whole feature manifest, `capabilities/schema-gate/capability.json`, fits on one screen:

```json
{
  "id": "schema-gate",
  "role": "feature",
  "version": "1.11.0",
  "title": "Schema push detection gate",
  "description": "Detects ORM schema-relevant files in the phase scope during planning…",
  "tier": "full",
  "requires": [],
  "engines": { "gsd": ">=1.6.0" },
  "runtimeCompat": { "supported": ["*"], "unsupported": [] },
  "skills": [], "agents": [], "hooks": [],
  "config": {
    "workflow.schema_push_detection": { "type": "boolean", "default": true, "description": "…" }
  },
  "steps": [],
  "contributions": [
    { "point": "plan:pre", "into": "planner",
      "fragment": { "path": "fragments/plan-pre.md" },
      "produces": [], "consumes": ["CONTEXT.md"],
      "when": "workflow.schema_push_detection", "onError": "skip" }
  ],
  "gates": []
}
```

Note how much of it is empty. Seven arrays are declared, one carries anything. That is not
laziness; it is the schema doing its job, and it is provable.

## The role is a shape contract, verified by set equality

The obvious way to check a manifest schema is to count fields. Counting is weak evidence —
two sets of size 22 need not be the same 22. So the check worth running compares *sets*:
for each key, collect the ids of packs declaring it, and test that collection against the set
of packs with a given role.

Running that over all 46 manifests:

- The **22 packs with `role: "feature"`** are *exactly* the 22 declaring every one of
  `runtimeCompat`, `skills`, `agents`, `hooks`, `steps`, `contributions`, `gates`.
  Set-identical, all seven times.
- The **19 packs with `role: "runtime"`** are *exactly* the 19 declaring a `runtime` block.

There is no near-miss, no pack that declares six of the seven. The role is not a label
attached to a bag of optional fields — it selects a fixed shape.

The optional keys sit outside that contract: `config` on 31 packs, `commands` on 6,
`activationKey` on exactly 5 (`claude-orchestration`, `graphify`, `intel`, `live-dom-uat`,
`refactor-trigger`).

And one field is on all 46 for a reason that is easy to misread. `requires` appears
universally because `validateCapability` *mandates* it — `gsd-core/bin/lib/capability-validator.cjs:371`
pushes `'requires must be an array of capability ids'` when it is not an array. It is a
required field that is universally empty, with exactly **one** non-empty edge in the entire
first-party set: `pattern-mapper` → `research`. The dependency graph is not sparse because
nobody bothered to declare edges. It is sparse because there is one.

### Role dispatch is enforced in source, not by convention

`validateCapability` branches on `cap.role` at `gsd-core/bin/lib/capability-validator.cjs:385-414`:

- `feature` → `validateFeatureBody`, plus an explicit error if a `reviewer` body is present.
- `runtime` → `validateRuntimeBody` **and** `validateReviewerBody`.
- `reviewer` → requires a `reviewer` body, forbids `runtime`, and rejects every member of
  `FEATURE_FIELDS_FORBIDDEN_ON_REVIEWER`.

That middle branch explains the one asymmetry in the shipped set. A `reviewer` block appears
on **12** packs but only **5** have `role: "reviewer"`. The other seven — `antigravity`,
`claude`, `codex`, `cursor`, `kimi-code`, `opencode`, `qwen` — are host runtimes that double
as review lanes. The comment at `capability-validator.cjs:397` states the rule directly: *"A
host that is ALSO a reviewer keeps exactly one manifest (ADR-2782 D1)."*

Worth noticing what that choice avoids. The alternative — a separate `claude-reviewer` pack
alongside `claude` — would have produced a second manifest whose identity, version envelope,
and consent binding all have to be kept in lockstep with the first by hand. One manifest with
a conditionally-admitted body pushes that consistency into the validator, where it is
checkable.

Rejecting a `reviewer` body on a feature pack as an **error** rather than ignoring the field
is the other half of the same posture. The comment gives the reasoning: declaring a body is
*"an ASSERTION of lane-ness"*. Silently dropping an unrecognised field is the usual liberal
move, and it is exactly how a manifest that believes it configured an egress destination ends
up configuring nothing.

## Two lifecycles, not one

The stages are usually named *load → activate → consent → trust → lock*. That is a useful
topic ordering and a misleading execution order. In the code there are two distinct paths,
and they do not agree on the sequence:

- **Install-time** is a mutating path: stage and validate, then **trust** (disclose the
  executable surfaces), then **consent** (prompt and record), then commit. The **lock** wraps
  the mutation, not the tail. Trust *precedes* consent, because the disclosure is the thing
  the user is consenting to.
- **Load-time** is read-only and lock-free. Consent is *checked*, never recorded. And
  **activation** is not in the loader at all — it is a separate later resolution in
  `src/capability-state.cts` and `src/capability-activation.cts`.

```kroki-plantuml
@startuml
skinparam defaultFontSize 12
skinparam sequenceMessageAlign left

box "INSTALL — mutating, locked" #FDF6E3
participant "capability-\nsource.cts" as SRC
participant "capability-\ntrust.cts" as T
participant "capability-\nlifecycle.cts" as L
participant "capability-\nconsent.cts" as C
end box

box "LOAD — read-only, never throws" #EEF6FB
participant "capability-\nloader.cts" as LD
participant "capability-\nstate.cts" as ST
end box

== install: stage, validate ==
SRC -> SRC : parseSpec + stage\n4 DoS bounds (:153-194)
SRC -> SRC : stageValidated (:748-882)
note right of SRC
capMap has ONE entry,
centralKeys is EMPTY —
2 of 3 cross-cap invariants
cannot fire (:836-841)
end note

== install: trust, THEN consent ==
SRC -> T : discloseExecutableSurfaces
T -> T : summarizeDisclosure\n→ raw stderr
T --> L : the ROUTER prompts and the\noperator consents (router :303)

== install: mutate under lock ==
L -> L : ledger _pending BEFORE\nany fs mutation (:1129)
L -> L : promoteStagingToFinal\nhooks confined (:494-534)\nMCP verbatim (:706-735)
L -> C : recordProjectConsent (:956)
C -> C : throw rather than\nwrite unlocked (:875-884)
L -> L : clear _pending\n← THIS is the commit
note right of L
A crash above leaves _pending set.
reconcile rolls forward, or restores
via the DUR-6 move-aside (:1649).
end note

== load: a different order ==
LD -> LD : overlayRoots (:318-391)\nskip _pending (:596)\nbounded read (:608)
LD -> LD : validate + contract\n+ consumes + cross-cap
LD -> C : hasProjectConsent(\nbundleContentHash) (:733)
C --> LD : match / no match
note right of LD
no match → kind:'unconsented',
CONTINUE — before fragments load
end note
LD -> LD : buildRegistry over\nfirst-party ∪ accepted
LD -> LD : any throw → frozen\nfirst-party registry (:876)

== activation: a third module ==
LD -> ST : registry
ST -> ST : active = installed && surfaced\n&& configActivation
@enduml
```

### Install: intent before mutation

The commit signal is not a version comparison. `capability-lifecycle.cts:1108-1136` writes a
ledger entry carrying `_pending` **before** touching the filesystem, and the commit is the
*clearing* of that field. The stated reason (`capability-lifecycle.cts:139-141`) is that a
same-version malicious bundle must not be mistakable for a committed one — a version check
would call a half-written reinstall "already at 1.11.0" and stop.

The rollback is written to never have zero copies on disk. The DUR-6 block
(`src/capability-lifecycle.cts:1649-1661`) sits in `reconcileCapabilities`' rollback branch,
not in `promoteStagingToFinal` (which is `src/capability-lifecycle.cts:547-577`). It moves
the uncommitted directory aside, renames the backup back, then drops the aside copy, and
its comment states the rule directly: never `rmSync` the final directory before restoring. At every instant at
least one intact copy exists.

Consent writing is deliberately *not* fatal. `bindProjectConsent`
(`capability-lifecycle.cts:941-977`) catches a consent-store write failure and warns rather
than failing the install, because the bundle is already committed — the capability simply
stays inactive until consent can be recorded. That is the right split: the mutation that
already happened is not rolled back to punish a read-only `$HOME`.

### Consent binds content, not provenance

`bundleContentHash` (`capability-consent.cts:487-588`) is a length-framed, injective, raw-byte
serialization: a uint64 entry count, then per entry a 1-byte type tag, a uint32 path length
plus raw path bytes from an `{encoding:'buffer'}` walk, and for files a uint64 content length
plus raw bytes.

Every element of that is load-bearing. Length framing kills delimiter ambiguity. Raw bytes
kill the U+FFFD collision on invalid-UTF-8 filenames *and* contents. Typed directory markers
bind empty directories, which a file-only walk would let an attacker create or remove for
free.

The module header (`capability-consent.cts:12-23`) is unusually direct about what the binding
is **not**:

> …bound to a RECOMPUTED full-bundle content hash (`bundleContentHash`), NOT to the ledger
> `integrity` (which is `''` for path/git/dir installs and taken verbatim from the
> repo-plantable project ledger — `'' === ''` is no binding) NOR to the `disclosureSignature`
> alone (which covers only executable surfaces, so a declarative-only cap has a constant
> signature and a repo-write attacker could swap `capability.json` for a malicious
> gate/contribution while consent still matched).

The implementation matches: `hasProjectConsent` (`capability-consent.cts:669-683`) compares
`rec.contentHash === contentHash` and nothing else. `integrity` and `disclosureSignature` stay
on the record purely for the human disclosure UX.

Two refusals in that hash are argued in source rather than waved off. It deliberately does
*not* exclude `.DS_Store`, `node_modules`, `dist` or `build`
(`capability-consent.cts:329-332`, `:496-498`): excluding an executed directory would stop
consent from binding executable content, converting a usability complaint into a supply-chain
hole. The two `__pycache__` carve-outs added by #3631 are narrow, and the residual risk they
create is written down at `capability-consent.cts:500-520`.

### Load: never crash is the top-level contract

The loader's job is to compose a registry from first-party packs plus whatever overlays are
present, and its hardest requirement is that a hostile overlay must not be able to stop the
framework from starting. That is implemented at three nesting depths:

1. **Per candidate** — the entire body is wrapped (`src/capability-loader.cts:645-855`) so any
   throw from any validator becomes a structured skip.
2. **Per validator** — an extra guard around the cross-capability trio (`:780-790`).
3. **Whole compose** — a `buildRegistry` throw falls back to the frozen first-party registry,
   clears `commandRoots`, and re-records dropped gates (`:876-901`).

Supporting that, `gatePointsOf` (`:482-493`) is deliberately *total* over `null`, non-object
and non-array `gates`, and over malformed entries. A helper that can throw would defeat the
fallback path that calls it.

Scope escalation is the one place the loader is stricter than it needs to be, and the comment
says why. `overlayRoots` (`:318-391`) demotes the trusted-global slot to consent-required
`project` unless `realpath(global)` **and** `realpath(project)` both succeed **and** differ —
and only when a genuine project marker (`.planning/` or `.git`) exists. The earlier one-sided
rule left a symlinked-`GSD_HOME` bypass open. It is a rule that got stricter after the weaker
version was shown insufficient, which is worth more than a rule that was right first time.

Ordering matters in one non-obvious place. The unconsented `continue` at `:726-748` happens
**before** `materializeHookFragments`, and the comment says so: a forged FIFO or oversized
fragment in an unconsented overlay is never read, so it cannot hang or OOM the loop.

!!! note "One builder, two entry points"
    `scripts/gen-capability-registry.cjs` compiles the 46 first-party packs into
    `gsd-core/bin/lib/capability-registry.cjs`. Requiring that module gives an object with
    twelve top-level keys: `capabilities` (46 entries), `version`, and ten derived views —
    `bySkill`, `byAgent`, `byLoopPoint`, `configKeys`, `configSchema`, `runtimes`,
    `commandFamilies`, `capabilityClusters`, `profileMembership`, `requiresClosure`.
    At runtime `capability-loader.cts:877` calls the **same** `buildRegistry` over
    `first-party ∪ accepted overlays`. No derived view can drift between the build path and
    the load path, because there is only one implementation of each.

### Activation is a third dimension, orthogonal by construction

`CapabilityStateEntry` (`src/capability-state.cts:75-107`) composes install profile × runtime
surface × config activation, and the composition is written to keep the old two-dimension
answer intact:

```
enabled = installed && surfaced          // unchanged
active  = enabled && configActivation    // configActivation defaults true when no activationKey
```

`_resolveActivationValue` walks four precedence levels
(`src/capability-activation.cts:78-115`): the `loadConfig` result, then the raw workspace
`config.json`, then the raw root `config.json` when it differs, then the registry
`configSchema` default — absent at all four means not found.

The containment rule is the good part. The validator forces `activationKey` to name a key in
the capability's **own** config slice (`capability-validator.cjs:695-718`), so a third-party
pack's activation gate can never point at somebody else's key. Without that, an overlay could
make its own activation depend on a first-party key it does not own, and toggling an unrelated
setting would silently light up third-party code.

## The committed-`.cjs` constraint and its three consequences

`gsd-core/bin/lib/` is mostly gitignored build output compiled from `src/*.cts` at publish
time (ADR-457). `git ls-files gsd-core/bin/lib/` returns 11 tracked entries: eight `.cjs` in
the directory itself, plus a vendored `re2js.cjs` / `re2js.d.cts` pair and a README under
`vendor/`. `capability-validator.cjs` is one of the eight, and its header explains why
(`capability-validator.cjs:4-14`):

> Extracted from `scripts/gen-capability-registry.cjs` per ADR-1244 D2 so that both the
> build-time generator and the runtime overlay loader can require the validator WITHOUT
> pulling in the generator's build-time-only machinery… This is a COMMITTED plain `.cjs` (not
> built from `.cts`) so it is available on a fresh worktree before `npm run build:lib` has run.

There is no `src/capability-validator.cts`. The reasoning is sound — a validator that only
exists after a build cannot validate anything on a fresh clone. But a `.cjs` that `src/*.cts`
must consume across a build boundary has costs, and this repo pays three distinct ones.

**One: a duplicated grammar, held by a test.** `LANE_SLUG_RE`
(`capability-validator.cjs:864`) carries a `⚠ DEFECT.GENERATIVE-FIX` note above it at
`:851-858` explaining that it *"cannot be imported"* from `src/review-lane-descriptor.cts`,
because that module compiles to gitignored build output while this file must load on a fresh
worktree. The note names its own compensating control: the parity assertion
`laneSlugGrammarMatchesPhase1Descriptor` in `tests/reviewer-manifest-body.test.cjs`.

**Two: a lazy, fail-closed dependency stub.** The bounded reader the validator needs lives in
`capability-ledger.cjs`, which *is* build output. So it is required lazily
(`capability-validator.cjs:2565-2573`), and if unavailable `readSmallRegularFile` becomes
`() => null` — turning every path fragment into a read error rather than silently reverting to
an unbounded `readFileSync`. The failure mode was chosen deliberately.

**Three: a hand-mirrored module type that dropped a security parameter.** This one shipped.

### DEFECT — the `centralPatterns` check cannot fire at runtime

`validateCrossCapability` is declared with three parameters
(`capability-validator.cjs:3063`):

```js
function validateCrossCapability(capMap, centralKeys, centralPatterns = []) {
```

Its third parameter drives a check at `capability-validator.cjs:3146-3154` whose own comment
(`:3138-3145`) states precisely what it exists to prevent:

> …`isCentralConfigKey` DOES consult the patterns, and `mergeFederatedConfig` skips every key
> for which it returns true. So a federated slice overlapping a central pattern is INERT — it
> carries no traffic — while the build stays green.

Grepping every non-test call site: only `scripts/gen-capability-registry.cjs:480` passes the
third argument. Both **runtime** callers pass two:

- `src/capability-loader.cts:784` — `validator.validateCrossCapability(acceptedMap, getCentralKeys())`
- `src/capability-source.cts:841` — `capValidator.validateCrossCapability(capMap, centralKeys)`

And this is not a forgotten argument at the call site. Both `.cts` files hand-mirror the
validator's module type, because the real `.cjs` has no types to import — and both mirrors
declare only two parameters:

```ts
// src/capability-loader.cts:62 — identical text at src/capability-source.cts:73
validateCrossCapability: (capMap: Map<string, unknown>, centralKeys: Set<string>) => string[];
```

So `centralPatterns` defaults to `[]`, `centralPatterns.find(...)` iterates nothing, and the
check is inert on both runtime paths. TypeScript would reject a fix at the call site until
the mirror is widened.

It is worth ruling out the charitable reading, which is that `getCentralKeys()` already folds
pattern-matched keys into the `Set` it passes as the second argument. It does not.
`loadCentralConfigKeys` (`scripts/gen-capability-registry.cjs:106-130`) ends with
`return new Set(Array.isArray(manifest.validKeys) ? manifest.validKeys : [])` — exact keys
only. The patterns live in a *separate* exported function, `loadCentralConfigPatterns`
(`:163`), whose own doc comment states the gap in the plainest possible terms:

> `loadCentralConfigKeys` above reads `validKeys` only, so a federated key claimed by a
> central *pattern* was invisible to the exclusivity check.

And the loader cannot reach that function either, because the *second* hand-written mirror
closes the gap from the other end. `GeneratorModule` at `src/capability-loader.cts:80-83`
declares exactly two members:

```ts
interface GeneratorModule {
  buildRegistry: (capMap: Map<string, unknown>) => Registry;
  loadCentralConfigKeys: () => Set<string>;
}
```

`loadCentralConfigPatterns` is exported by the generator at `:918` and has no caller outside
the generator itself. So two independently hand-maintained module interfaces — one dropping a
parameter, one dropping a member — combine to make the check unreachable from the runtime,
and neither omission is visible to the compiler or to any test.

I verified the consequence end to end rather than trusting the comment:

- `src/config-schema.cts:63` — `isCentralConfigKey` returns true via
  `DYNAMIC_KEY_PATTERNS.some((p) => p.test(keyPath))`.
- `src/federated-config.cts:14` — `mergeFederatedConfig` skips any key where `isCentralKey(key)`.

A third-party overlay declaring a config key matched by a central dynamic pattern — the
`review.models.<slug>` family, for instance — passes the loader's validation, is composed into
the registry, and its federated slice then carries no traffic. Silently inert, exactly as the
comment predicts, on exactly the path the comment was written for.

The parity harness that ought to catch this cannot. `tests/capability-registry.test.cjs:5648`
checks that the validator module *exports* each name (`assert.ok(sym in capValidatorModule)`)
and that the generator re-exports the same object references. It is a name-and-identity check.
Nothing anywhere asserts that the hand-written `.cts` interfaces match the real function
arities, so a mirror can lose a parameter without a single test going red.

!!! note "If you are building this yourself"
    The moment you have a module that cannot be typed by its own source, you have accepted a
    hand-maintained interface, and a hand-maintained interface is a place where a security
    parameter can go missing without a compile error or a test failure. If you must have a
    pre-build-available validator, generate its `.d.cts` from the `.cjs` and check the
    generated file in — or assert arity in the parity test. Either one turns this defect into
    a red build.

### DEFECT — the install-time cross-capability pass is doubly vacuous

`src/capability-source.cts:836-841` builds a one-entry map and an empty key set, then runs the
full cross-capability suite over them:

```ts
const capMap = new Map<string, unknown>([[id, cap]]);
const centralKeys = new Set<string>();
```

With one entry, no owner-uniqueness collision against first-party can fire. With an empty
`centralKeys`, the central-schema exclusivity check at `capability-validator.cjs:3127` is a
no-op. Both are defensible — the loader re-runs the suite over the merged set, so the
invariants are enforced *somewhere*. The defect is that the comment at
`capability-source.cts:835` says only *"Cross-capability validations (contract, consumes,
cross-capability)"* and never notes that two of the three invariants cannot trigger there. The
next person to read it will believe install-time validation is stronger than it is.

## A cluster of stale comments around the trust model

Three separate places in this subsystem document a design that the shipped code abandoned.
They are worth listing together because the pattern is the same each time: the *prose* was
updated at one altitude and missed at another.

**Gate policy — fail-closed JSDoc over fail-open code.** `src/capability-loader.cts:158` reads
*"Skipped capabilities that declared a gate — the loop must fail CLOSED for these."* Lines
`:160-165` say the loop resolver *"must inject a blocking gate at that point rather than
proceeding as if the gate had passed."* The actual consumer,
`src/loop-resolver.cts:543-591`, injects no gate. It emits a stderr warning:

```
its gate at ${point} is SKIPPED and NOT enforced (failing open)
```

Two precision points. The `blockedGates` field is **not** dead — `loop-resolver.cts` says
outright that *"blockedGates is still recorded by the loader; only the consequence changes
from block to warn."* The field is load-bearing for the warning. What is stale is the field's
JSDoc, which asserts an enforcement policy its only consumer abandoned. And this is localized
staleness, not systemic rot: the loader's own module header at `:18-25` is already correct
post-#2009, as are `CONTEXT.md:323`, `docs/INVENTORY.md:461` and
`docs/explanation/capability-overlay-model.md:205-215`. Only the JSDoc on the `OverlayMeta`
fields and one inline comment were missed when the policy flipped.

The #2009 rationale for dropping the injection rather than making it non-blocking is
unusually candid (`loop-resolver.cts:559-564`): no host workflow generically surfaces an
arbitrary gate's message, and generic gate consumers expect an object-shaped `check`, not a
prose string — so an injected advisory gate would be silently dropped or mis-dispatched. The
fix chosen was the channel that is actually surfaced.

**`signatureForManifest` — a shared binding that is not shared.**
`src/capability-trust.cts:1362-1367` claims:

> Both the loader (which checks whether a previously-consented project cap still matches) and
> the lifecycle (which records the consent) compute the binding through THIS helper so they
> can never drift.

The loader does not consume it and has not since #1459. Grepping every `require(` in
`src/capability-loader.cts`, it pulls `project-root`, `capability-registry`,
`capability-validator`, `semver-compare`, `capability-ledger`, `capability-consent`,
`external-descriptor-trust` and lazily the generator — never `capability-trust`, and not
transitively either (`grep -n "capability-trust" scripts/gen-capability-registry.cjs` is
empty). Its actual
binding is `consentMod.bundleContentHash(capDir)` at `src/capability-loader.cts:733`. The only
production caller of `signatureForManifest` is `src/capability-lifecycle.cts:956`.

The same stale claim is repeated at `capability-trust.cts:187-191`, `capability-trust.cts:930`,
`CONTEXT.md:350`, and `docs/adr/2782-reviewer-lane-capability-surface.md:343` — and
`CONTEXT.md` contradicts itself, because `CONTEXT.md:344` describes the `contentHash` binding
correctly.

**The trust-model doc describes the pre-CB-1/CB-2 model.**
`docs/explanation/capability-trust-model.md` states the consent binding as the two things
`capability-consent.cts:12-23` explicitly says it is *not*. Doc lines 411-413 say consent
*"binds the bundle's `integrity` and its disclosure signature"*; lines 425-427 say *"because
the consent binds the integrity and the disclosure signature, tampering with the bundle …
deactivates it"*; lines 206-210 repeat it. The conclusion happens to be right and the
mechanism is wrong, which is the worst combination — a reader who trusts it will reason about
`integrity` on a path where `integrity` is `''`.

That document has two further inaccuracies worth naming, because they are the ones a builder
would copy:

- **It contradicts itself on integrity pinning.** Line 66-67 asserts *"GSD requires an
  `integrity` SHA-512 pin in the ledger, verified before extraction"*, while lines 291-300 of
  the *same* document accurately describe the unpinned case (the `content: NO PINNED HASH —
  staged unverified` prompt line added by #3514). The source resolves it against line 66:
  `src/capability-source.cts:966` throws *"integrity pinning is not supported for local
  sources"* and `:999` throws *"integrity pinning is not supported for git sources; pin the
  commit with `#sha:<commit>`"*. `verifyIntegrity` is called only at
  `capability-source.cts:1078` (npm) and `:1126` (tarball) — two of five spec kinds. The
  document's accurate half was never propagated to its summary.
- **It describes a richer consent prompt than ships.** Lines 197-199 claim the pre-install
  summary names each executable surface *"their kinds (`step`, `contribution`, `gate`), and
  the loop extension points they register into."* The `Disclosure` object carries no such
  data. `HookSurface` (`src/capability-trust.cts:115-118`) is `{ event, script }`, and
  `summarizeDisclosure` (`capability-trust.cts:1533-1539`) renders `    - ${event} -> ${script}`.
  A capability's `steps` / `gates` / `contributions` arrays are never collected into a
  disclosure at all. I checked for a second renderer that might supply the difference:
  `gsd-core/bin/lib/capability-command-router.cjs` calls `trust.summarizeDisclosure` at lines
  303, 309 and 392 and adds no supplementary section.

**And one comment in the wrong place.** `gsd-core/bin/gsd-tools.cjs:512` documents the
third-party dispatch gate as *"CONSENT: `loadRegistry({ includeInstalled })` excludes
`_pending` (unconsented) capabilities"*. Those are two distinct gates with opposite semantics:
the `_pending` skip (`capability-loader.cts:596-599`) means *install did not commit*, while
the consent gate (`:726-748`) emits `kind: 'unconsented'`. Conflating them in a comment on the
code path where third-party code actually executes is the wrong place to be imprecise.

## Smaller defects, and one with a reachability chain

**An unbounded read on a reconcile path.** `readManifest`
(`src/capability-lifecycle.cts:326-335`) uses a raw
`fs.readFileSync(path.join(dir, 'capability.json'), 'utf8')`. The chain into
project-controlled bytes: `capability-command-router.cjs:74` resolves `--scope project` to
`runtimeDir: cwd`; `capRunReconcile` is invoked from router lines 284, 354 and 447 on
install/update/remove; it calls `reconcileCapabilities`, whose rollback branch calls
`resyncCapabilitySharedEdits` (`capability-lifecycle.cts:604-617`) →
`readManifest(capDir(runtimeDir, capId))` on
`<projectRoot>/.gsd/capabilities/<id>/capability.json`. The per-scope ledger read that gates
this *is* bounded; the manifest read is not, so a planted FIFO at that path blocks the
reconcile. This is exactly the file the loader hardened at `src/capability-loader.cts:608`
with `readSmallRegularFile(manifestPath, MANIFEST_MAX_BYTES)`, for exactly this reason
(#1459 finding 2). Stated honestly: reaching it requires the user to run a project-scoped
capability command, and the planted ledger must survive `isValidLedgerEntry` — so this is
plausible-and-reachable, not demonstrated.

**A comment that claims an enforcement the module does not perform.**
`src/capability-lifecycle.cts:53` declares `readSmallRegularFile` on its `ledgerMod` type with
a load-bearing comment — *"the shared fd-based bounded reader … THROWS for a non-regular
(FIFO/device/dir) / oversized / IO-error file (fail closed)"*. `rg readSmallRegularFile
src/capability-lifecycle.cts` returns only that declaration. The module never calls it. Read
alongside the raw `readManifest` above, the comment actively misleads.

**Type-mirror drift with no runtime consequence.** `src/capability-lifecycle.cts:101-120`
keeps a local mirror of the `Disclosure` interface listing only `hooks`, `commandModules`,
`mcpServers`, `hasExecutable`, `missingArtifacts`. It is missing `reviewerLanes` (ADR-2782)
and `instructionSurfaces` (ADR-2363). This is explicitly **not** a functional defect:
`discloseExecutableSurfaces` returns the full five-class object regardless of how the consumer
types it, and `executableSetChanged` computes the signature inside `capability-trust`. It is
the same failure mode as the `centralPatterns` mirror, caught before it cost anything.

## The parts worth stealing

Several decisions here are better than the defects above suggest, and they generalise.

**Signature stability is treated as a security property.** The highest-consequence line in
`disclosureSignature` (`src/capability-trust.cts:1350`) appends the reviewer-lane element
*only* when a lane is declared, so a lane-free manifest's signature stays byte-identical to
its pre-ADR-2782 encoding. The same discipline excludes instruction surfaces, `resolvedHost`,
`integrityStatus`, and the cosmetic `reviewsSection` / `timeoutFloorMs`. The stated reason
each time: a re-consent prompt that carries no new security information trains click-through.
That is the correct model of the human in the loop, and it is rare to see it written into an
encoding function.

**Closed enums get residual backstops rather than more names.** Where a whitelist proved
unsafe, the fix was a residual: `McpServerSurface.rawConfig`
(`capability-trust.cts:173-181`) and the lane's `residualInvoke` / `residualLane`
(`capability-trust.cts:253-276`). The comment at `capability-trust.cts:1310-1320` counts the
failure precisely — `resolveLanePlan` reads thirteen distinct `inv.*` fields and the
pre-#2483 tuple bound four. Correspondingly, `VALID_LANE_HANDLERS`
(`capability-validator.cjs:879-902`) carries an explicit admission rule (two or more lanes
sharing the behaviour, or a documented upstream defect data cannot express) on the grounds
that an enum growing one member per lane *"is a plugin system wearing a manifest"*.

**Non-determinism is structurally forbidden from blocking.**
`gates[].check.agentVerdict` forces `blocking: false` (`capability-validator.cjs:2803-2807`)
— *"non-deterministic checks may not halt the loop."* An LLM verdict can advise; it cannot
stop you. In a framework whose whole premise is model-driven workflow, drawing that line in
the validator rather than in guidance is the single most defensible decision on this page.

**Error text is made order-independent on purpose.** Reviewer-lane collision errors are
accumulated and sorted rather than reported pairwise
(`capability-validator.cjs:3178-3184`). The reason is one I would not have anticipated: with
three lanes colliding on one key, pairwise reporting names whichever pair arrived first, so
the message text depends on Map insertion order — `readdir` order at build time, candidate
order at load time — and a cross-platform CI lane would then disagree with a local run about
the text of the same failure.

**Known limits are recorded where someone would break them.** `stableJson`
(`capability-trust.cts:1221-1233`) documents a signature-collision limit rather than hiding
it: `NaN`, `Infinity` and `undefined` all render as `null`. The comment notes that
reachability rests entirely on the ingest path staying `JSON.parse`-only, and says it is
recorded *"here rather than only in the PR that found it"* so anyone adding a non-JSON
manifest loader sees it at the point they would break it.

**Availability is traded for DoS resistance, wholesale — and you should notice the price.**
`readConsentStore` refuses the *entire* store when `keys.length > MAX_RECORDS`
(`capability-consent.cts:645`, 4096), returning `{records:{}}`. A single oversized store
therefore silently deactivates every project-scope capability on the machine at once. The
same all-or-nothing posture appears in `readLedgerRaw` (`capability-ledger.cts:272`). This is
a coherent choice for a security boundary — partial parsing of a hostile store is worse — but
it is a choice, and the failure it produces is confusing rather than loud.

**One asymmetry is disclosed instead of closed.** Hook scripts are resolved through
`confinedBundleScript` (`capability-lifecycle.cts:494-534`) to an absolute realpath-confined
path and POSIX single-quoted. MCP `command` / `args` / `env` / `cwd` are written **verbatim**
(`capability-lifecycle.cts:706-735`), because confining them would break every global or
`npx` server. The compensating control is an explicit *"intentionally NOT confined to the
bundle"* prompt line rendered only for **spawned** servers
(`capability-trust.cts:1563-1568`), using one shared `isRemoteMcpServer` predicate so the
notice and the per-server rendering cannot drift into an inexact claim. Naming a hole you
cannot close, in the prompt where the decision is made, beats closing it badly.

**And one surface is deliberately left undisclosed, with the reason stated.** `agents` are
formally an instruction surface under ADR-2363 but are not disclosed:
`INSTRUCTION_SURFACE_FIELDS` (`capability-trust.cts:861-863`) is a one-row table because
`stageAgentsForRuntimeWithConverter` has no registry-aware third-party path, so *"disclosing
them would name a surface that does not exist."* Disclosing a capability the framework does
not actually grant would be worse than disclosing nothing.

## Two things I would do differently

**Make the prompt-injection defence a first-class concern earlier.** `UNSAFE_PROMPT_CHARS`
(`capability-trust.cts:1410`) escapes C0/C1/DEL, bidi overrides and isolates, and
U+2028/2029 — because summary lines are joined with `\n` and written raw to stderr, so an
unescaped manifest value could forge a disclosure line or clear output already printed. The
threat is *the consent prompt itself*, not the shell. It is modelled in at least one other place — `src/loop-resolver.cts:468-474` strips C0 control characters, DEL and backticks from a third-party-supplied failure reason before it reaches stderr, with the call site (`:570-573`) naming the same threat. Two independent sites, no shared helper, which is itself the point in the repo
I found it modelled. Any framework that renders untrusted metadata into a security decision
has this problem on day one.

Related, and instructive: the environment-variable denylist exists in two deliberately
separate copies with *identical* membership — 20 names each, and the set difference is
empty in both directions. `DENIED_LANE_ENV_KEYS`
(`capability-validator.cjs:930-937`) refuses the manifest; `EXECUTION_PRIMITIVE_ENV`
(`capability-trust.cts:1389-1394`) only warns in the prompt. The reason is stated at
`capability-validator.cjs:917-919`: disclosure runs *before* validation, on manifests
validation would reject — so a hand-placed capability still gets a loud line. The validator
set is matched case-insensitively because Windows env lookup is. The duplication is not
justified by divergent membership — the lists agree exactly. It is justified by *layering*,
stated at `src/capability-trust.cts:1667-1677`: one copy refuses, the other warns, and the
warning must survive on manifests the refusing layer would have rejected. That is a
genuine reason to duplicate a constant, and a different one from the usual "these lists
drifted apart" story. It is also fragile in the ordinary way: nothing tests that the two
lists stay in sync.

**Do not require a first-party module to reach into the build directory.** The validator was
extracted from the generator specifically so the loader could validate *without* the
generator's build-time machinery. But `src/capability-loader.cts:562` then requires
`../../../scripts/gen-capability-registry.cjs` for `buildRegistry` and
`loadCentralConfigKeys` — and that module pulls in `ExitError`,
`config-schema.manifest.json`, `install-profiles.cjs`, `clusters.cjs` and
`gen-loop-host-contract.cjs`, the exact list the extraction's header names as what it was
avoiding.

Two mitigations make it work rather than break. The require is **lazy** (`getGenerator`,
`capability-loader.cts:558-565`, reached only when at least one overlay candidate exists), so
the no-overlay fast path really is isolated. And everything resolves in a published install
because `files[]` includes `"scripts"` with only eight named exclusions, none of them the
generator. It functions. But "the extraction achieved isolation" and "the loader requires the
generator" cannot both be the whole truth, and the header still tells the first story.

---

The through-line, if you are building something like this: the manifest schema is the easy
part, and this repo got it right — a role that selects a fixed shape, verifiable by set
equality, with required-but-empty fields that make the shape explicit. The hard part is
everything the schema cannot express. Consent must bind content rather than provenance,
because provenance is empty on three of five install kinds. The loader must be total, because
a hostile pack that prevents startup is a denial of service on the framework itself.
Non-deterministic checks must be structurally barred from blocking. And every module type you
write by hand — because a build boundary stopped you importing the real one — is a place
where a parameter can quietly go missing, which is precisely how the one cross-capability
security check that this subsystem's own comments call load-bearing came to never run.
