# Task decomposition

How to split configuration between layers, between tasks, and between projects. Examples were checked on moon 2.5 (`moon task <target> --json`); v2.6 features are marked.

## 1. One fact, one layer

Ask for each line you are about to write: "for which projects is this true?"

| True for                                   | Goes to                                        | Example                                   |
| ------------------------------------------ | ---------------------------------------------- | ----------------------------------------- |
| every project                              | `.moon/tasks/all.yml`                          | `/.prototools` input, pinned task shell   |
| every project of a package manager         | `.moon/tasks/<ecosystem>.yml` by toolchain/tag | `/pnpm-lock.yaml`, `/uv.lock`, `fmt`      |
| every project of a language                | `.moon/tasks/<language>.yml` by language       | `typecheck` with `tsc`, `go vet`          |
| projects that opted into a capability      | `.moon/tasks/<capability>.yml` by tag          | `lint-css` for tag `css`                  |
| one project                                | its `moon.yml`                                 | `build` with Vite, file groups            |
| one root config used by one tool           | a `cfg-<tool>` group next to that tool's tasks | `cfg-css: [/stylelint.config.js]`         |

`inheritedBy` (v2.0+) conditions combine with AND; each takes one value or a list (OR). Keys (singular or plural): `toolchains`, `languages`, `tags`, `stacks`, `layers`, `files` (literal paths, no globs: selects projects containing that file). `toolchains` and `tags` accept `and`, `or` and `not` clauses. A file without `inheritedBy` is inherited by every project. `languages` only matches a `language` written in `moon.yml`; a detected language does not count, so a project without `language:` silently inherits nothing from a language-selected file.

```yaml
inheritedBy:
    languages: typescript
    tags:
        not:
            - e2e
```

```yaml
inheritedBy:
    toolchains:
        or: [javascript, typescript]
    layers: [library, application]
```

The root project inherits `all.yml` too. That is why `fmt` and `check` live in the ecosystem files and not in `all.yml`: a root project that defines its own `fmt` and `check` would get an inherited copy merged into them.

Prefer `toolchains` over `tags` when the condition really is "uses this toolchain": the toolchain is detected from the project's files, a tag must be remembered by hand. Prefer `tags` for opt-in capabilities.

### Workspace-level env and options

- `.moon/tasks/*.yml` `taskOptions` sets defaults for every task in the file; project `moon.yml` `taskOptions` (v2.4+) sets defaults for every task of that project and wins over the workspace layer; a task's own `options` win over both.
- `.moon/tasks/*.yml` `env` (v2.5+) is merged into every inheriting project's `env`; project values win. `workspace.mergeStrategies` in `moon.yml` (`env`, `fileGroups`: `append | prepend | preserve | replace`) changes how a project merges with inherited values.

## 2. Check, fix and the `check` proxy

A read-only task and its fixing twin differ by one flag. Two ways to share the rest:

```yaml
tasks:
    lint:
        command: ruff check --no-fix $projectSource
        inputs: &lint-inputs
            - "@group(src)"
            - "@group(test-unit)"
            - "@group(cfg-lint-py)"

    lint-fix:
        command: ruff check --fix $projectSource
        inputs: *lint-inputs
```

```yaml
tasks:
    lint-fix:
        extends: lint
        args: --fix
```

`extends` copies command, inputs, deps and options of the sibling (also an inherited sibling inside `.moon/tasks/*.yml`) and appends `args`. Use it when the fix flag may go last; otherwise keep the anchor form. YAML anchors do not cross files: an anchor in `.moon/tasks/python.yml` cannot be used in a project `moon.yml`.

`check` never runs anything itself, but it needs inputs:

```yaml
tasks:
    check:
        command: noop
        inputs:
            - "@group(src)"
            - "@group(test-unit)"
            - "@group(test-int)"
            - "@group(cfg)"
        deps:
            - ~:fmt
            - ~:lint-fix
```

With `inputs: []` the proxy is never affected by changed files, so `moon run :check --affected --status staged` in a pre-commit hook prints "No tasks affected" and checks nothing (verified on moon 2.5; `--include-relations` does not change it). A `noop` with inputs costs only hashing. The same applies to any orchestration task meant to be selected by `--affected` or `moon ci <target>`.

Each capability file adds itself to it (`css.yml` adds `~:lint-css-fix` to `check` and to `fmt`), so a project gets exactly the checks its tags and language select.

Fix tasks write into source files. Two fix tasks touching the same files must not run in parallel: order them with `deps` (`fmt` depends on `lint-fix`), or share an `options.mutex`.

## 3. Splitting a task into steps

When one operation has independent parts, keep the public name and hide the parts:

```yaml
tasks:
    build-client:
        command: vite build
        inputs:
            - "@group(src)"
            - "@group(css)"
            - "@group(cfg)"
        outputs:
            - dist/client
        options:
            internal: true
            runFromWorkspaceRoot: false

    build-server:
        command: vite build --ssr src/entry.server.tsx --outDir dist/server
        inputs:
            - "@group(src)"
            - "@group(cfg)"
        outputs:
            - dist/server
        options:
            internal: true
            runFromWorkspaceRoot: false

    build:
        command: noop
        inputs:
            - "@group(src)"
            - "@group(css)"
            - "@group(cfg)"
        deps:
            - frontend:build-client
            - frontend:build-server
```

Each step is cached on its own inputs, both run in parallel, and people keep running `moon run frontend:build`. An `internal` task cannot be run from the command line, only depended on. The proxy repeats the steps' inputs so that `moon ci :build` selects it.

## 4. Deps versus inputs

