---
name: moonrepo
description: Rules and verified recipes for moon (moonrepo.dev) workspaces in any language; moon is the only entry point (moon run / moon ci, never npm scripts, Make or direct tool calls). Use before creating or editing moon.yml, .moon/*, .prototools or a package manifest in a moon workspace; when adding a project, task, toolchain or language (TypeScript, Go, Python and others); when writing inputs, outputs or file-group globs; when a task is not cached, re-runs or behaves differently in CI; when setting up CI, CI caching or a remote cache; when writing a Dockerfile for a moon project; when creating code-generation templates; when considering WASM plugins or extensions; when wiring webhooks, OpenTelemetry or profiling; and when porting scripts, Makefiles, Nx or Turborepo to moon.
license: Moonrepo Skill License (free use, no resale) - https://github.com/neoplasmes/moonrepo/blob/master/LICENSE
metadata:
  author: Egor Zverkov (neoplasmes)
  source: https://github.com/neoplasmes/moonrepo
  verified-on: moon 2.5.5 (docs of 2.6.0)
---

# moon

moon is a task runner and build system: a workspace of projects, each with tasks, connected into a graph. A task's hash is computed from its command, args, inputs, env, dependencies and toolchain versions; a matching hash replays the task's outputs from the cache instead of running it. Almost every rule below exists to keep that hash honest: include everything that changes the result, nothing that does not.

## Before you change anything

1. Find the repository contract. Read `AGENTS.md`, `CLAUDE.md`, `ARCHITECTURE*.md`, a "Local adaptation" section if this skill was copied into the repo, and the existing `.moon/tasks/*`. **Repository rules win over the defaults in this skill.** The defaults apply to new workspaces and to gaps the repository does not cover.
2. Check the version: `moon --version`, `.prototools`, `versionConstraint` in `.moon/workspace.yml`. Features below carry the version that introduced them; do not use a v2.6 option in a v2.5 repository. The local CLI may lag behind the docs.
3. Read the project's `moon.yml` and what it inherits: `moon project <id>` lists inherited configs, `moon task <project>:<task> --json` shows the merged result.

### Local adaptation

When this skill is copied into a repository, put a `## Local adaptation` section right under the title. It overrides the defaults below and should state: the tool executor (`pnpm exec`, `bunx`, `uv run --locked`, `go tool`), the formatter setup, the pinned task shell, the project layout and root project id, task names that differ from the default vocabulary, and what is out of scope (Docker, deploys, container test suites).

## MOONREPO FIRST

In a moon workspace, moon is the only entry point for project operations. These rules are hard:

1. **Run through moon.** Format, lint, typecheck, test, build, generate, start and deploy-preparation steps run as `moon run <target>` (or `moon ci` in CI). Do not run `npm run`, `pnpm run`, `make`, `just`, `task`, `go test`, `pytest`, `cargo test` or another tool directly to do a job that has, or should have, a moon task. A direct tool call bypasses the cache, the affected graph, dependency ordering and the hash, so its result tells you nothing about what CI will do. Calling a tool directly is fine to investigate a failure after `moon run` reported it, never as the way to run the operation.
2. **A new operation becomes a moon task first.** If an operation has no task, create the task (see [Decomposition](#decomposition)), then run it through moon. Do not run it ad hoc and promise to "add the task later".
3. **No parallel entry points.** Do not add `scripts` to `package.json`, targets to a `Makefile`/`justfile`/`Taskfile`, `project.json` targets, `tox`/`nox` sessions, `pyproject` script runners, shell wrappers, or CI steps that repeat a task's command. Every such file is a second source of truth that drifts from the moon task and escapes the cache. CI calls `moon ci` or `moon run`, never the underlying tools.
4. **Existing parallel entry points are migration debt.** When you touch one, move its logic into a moon task and delete the old entry or reduce it to a one-line wrapper (`"test": "moon run :test"`) that exists only so muscle memory keeps working. See [migration.md](references/migration.md).
5. **Exceptions come only from the repository.** If `AGENTS.md`, `ARCHITECTURE*.md` or the Local adaptation section says otherwise (for example "root `package.json` keeps a `prepare` script for hooks"), follow the repository. Without such a written exception, rules 1 to 4 apply. "It is quicker to run it directly" is not an exception.

Manifests keep what is theirs: dependencies, metadata, `engines`, `bin`, lifecycle hooks the package manager requires (`postinstall` for native modules). They are not task runners.

## References

| Read                                                  | When                                                                                     |
| ----------------------------------------------------- | ---------------------------------------------------------------------------------------- |
| [patterns.md](references/patterns.md)                 | Writing or reviewing a file group, input or output glob.                                 |
| [decomposition.md](references/decomposition.md)       | Deciding where a task lives, splitting tasks, deps versus inputs, inheritance, overrides. |
| [workspace.md](references/workspace.md)               | Workspace setup: projects, toolchains and proto, layers, constraints, shared config, daemon, MCP, VCS hooks. |
| [lang-typescript.md](references/lang-typescript.md)   | A JavaScript or TypeScript project (Node, Bun, Deno; pnpm, npm, yarn).                   |
| [lang-go.md](references/lang-go.md)                   | A Go module or a Go workspace.                                                           |
| [lang-python.md](references/lang-python.md)           | A Python project (uv, pip, Poetry).                                                      |
| [lang-other.md](references/lang-other.md)             | Rust, Terraform, Hurl, shell and script-only projects, any other language.               |
| [caching.md](references/caching.md)                   | Anything about hashes, outputs, cache misses, CI cache persistence or a remote cache.    |
| [ci.md](references/ci.md)                             | Writing or debugging a CI pipeline that runs moon.                                       |
| [docker.md](references/docker.md)                     | A Dockerfile, `moon docker *`, compose tasks, containers on Windows.                     |
| [debugging.md](references/debugging.md)               | A task fails, re-runs, never caches, or differs between machines; slow runs.             |
| [observability.md](references/observability.md)       | Webhooks, OpenTelemetry, run reports, terminal notifications, task profiling.            |
| [codegen.md](references/codegen.md)                   | You notice a repeated structure, or write or run a `moon generate` template.             |
| [wasm-plugins.md](references/wasm-plugins.md)         | Custom toolchains, extensions (`moon ext`), proto tool plugins; deciding if WASM is worth it. |
| [migration.md](references/migration.md)               | Porting scripts, Makefiles, `package.json` scripts, Nx or Turborepo to moon.             |
| [scale.md](references/scale.md)                       | Large monorepos, many teams, mixed OS fleets, merge queues, cost and security trade-offs. |

## Workflow

1. Decide where the change belongs: the most general layer that is true for every inheritor (see [Decomposition](#decomposition)).
2. Write file groups first, then tasks that reference them. Never write a cached task with catch-all inputs.
3. Name tasks with the repository vocabulary, or the default one below.
4. Run the checks in [Verify before finishing](#verify-before-finishing).

## Mental model

- **Workspace**: `.moon/workspace.yml` lists projects; `.moon/toolchains.yml` enables language toolchains; `.moon/tasks/**/*.yml` holds inherited tasks; `.prototools` pins tool versions.
- **Project**: a folder with an optional `moon.yml`: `language`, `layer`, `stack`, `tags`, `dependsOn`, `fileGroups`, `tasks`.
- **Task**: `command` (or `script`), `args`, `inputs`, `outputs`, `deps`, `env`, `options`. Target syntax: `project:task`, `:task` (all projects), `#tag:task`, `~:task` (this project), `^:task` (dependency projects).
- **Hash**: command and args, every input file's content hash, input env vars, deps' hashes (by `cacheStrategy`), toolchain versions and toolchain-specific content, plus `.moon/workspace.yml` and `.moon/toolchains.yml`. Absolute paths and unpinned tools leak machine state into it.
- **Affected**: changed files (VCS) intersected with task inputs. Only tracked, non-ignored files count.

