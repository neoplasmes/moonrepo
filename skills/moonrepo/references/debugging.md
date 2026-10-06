# Debugging tasks

Most task problems are one of: the task is not configured the way you think (inheritance), the hash includes too much or too little, or the environment differs between machines. Work through the steps in order; most issues end at step 3.

moon ships an official agent skill with the same workflow: `npx skills add moonrepo/moon --skill debug-task` (v2.2+). Use it alongside this file if it is installed.

## 1. See the resolved task

```shell
moon task app:build --json      # after inheritance, tokens expanded
moon project app                # which .moon/tasks files were inherited
```

Check: `command` and `args` (tokens expanded as you expect?), `inputFiles`/`inputGlobs`/`inputEnv`, `outputFiles`/`outputGlobs`, `deps`, `toolchains`, `options` (especially `cache`, `runInCI`, `runFromWorkspaceRoot`, `shell`). The same data, plus the inheritance chain (`inherited.config`, `inherited.layers`), is in `.moon/cache/states/<project>/snapshot.json`.

For a group, list matched files with a throwaway task (`command: echo @files(<group>)`, `cache: false`).

## 2. Run with full logs

```shell
MOON_DEBUG_PROCESS_ENV=true MOON_DEBUG_PROCESS_INPUT=true \
    moon run app:build --force --log trace --log-file moon.log
```

- `--log debug` is enough for most problems; `trace` (very verbose since v2.4) shows every child process, `verbose` adds spans.
- `MOON_DEBUG_PROCESS_ENV` reveals the full environment passed to the task (hidden by default to avoid leaking secrets; do not paste it into public issues).
- Find `Generated a unique hash task_target="app:build" hash="…"` and copy the hash.
- `pipeline.logRunningCommand: true` prints the command, args and working directory for every task.

## 3. Inspect and diff hashes

```shell
moon hash <hash>                 # the manifest: inputs, env, deps, toolchain entries
moon hash <hash-a> <hash-b>      # git-diff of two manifests
```

Get two hashes from two runs (or two machines: local vs CI log) and diff them. The diff names the culprit.

| Symptom                                    | Typical cause                                                                       |
| ------------------------------------------ | ----------------------------------------------------------------------------------- |
| Never cached locally                       | a volatile input (generated file, log, timestamp) inside an input glob; `cache: false` inherited from `taskOptions`; a declared output missing (the run fails) |
| Cached locally, never in CI                | absolute path in args (`$workspaceRoot`), env var input set only in CI, different tool version, different toolchain platform entry |
| Remote hits on Linux, never on macOS/Windows | platform in the hash (Go/Rust toolchains) — expected; or CRLF checkouts              |
| Stale result after a change                | the changed file is not an input (git-ignored, outside the globs, a script the command calls, a root config not in `cfg-*`) |
| Everything re-ran after a tiny edit        | edit to `.moon/workspace.yml`/`toolchains.yml`, `.prototools`, a lockfile, or a catch-all input |
| Dependency change does not rebuild         | dep without outputs defaults to `cacheStrategy: ignored`                           |
| Rebuilds whenever a dependency's tests change | `^:build` dep with `cacheStrategy: hash` where `outputs` was intended           |

Before v2.6 manifests are JSON files in `.moon/cache/hashes/<hash>.json`; since v2.6 they are blobs in the local CAS and `moon hash` is the way to read them.

## 4. Environment and shell differences

- Task shell: `unixShell` falls back to `$SHELL`, `windowsShell` to `pwsh`. Pin both.
- `env` loses to the shell/CI environment unless `envOverride` (v2.6). `envFile` never overrides.
- Toolchain activation (v2.6) sets tool env vars (`JAVA_HOME`, `GOROOT`) and prepends `PATH`; a task that worked by accident with a global tool may now use the pinned one.
- Working directory: project root by default, workspace root with `runFromWorkspaceRoot`. `PWD` in the task tells you which.
- Run the exact command by hand from the same directory to separate moon problems from tool problems.

## 5. Affected and CI differences

```shell
moon query changed-files --base origin/main --head HEAD
moon query affected --upstream deep --downstream direct
moon ci --base origin/main --head HEAD     # reproduce CI selection locally
```

- Nothing ran in CI: shallow clone, wrong base (push to main compares `HEAD~1`), files not matching any task's inputs, `runInCI: false`.
- Too much ran: catch-all inputs, an implicit input touched by the change, `--downstream deep` where `direct` was meant.

## 6. Slow runs

| Question                              | Tool                                                                                      |
| ------------------------------------- | ----------------------------------------------------------------------------------------- |
| Where does moon itself spend time?    | `moon run <target> --dump`, open the trace in `chrome://tracing` or ui.perfetto.dev       |
| Which tasks are slow across CI runs?  | OpenTelemetry metrics `moon.task.duration` (v2.6) or webhooks `task.ran` durations         |
| Why is a Node task slow?              | `moon run --profile cpu <target>` / `--profile heap`                                      |
| Graph building slow?                  | daemon (v2.2+), `globFormat`/explicit project map, restrict Go `inferRelationshipsPackages` |
| Hashing slow?                         | huge input globs; `hasher.ignorePatterns` for binary assets that never matter            |

See [observability.md](observability.md).

## 7. Validate the fix

1. Run the task twice: first a miss, then `cached`.
2. Change a file that must invalidate it: miss. Change one that must not: hit.
3. On CI, confirm the hash printed in the log matches the local one for the same commit (when the platform is the same).
4. Clean up throwaway tasks and `--log-file` outputs.

## Other diagnostics

- `MOON_DEBUG_REMOTE=true`: remote cache connection errors and extra logging.
- `MOON_DEBUG_WASM=true`: WASM plugin logs, memory/core dumps.
- `moon daemon logs`: archive/hydration failures when the daemon is enabled (v2.5+).
- `moon action-graph <target>`, `moon task-graph`, `moon project-graph`: visual graphs (served locally, or `--dot`).
- `moon clean --all` resets the cache; use it to rule out a corrupted cache, not as a fix.