| You need                                                      | Use                                                                  |
| ------------------------------------------------------------- | -------------------------------------------------------------------- |
| Task B must run after task A                                  | `deps: [a-project:a]`                                                |
| B consumes A's build output                                   | `deps` with `cacheStrategy: outputs` (B reruns only when A's output changes) |
| B reads another project's sources directly (no build step)    | input `project://<id>?group=<group>`, plus `dependsOn: [<id>]`       |
| B and C must not run at the same time (shared DB, port, file) | `options.mutex: <resource-name>` on both                             |
| B is ordering only, its changes must not invalidate A         | default for deps without outputs (`cacheStrategy: ignored`)          |
| B needs a server running while it runs (v2.6)                 | `deps: [{ target: api:serve, type: wait }]`                          |
| Something must be torn down after B, pass or fail (v2.6)      | `deps: [{ target: db:stop, type: cleanup }]`                         |
| B needs a dep that only some projects define                  | `deps: [{ target: ~:codegen, optional: true }]`                      |

```yaml
tasks:
    build:
        deps:
            - target: design-system:build
              cacheStrategy: outputs
```

A dependency without outputs (a check, a test) never invalidates the dependent task by default; a dependency with outputs (a build) does (`cacheStrategy: hash`). `outputs` (v2.3+) is the best choice for build chains: B reruns only when A's artifact bytes change, not when A's comments change.

### Service stacks for integration and e2e tests (v2.6)

```yaml
tasks:
    test-e2e:
        command: playwright test
        deps:
            - db:compose-up                 # required: must finish first
            - target: api:serve             # wait: only has to start
              type: wait
            - target: api:stop
              type: cleanup                 # runs after, pass or fail
            - target: db:compose-down
              type: cleanup
        options:
            cache: false                    # a cache hit would start and stop the stack for nothing
```

- `wait` means started, not ready: poll a health endpoint in the test setup.
- A task that runs in CI cannot depend on one that does not: give `api:serve` `runInCI: true` (persistent tasks default to false since v2.6).
- Cleanups do not run on Ctrl+C. Make `compose-up` idempotent (`docker compose up -d --wait`) so a leftover stack is harmless.
- Before v2.6, keep start, test and stop as separate tasks and a `mutex`, and run the teardown from CI with `if: always()`.

## 5. Overriding inherited tasks

- Add to an inherited task by redeclaring only the addition. Lists merge with `append`:

```yaml
tasks:
    lint-css:
        deps:
            - design-system:lint-css
```

- Change a list completely with `options.mergeInputs: replace` (also `mergeArgs`, `mergeDeps`, `mergeEnv`, `mergeOutputs`). Explain the reason in the commit message.
- Drop or rename inherited tasks per project:

```yaml
workspace:
    inheritedTasks:
        exclude:
            - typecheck
        rename:
            build: build-docs
```

Excluding is a smell: usually the project has the wrong tag or language. Fix that first.

## 6. Options worth knowing

| Option / field                     | Use it for                                                                 |
| ---------------------------------- | -------------------------------------------------------------------------- |
| `preset: server`                   | `dev`, `start`: no cache, streamed output, persistent, not in CI           |
| `preset: utility`                  | interactive helpers: no cache, interactive, streamed, skipped in CI        |
| `options.internal: true`           | steps that only exist to be depended on                                    |
| `options.os: windows` / `linux`    | OS-specific variants of one operation                                      |
| `options.envFile: .env`            | loading a dotenv file without a wrapper script (`/.env.shared` from the root) |
| `options.envOverride` (v2.6)       | task `env` wins over the shell/CI env (`true` or a list of names)          |
| `options.expectFailure` (v2.6)     | a check known to fail during a migration; the pipeline fails once it passes |
| `options.allowFailure`             | advisory tasks; dependents cannot rely on it                               |
| `options.runInCI`                  | `true`/`affected`, `always`, `false`, `only`, `skip` (see [ci.md](ci.md))  |
| `options.cache`                    | `true`, `false`, `local`, `remote`; `cacheKey` to bust, `cacheLifetime` to expire |
| `options.priority`                 | `critical`/`high` for long tasks on the critical path                     |
| `options.outputStyle`              | `buffer-only-failure` for chatty transitive deps                          |
| `options.affectedFiles: args`      | passing only changed files under `--affected`; outside it moon appends `.`, so not for tasks that pass `$projectSource` from the workspace root |
| `options.timeout`, `retryCount`    | bounding hung or network-bound tasks                                       |
| `checks` (v2.4)                    | `requirement`: fail fast if a tool or daemon is missing (`docker info`); `condition`: skip when all pass; `fingerprint`: hash a command's output (tool version, platform) |
| `description`                      | text shown by `moon task` and `moon project`; write it for non-obvious tasks |
| task `tags`                        | selecting tagged tasks of a project: `moon run 'frontend:#quality'`        |

## 7. Tokens you will need

| Token                                  | Expands to                                              |
| -------------------------------------- | ------------------------------------------------------- |
| `$projectSource`                       | project path from the workspace root (`apps/frontend`)  |
| `$projectRoot`, `$workspaceRoot`       | absolute paths                                          |
| `$project`, `$task`, `$target`         | ids of the running task                                 |
| `@files(group)`, `@globs(group)`       | matched files, or the group's glob patterns             |
| `@dirs(group)`, `@root(group)`         | matched directories, or their lowest common directory   |
| `@in(0)`, `@out(0)`                    | the task's own input or output by index                 |
| `@meta(key)`                           | a value from the project's `project:` metadata          |
| `$MOON_*` env inside the process       | `MOON_PROJECT_ID`, `MOON_TARGET`, `MOON_TASK_HASH`, `MOON_PROJECT_SNAPSHOT`, ... |

Absolute tokens (`$workspaceRoot`) in `args` become part of the hash, so a machine with a different checkout path gets a different hash. Prefer `$projectSource` and workspace-relative paths when the task runs from the workspace root.