## Task vocabulary (default)

Use the repository's names if it has them. Otherwise:

| Group        | Tasks                                                                                         |
| ------------ | --------------------------------------------------------------------------------------------- |
| Setup        | `setup` (install tools and dependencies), `lock` (refresh the lockfile)                       |
| Quality      | `fmt` (formatter, writes), `lint` (read-only), `lint-fix` (writes), `typecheck`, `check` (proxy) |
| Tests        | `test-unit`, `test-int`, `test-e2e`                                                           |
| Delivery     | `build` (declares `outputs`), `dev` (persistent), `start` (runs the build), `image`, `compose-up`, `compose-down` |

- `check` is a proxy: `command: noop`, `deps` on `fmt`, `*-fix` and `typecheck`, and `inputs` = the groups its deps read (`@group(src)`, `@group(test-*)`, `@group(cfg)`). What can be fixed automatically gets fixed; what cannot still fails.
- **A task with `inputs: []` is never affected**, even when its deps are (verified): `moon run :check --affected` and `moon ci :check` then run nothing, and an `image` task with no inputs never runs in `moon ci`. Give every proxy or orchestration task that may be selected by `--affected` the inputs of the work it stands for.
- `fmt` and `*-fix` write files, so CI either runs the read-only tasks (`lint`, `typecheck`, tests) plus a formatter check, or runs `check` and fails on `git diff --exit-code`. Pick one convention and keep it everywhere.
- Name the operation, never the tool or language: `lint`, not `eslint`/`golangci`; `lock`, not `tidy`. Same operation, same name in every project and language, so `moon run :lint` works across the workspace.
- kebab-case, verb first; a narrower variant adds a scope (`lint-css`, `test-int`); a fixing variant ends in `-fix`; a step of a bigger task is `<task>-<part>` with `internal: true`.
- Language-mandated test layouts (`*_test.go`, Rust `tests/`, `tests/test_*.py`) keep their file names and map onto `test-unit`/`test-int`.

