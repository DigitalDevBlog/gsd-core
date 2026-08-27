# Diagram authoring reference

!!! info "Tooling page"
    This page is not part of the explanation — it is a reference for authors of the
    content phases. It is deliberately excluded from site navigation.

Diagrams are rendered by [Kroki](https://kroki.io) via `mkdocs-kroki-plugin`, pinned to
`0.9.0`. Author a diagram by opening a fence named `kroki-<engine>`.

## PlantUML

Use for structural and sequence diagrams — anything with boxes, arrows and layering.

````text
```kroki-plantuml
@startuml
component "skills/*" as skills
component "agents/*" as agents
component "src/*.cts" as runtime
skills --> agents
skills --> runtime
@enduml
```
````

Renders as:

```kroki-plantuml
@startuml
skinparam componentStyle rectangle
component "skills/*\n(prompt layer)" as skills
component "agents/*\n(subagents)" as agents
component "src/*.cts\n(runtime)" as runtime
skills --> agents : spawns
skills --> runtime : calls
@enduml
```

## Excalidraw

Use for freeform, hand-drawn explanatory sketches. Author the `.excalidraw` JSON, then
embed its contents in a `kroki-excalidraw` fence:

````text
```kroki-excalidraw
{ "type": "excalidraw", "elements": [ ... ] }
```
````

## Self-hosting the renderer

Builds POST diagram source to the public `kroki.io` by default, so a strict build needs
network access. Point at a local instance instead:

```bash
docker run -d -p 8000:8000 yuzutech/kroki
export KROKI_SERVER_URL=http://localhost:8000
mkdocs build --strict
```

The `ServerURL` setting in `mkdocs.yml` reads `KROKI_SERVER_URL` and falls back to the
public server, so no config change is needed to switch.

## Publishing versions

Docs are versioned with [mike](https://github.com/jimporter/mike), so each published
version states which GSD Core release it describes.

```bash
# build and record a version (writes to the gh-pages branch, locally)
mike deploy --update-aliases 1.11 latest
mike set-default latest

# preview the version switcher locally
mike serve

# publishing is a separate, explicit step
mike deploy --push --update-aliases 1.11 latest
```

!!! warning "Not yet deployed"
    No version has been deployed. `mike deploy` writes a `gh-pages` branch into this
    repository — which is also the published `@opengsd/gsd-core` package repo — and
    `--push` publishes it. Both are left to the repo owner.
