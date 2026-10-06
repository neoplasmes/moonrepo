# Workspace setup

Everything that lives in `.moon/` or at the repository root: project discovery, toolchains, version pinning, boundaries, shared config and the optional machinery (daemon, MCP, hooks, Pkl).

## Minimal layout

```text
.moon/
    workspace.yml      # projects, vcs, pipeline, remote cache, docker, generator
    toolchains.yml     # language toolchains (v2: WASM plugins)
    extensions.yml     # optional: moon ext plugins
    tasks/
        all.yml        # inherited by every project
        <name>.yml     # inherited by projects selected by `inheritedBy`
.prototools            # pinned versions of moon, proto and every CLI tasks call
moon.yml               # optional root project
```

`.gitignore` and `.dockerignore` must contain `.moon/cache` and `.moon/docker`. Config files may be YAML (`.yml`/`.yaml`), JSON, TOML, HCL or Pkl; keep one format per repository.

## Projects

```yaml
# .moon/workspace.yml
projects:
    - apps/*
    - packages/*
    - services/*/pyproject.toml   # v2.5+: a matched file makes its folder a project
    - crates/*/Cargo.toml
```

- Globs are cheap to maintain; a map (`api: services/api`) gives stable ids that do not depend on folder names. Both can be mixed (`projects: { globs: [...], sources: {...} }`).
- Glob-discovered ids are the folder name, so `apps/api` and `services/api` collide. In large trees set `globFormat: source-path` (id = workspace-relative path) or use a map.
- A project id is a public API: it is in targets, `dependsOn`, CI configs and remote cache keys. Renaming it is a breaking change for scripts and dashboards.
- Ecosystem toolchains add aliases (`package.json` `name`, Go module path, `Cargo.toml` name). Aliases work in targets but are fragile; write ids in config.
- `dependsOn` is usually inferred by toolchains (workspace `package.json` deps, Go imports with `inferRelationships`, Cargo path deps). Declare it explicitly only where inference cannot see the edge (a Go service calling a TS package's generated client, an e2e suite testing an app). Since v2.5, production and development dependency scopes are checked for cycles separately, so a test-only back edge is legal.

### Root project

```yaml
projects:
    root: "."
```

- With globs the root id is the checkout folder name, which differs between machines. Always map it explicitly.
- The root inherits every `.moon/tasks/*` file without `inheritedBy`. Give it its own tasks, select ecosystem files by toolchain/language/tag so they skip the root, or use `workspace.inheritedTasks.exclude`.
- Root tasks need explicit, narrow `inputs`. Since v1.24 root tasks default to no inputs, and a task without inputs is never affected: fine for `setup`/`lock` run by hand, wrong for anything `moon ci` or `--affected` should pick up. Older repositories may still hash the whole tree.
- `defaultProject: root` (v2.0+) makes `moon run lock` mean `moon run root:lock`.

### Layers, stacks and boundaries

```yaml
# project moon.yml
layer: application      # application | automation | configuration | library | scaffolding | tool | unknown
stack: backend          # frontend | backend | infrastructure | systems | unknown
tags: [node, css]
project:
    owner: team-payments
    channel: "#payments"
```

```yaml
# .moon/workspace.yml
constraints:
    enforceLayerRelationships: true   # a library cannot depend on an application
    tagRelationships:
        public-api: [public-api]      # projects tagged public-api may only depend on public-api projects
```

Boundaries are free architecture enforcement: the graph fails before any task runs. Turn them on early; retrofitting them into a large repository means fixing every violation at once.

### Code owners

`owners` in `moon.yml` plus `codeowners` in the workspace config generate `CODEOWNERS` (`moon sync code-owners`, or `codeowners.sync: true`). Worth it once you have more than a handful of teams; the ownership lives next to the project instead of in one file everyone edits.

## Toolchains and version pinning

```yaml
# .moon/toolchains.yml
proto:
    version: "0.62.3"
javascript:
    packageManager: pnpm
node: {}
pnpm: {}
typescript:
    syncProjectReferences: true
go:
    workspaces: true
unstable_python:
    packageManager: uv
unstable_uv: {}
```

```toml
# .prototools
moon = "2.6.0"
node = "24.10.0"
pnpm = "10.18.0"
go = "1.25.1"
python = "3.13.7"
uv = "0.9.2"
golangci-lint = "2.5.0"
```

- `versionFromPrototools` defaults to `true` (v2.0+): toolchains take their version from `.prototools`. Keep versions in one place; Renovate and Dependabot understand `.prototools`.
- `moon toolchain info <id>` prints every setting, the files the toolchain detects and which plugin APIs it implements (does it hash platform data, install dependencies, prune Docker images). Read it before relying on a behaviour.
- A toolchain does more than install the binary: it can install dependencies before tasks (`installDependencies`, on by default), sync manifests (`syncProjectWorkspaceDependencies`, TypeScript project references), add data to task hashes and infer project edges. Each of these writes files or runs commands you did not ask for. Enable sync features deliberately, and run `moon sync` locally so CI does not discover drift.
- `unstable_*` toolchains (Python, Ruby, Nub) may change between minor releases. Pin moon when you depend on them.
- Tools with no toolchain (linters, `hurl`, `terraform`) are pinned in `.prototools` and called by name; add `/.prototools` to `implicitInputs` in `.moon/tasks/all.yml` so a version bump invalidates caches.
- Corporate networks: `moon.downloadUrl`/`manifestUrl` and proto's `[settings.http]` point at internal mirrors; `PROTO_OFFLINE=1` forces offline mode (toolchains fall back to cached tools, then `PATH`).

## Sharing configuration across repositories

`extends` works in `.moon/workspace.yml`, `.moon/toolchains.yml`, `.moon/extensions.yml` and `.moon/tasks/*.yml`, with a relative path or an HTTPS URL.

- Always pin a remote `extends` to a commit or tag. A branch URL lets an upstream edit change every repository's tasks without a commit in any of them.
- Local settings merge over the extended file. Keep the shared file to conventions (task names, shells, options); keep project layout and secrets local.
- Templates are shared separately through `generator.templates` (`git://`, `npm://`, archives); see [codegen.md](codegen.md).

## Pipeline settings worth knowing

```yaml
pipeline:
    autoCleanCache: true          # deletes cache older than cacheLifetime after each run
    cacheLifetime: "7 days"
    installDependencies: [node]   # or false; limit auto-installs to some toolchains
    syncProjects: true
    syncWorkspace: true
    killProcessThreshold: 2000    # ms before children are killed after Ctrl+C
    logRunningCommand: false
hasher:
    walkStrategy: vcs             # or glob; vcs respects .gitignore
    warnOnMissingInputs: true
    ignoreMissingPatterns: ["**/.env", "**/.env.*"]
```

Telemetry is on by default (`telemetry: false` or `MOON_TELEMETRY=false` to opt out); some companies require turning it off.

## Daemon (v2.2+, unstable)

`unstable_daemon: true` (or `MOON_DAEMON=true`) keeps the project and task graphs in memory, watches files, and since v2.5 archives and hydrates outputs in the background.

- Worth it from a few hundred projects, or when agents and editors call moon many times per minute.
- Archive or hydration failures appear only in `moon daemon logs`, not in the command output. When outputs look missing, look there first.
- Try it with the env var on one machine before committing the setting.

## MCP server

`moon mcp` exposes the workspace (projects, tasks, templates, sync actions) to AI agents. Register it per project with `MOON_WORKSPACE_ROOT` set; since v2.6 it speaks only the stateless MCP `2026-07-28` protocol, so older clients cannot connect. moon also publishes an official `debug-task` agent skill (`npx skills add moonrepo/moon --skill debug-task`); [debugging.md](debugging.md) covers the same workflow.

## Pkl configs (v2.6 type checking)

`.moon/*.pkl` and `moon.pkl` can `amends ".../.moon/cache/schemas/pkl/ProjectConfig.pkl"` for type checking and editor completion. Pkl pays off when configs are generated or heavily parameterised; for hand-written configs YAML plus the JSON schemas (`$schema: https://moonrepo.dev/schemas/project.json`) is enough.

## VCS hooks

moon can write Git hooks from config:

```yaml
vcs:
    hooks:
        pre-commit:
            - moon run :lint :fmt --affected --status staged
    sync: true          # install for everyone on every run; otherwise `moon sync hooks` per developer
```

- moon writes scripts to `.moon/hooks/` and points `core.hooksPath` there. Disabling requires `moon sync hooks --clean` on every machine.
- **If the repository already uses lefthook, hk, husky, pre-commit, simple-git-hooks or any other hook manager, do not enable moon's hooks.** They compete for `core.hooksPath` and the last one to run wins. Call moon from the existing manager instead.
- Keep hooks to the staged/affected slice (`--affected --status staged`, `affectedFiles: args` for linters). A hook slower than a few seconds gets bypassed with `--no-verify`, and then it protects nothing; CI is the real gate.