## Decomposition

| Layer                             | Selected by                                  | Holds                                                            |
| --------------------------------- | -------------------------------------------- | ---------------------------------------------------------------- |
| `.moon/tasks/all.yml`             | every project                                | inputs and options true for any language (`/.prototools`, shell) |
| `.moon/tasks/<ecosystem>.yml`     | `inheritedBy: { toolchains / tags }`         | lockfile and manifest inputs, shared quality tasks               |
| `.moon/tasks/<language>.yml`      | `inheritedBy: { languages }`                 | compiler and type-checker tasks                                  |
| `.moon/tasks/<capability>.yml`    | `inheritedBy: { tags }`                      | optional capability tasks (`lint-css`) and their `check` wiring  |
| project `moon.yml`                | the project                                  | file groups, `dependsOn`, project-only tasks, additions          |

- A root config used by one tool (`/.golangci.yml`, `/ruff.toml`) goes into a `cfg-<tool>` group used by that tool's tasks, never into `implicitInputs`; otherwise editing it invalidates every cache.
- Overriding an inherited task only adds; lists merge by `append`. Use `options.merge*: replace` only when appending is wrong, and say why.
- `deps` express order and output invalidation; `inputs` express which files make a task stale. Never add a dep to get a file into the hash, never add an input to get ordering.
- Inside `.moon/tasks/*` write deps as `~:task`; in a project `moon.yml` write `project:task`.

Details, inheritance conditions and verified examples: [decomposition.md](references/decomposition.md).

## File groups and globs

- Every project defines role-based groups: `src` (sources only), `test-unit`, `test-int`, `cfg` (the project's own config files). Add `gen`, `assets`, `migrations`, `stories` when a task needs them. Shared task files declare empty defaults, because referencing a missing group fails with `unknown_file_group`.
- Explicit extensions, suffixes and exclusions. Never `**/*` or `src/**/*` in a cached task.
- Globs are Rust globs (wax), not JS globs: `/` separators even on Windows, no leading `./` or `..`, a leading `/` means workspace-relative.
- Pitfalls: `{,.int}` does not parse; a `{a,b}` alternative must not contain `/`; there is no `!(...)` extglob; `src/*.ts` is one level only; git-ignored files are never inputs; `?` cannot be used in the `file://`/`glob://` URI form.
- `@files(group)` passes matched files with exclusions applied; `@globs(group)` passes raw patterns for tools that expand them. Prefer `@files`; use `@globs` when the file list would exceed the Windows command-line limit (~32k chars).

Cookbook: [patterns.md](references/patterns.md).

## Commands

- Call the tool directly from the task (`ruff check`, `go vet ./...`, `pnpm exec vitest run`), not through `package.json` scripts, `make` targets or wrapper scripts. The real command then lives in the hash and in `moon task --json`. This is the other half of [MOONREPO FIRST](#moonrepo-first): people run moon, moon runs the tool.
- Pin every CLI the tasks call: in `.prototools` (and therefore in the hash through the `/.prototools` implicit input), in the language's own manifest (`go.mod` `tool` directives, `devDependencies`, `uv` dev groups), or both. No global installs.
- Pin the task shell in `.moon/tasks/all.yml` `taskOptions` (`unixShell`, `windowsShell`). Unset, `unixShell` comes from the developer's `$SHELL`, so a fish user and a bash user run different shells. `windowsShell` defaults to `pwsh`.
- Multi-step logic goes into a script file under `tools/` called by the task, or into several tasks wired with `deps`. No long `bash -c '... && ... && ...'` one-liners: they hide failures, defeat per-step caching and do not run on Windows.
- Inputs, outputs and globs never use `..`: reach root files with a workspace-relative input (`/file`). In commands, prefer tools that discover their config by walking up (golangci-lint, ruff, dprint, eslint); `$workspaceRoot`/`$projectRoot` expand to absolute paths **inside the hash** (verified), so machines with different checkout paths never share remote-cache hits.
- `cache: false` only for tasks with side effects or no reproducible result (`dev`, `start`, `setup`, `lock`, `compose-*`, deploys). Interactive helpers use `preset: utility`, long-running servers `preset: server`. Every other task declares `inputs`, and every task that produces files declares `outputs`.

## Caching, CI, Docker in one paragraph each

**Cache.** Local cache lives in `.moon/cache` and must be git-ignored and docker-ignored. Outputs are archived per hash and hydrated on a hit. Toolchains decide what platform data enters the hash: the Go toolchain adds os/arch/libc, Node adds only its version, Python adds nothing. Treat a remote cache as shared state with a security boundary: CI on protected branches writes, everyone else reads. See [caching.md](references/caching.md).

**CI.** Run `moon ci` (affected tasks, their upstream deps, direct dependents, `--on-failure=continue`). Check out full history (`fetch-depth: 0`, `filter: blob:none`); a shallow clone breaks affected detection. On a push to the default branch moon compares with `HEAD~1`, so a multi-commit push needs `MOON_BASE` set to the previous tip. See [ci.md](references/ci.md).

**Docker.** `moon docker scaffold/setup/prune` gives `O(1)` Dockerfiles for dependency-heavy ecosystems (Node, Python). For Go and Rust services, building the binary in a moon task and copying it into a minimal image is usually simpler and remote-cacheable. Without `.git` in the build context moon disables task caching inside the build. See [docker.md](references/docker.md) for Windows/WSL2 caveats.

## VCS hooks

moon can generate Git hooks from `vcs.hooks` (`moon sync hooks`, or `vcs.sync: true` to install them for everyone). If the repository already uses lefthook, hk, husky, pre-commit or similar, do not enable moon's hooks: both write `core.hooksPath` and will fight. Call moon from the existing hook manager instead (`moon run :lint --affected --status staged`). Details: [workspace.md](references/workspace.md#vcs-hooks).

## Templates

When you are about to create a structure that already exists at least twice (a service, a package, a page slice, an ADR), or you create the second copy yourself, stop and propose a `moon generate` template: show the repeated paths, the variables and the destination. Create it only after the user agrees; afterwards generate with it instead of copying by hand. See [codegen.md](references/codegen.md).

## Version notes

| Version | What it enables                                                                                         |
| ------- | ------------------------------------------------------------------------------------------------------- |
| 2.0     | `.moon/toolchains.yml` (WASM toolchain plugins), `.moon/extensions.yml`, `inheritedBy`, `defaultProject` |
| 2.1     | Execution plans (`moon exec --plan`), `runInSyncPhase`                                                  |
| 2.2     | Daemon (unstable), official `debug-task` AI skill                                                       |
| 2.3     | Task `tags`, dep `cacheStrategy`, native file hashing, local CAS (experiment)                           |
| 2.4     | Task `checks` (requirement, condition, fingerprint), project-level `taskOptions`, Poetry, Ruby          |
| 2.5     | OpenTelemetry, workspace `env` in `.moon/tasks`, `workspace.mergeStrategies`, project discovery by file glob, shared worktree cache (unstable) |
| 2.6     | Dep `type: cleanup`/`wait`, `expectFailure`, `envOverride`, `--output-style`, persistent tasks never run in CI by default, remote HTTP retries, Pkl type checking, hash manifests in CAS (`moon hash`) |

## Verify before finishing

1. `moon query projects` lists the expected projects; `moon project <id>` shows the expected inherited configs.
2. `moon task <project>:<task> --json` for every task you touched: check `command`, `args`, `inputFiles`, `inputGlobs`, `inputEnv`, `outputs`, `deps`, `options`.
3. For a new or changed group, list what it matches with a throwaway task (`command: echo @files(<group>)`, `cache: false`), run it once, remove it.
4. Run the task twice: the second run must say `cached`. Touch a file that should invalidate it and one that should not; confirm both. `moon hash <a> <b>` diffs two hashes.
5. `moon run :check` (or the repository's equivalent) and the affected tests pass, run through moon and not through the tools directly.
6. No new `scripts`, Make targets or CI steps duplicate a task ([MOONREPO FIRST](#moonrepo-first)).
6. Update the repository's architecture or conventions document in the same change when a rule changes.
