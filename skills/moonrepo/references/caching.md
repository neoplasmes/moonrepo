# Caching: local, CI and remote

moon skips a task when its hash matches a previous run and replays the recorded outputs and logs. This file explains what goes into the hash, how outputs are stored, and how to share the cache between CI runs and machines without sharing wrong results. Observations marked "verified" were made with moon 2.5.5 hash manifests.

## 1. What is hashed

A hash manifest (`moon hash <hash>`) is a JSON array. For a Go task it looked like this (verified):

```json
[
  {
    "command": "echo", "args": ["go-build"],
    "inputs": {
      ".moon/toolchains.yml": "a8e3…", ".moon/workspace.yml": "483f…",
      "apps/api/cmd/api/main.go": "a774…", "apps/api/go.mod": "d66b…"
    },
    "inputEnv": {}, "env": {}, "deps": {}, "outputs": [{ "file": "bin/api" }],
    "projectDeps": [], "target": "api:build", "toolchains": ["go"], "version": "3"
  },
  { "toolchain": "go", "version": "1.25.1", "contents": [{ "os": "linux", "arch": "x64", "libc": "gnu" }] },
  { "checks": { "uname -sm": { "stdout": "Linux x86_64" } } }
]
```

| Source                         | Notes                                                                                                 |
| ------------------------------ | ----------------------------------------------------------------------------------------------------- |
| `command`, `args`              | After token expansion: `$workspaceRoot`, `$projectRoot` become absolute paths (verified).             |
| `inputs`                       | Content hash of every matched, tracked, non-ignored file. `.moon/*.yml` and the `.moon/tasks/*.yml` file a task is inherited from are added automatically (verified). |
| `inputEnv`, `env`              | Values of `$VAR` inputs and the task's `env`.                                                         |
| `deps`                         | Dependency hashes or output hashes, by `cacheStrategy` (`hash`, `outputs`, `ignored`). `wait`/`cleanup` deps never count. |
| `projectDeps`                  | `dependsOn` ids.                                                                                     |
| toolchain entries              | Version always. Platform data only where the toolchain implements it: Go and Rust add os/arch/libc; Node adds the version and resolved dependency versions; Python adds nothing (verified via `moon toolchain info`). |
| `checks` fingerprints (v2.4+)  | stdout / stderr / exit code of a fingerprint script.                                                  |
| `options.cacheKey`             | Any string you bump to invalidate a task everywhere, including the remote cache.                     |

Consequences:

- Editing `.moon/workspace.yml` or `.moon/toolchains.yml` invalidates **every** task in the workspace. Keep volatile, per-environment settings (remote host, token variable, webhook URL) in environment variables (`MOON_REMOTE_HOST`, `MOON_WEBHOOK_URL`, ...) instead of committing them to `workspace.yml` back and forth.
- Editing `.moon/tasks/python.yml` invalidates every task inherited from it, which is correct.
- The hash knows nothing about the OS or CPU unless a toolchain or a check adds it. See [section 6](#6-portability-between-machines).
- Without a `.git` directory moon disables task caching and hashes no inputs (verified: `"inputs":{}` and "Caching is disabled for task"). This matters inside Docker builds.

## 2. Outputs, archiving and hydration

- `outputs` are files or globs relative to the project (or `/`-prefixed, workspace-relative). After a successful run they are archived; on a later hit with outputs missing or changed, they are restored ("hydrated").
- A declared output that does not exist after the run is an error. Optional outputs must be globs.
- A task with no `outputs` is still cached: a hit skips it and replays its stdout/stderr. That is what makes `lint`, `typecheck` and tests cheap.
- Do not declare tool caches (`node_modules/.cache`, `.next/cache`, `target/`, `.venv`) as outputs. They are large, machine-specific and already handled by the tool.
- Since v2.3 the `casOutputsCache` experiment stores outputs content-addressed (dedupe across tasks, reflinks); since v2.6 hash manifests always live in the local CAS (`.moon/cache/blobs`). With the v2.5 daemon, archiving and hydration happen in the background and their errors appear only in `moon daemon logs`.

## 3. Controls

| Need                                              | Use                                                                 |
| ------------------------------------------------- | ------------------------------------------------------------------- |
| Run once without reading the cache                | `moon run <target> --force` (`MOON_FORCE`)                          |
| Read the cache but never write (PR builds)        | `--cache read` / `MOON_CACHE=read`                                  |
| Write but never read (cache warmers)              | `--cache write`                                                     |
| No cache at all for this invocation               | `--cache off`                                                       |
| Never cache a task                                | `options.cache: false`                                              |
| Cache a task only locally / only remotely         | `options.cache: local` / `remote` (v1.40+)                          |
| Invalidate a task everywhere                      | bump `options.cacheKey`                                             |
| Expire a task's cache by time                     | `options.cacheLifetime: "1 day"` (useful for tasks reading external state) |
| Keep the local cache small                        | `pipeline.cacheLifetime` (default 7 days, auto-cleaned), `cache.cas.maxSize`, `moon clean` |
| Share one cache between git worktrees (agents)    | `experiments.casOutputsCache: true` + `cache.unstable_sharedWorktreeCache: true` (v2.5) |

Tasks that should not be cached: anything with side effects (deploys, migrations against a real database, `compose-up`), anything whose result depends on state moon cannot hash (network, clock, external APIs), interactive and persistent tasks.

## 4. Local cache hygiene

- `.moon/cache` and `.moon/docker` are git-ignored and docker-ignored. `moon docker scaffold` warns when `.dockerignore` misses `.moon/cache`.
- A cache hit on an unchanged checkout with outputs present prints `cached` and does not even hydrate.
- Many worktrees (one per branch or per AI agent) each start cold. On v2.5+ the shared worktree cache fixes that; before, a remote cache with `localReadOnly` is the workaround.

## 5. CI caching without a remote cache

You can persist moon's cache between CI runs with the CI system's own cache.

```yaml
# GitHub Actions
- uses: actions/cache@v4
  with:
      path: |
          .moon/cache/hashes
          .moon/cache/outputs
          .moon/cache/blobs
          .moon/cache/manifests
      key: moon-${{ runner.os }}-${{ github.ref_name }}-${{ github.sha }}
      restore-keys: |
          moon-${{ runner.os }}-${{ github.ref_name }}-
          moon-${{ runner.os }}-main-
```

- Persist only `hashes`, `outputs`, and on newer versions `blobs` and `manifests`. Never `states`, `locks` or `daemon`: they are machine-specific.
- The key must change every run (`github.sha`), otherwise the cache is saved once and never updated. Restore falls back to the latest entry of the branch, then of `main`.
- Limits you will hit: GitHub's per-repository cache quota (10 GB by default) and 7-day eviction, branch scoping (a PR can read its own and the base branch's caches, never a sibling PR's), and whole-archive restore cost that grows with the cache. `pipeline.cacheLifetime` keeps the directory from growing forever.
- Language caches are separate and usually worth more: pnpm store, `GOCACHE`/`GOMODCACHE`, `~/.cache/uv`, `~/.cargo` + `target/`. moon's cache skips whole tasks; those caches make the remaining tasks fast.
- This is good enough for small and medium repositories. It stops scaling when the archive gets large, when many parallel jobs each want the whole cache, or when developers should benefit from CI's results. That is what a remote cache is for.

## 6. Portability between machines

A cache shared between machines is only correct when equal hashes mean equal results.

| Leak                                         | Symptom                                                    | Fix                                                                       |
| -------------------------------------------- | ---------------------------------------------------------- | ------------------------------------------------------------------------- |
| Absolute paths in args (`$workspaceRoot/...`) | Never a remote hit between machines with different checkout paths | let the tool discover its config, or use a relative path in the command |
| Native output, toolchain does not hash platform (Node, Python) | macOS binary hydrated on Linux, `exec format error` | `fingerprint` check over platform (`node -p "process.platform+process.arch"`), or `cache: local` |
| Unpinned tool from `PATH`                    | Different lint results for the same hash                    | pin in `.prototools` (implicit input) or the language manifest            |
| Env var baked into output but not an input   | staging config shipped to production                        | `$VAR` input (or family `$VITE_*`)                                       |
| Generated file git-ignored and read as input | Change never invalidates                                    | depend on the generator task with `cacheStrategy: outputs`               |
| `unixShell` derived from `$SHELL`            | Different behaviour per developer                           | pin `unixShell`/`windowsShell` in `.moon/tasks/all.yml`                  |
| Go binary embeds VCS info                    | Cached binary reports an old commit                          | `-buildvcs=false`; inject release versions explicitly                     |
| Timestamps in outputs                        | Byte-different artifacts, `cacheStrategy: outputs` misses   | `SOURCE_DATE_EPOCH`, tool flags for reproducible builds                   |

## 7. Remote cache

### How it works

moon speaks the Bazel Remote Execution API v2 (action cache + content-addressable storage, SHA-256) over gRPC, or Bazel's simpler HTTP cache protocol (`api: http`, v1.32+). It uploads a task's outputs and logs under its hash, and downloads them on a hit. It does not upload source code. Since v2.4 a remote hit also warms the local cache, and a blob missing in one layer is fetched from the other.

### Options

| Server                                     | Notes                                                                                           |
| ------------------------------------------ | ----------------------------------------------------------------------------------------------- |
| [bazel-remote](https://github.com/buchgr/bazel-remote) | Single binary, disk LRU, optional S3/GCS/Azure backend. The documented self-hosted option. No bearer-token auth: put it behind a proxy that checks tokens, or use mTLS. |
| Depot Cache                                | Hosted, `grpcs://cache.depot.dev`, token in `DEPOT_TOKEN`. No servers to run; pay per usage.   |
| Other REAPI servers (BuildBuddy, NativeLink, Buildbarn, EngFlow) | Should work if they offer AC + CAS + SHA-256 over gRPC; moon documents only bazel-remote and Depot, so test before committing. |

```yaml
# .moon/workspace.yml
remote:
    host: grpcs://cache.internal.example.com:443
    auth:
        token: MOON_REMOTE_TOKEN          # name of the env var, not the token
    cache:
        instanceName: my-repo             # partition per repository
        compression: zstd                 # gRPC only; server must use the same storage mode
        localReadOnly: true               # developers download, only CI uploads (v1.40+)
        verifyIntegrity: false            # true costs CPU, catches corrupt blobs
        retryCount: 3                     # HTTP API only (v2.6)
```

- If `auth.token` names a variable that is not set, remote caching is silently disabled. That is a feature for fork PRs (no secrets, no remote) and a trap for a misconfigured CI job (everything quietly runs cold). Watch the hit rate.
- `host` can come from `MOON_REMOTE_HOST`, so the same `workspace.yml` works with and without the remote, and changing the host does not invalidate every hash.
- TLS/mTLS support is documented as rudimentary. Prefer a TLS-terminating proxy in front of the cache with a bearer token, or a private network.
- Debug with `MOON_DEBUG_REMOTE=true moon run <target> --log debug`. Task status `cached-from-remote` (also in OpenTelemetry metrics) tells you hits came from the remote.

### Who may write

The cache is shared mutable state: whoever can write can make everyone else replay their artifact.

| Writer                          | Policy                                                                               |
| ------------------------------- | ------------------------------------------------------------------------------------ |
| CI on protected branches (main, release) | read-write                                                                  |
| CI on internal PRs              | read-only (`MOON_CACHE=read`), or write to a separate `instanceName`                 |
| CI on fork PRs                  | no remote (secrets are not exposed, so the token is missing)                         |
| Developers                      | read-only (`localReadOnly: true`), token with read scope if the server supports it   |

Reasons: a developer's uncommitted environment (different tool version not captured in the hash, local patches in `node_modules`) produces artifacts that CI would then ship; a malicious PR could upload a poisoned artifact for a hash main will compute later. Artifacts are not encrypted by moon; encryption at rest is the storage provider's job, and outputs may contain secrets that were baked into bundles.

### When the remote cache does not pay off

- Tasks that take less time than a round trip (fast linters on small projects). Use `cache: local` for them.
- Huge outputs that change every commit (container tarballs, full `dist` of a monolith). You pay upload and storage for artifacts nobody reuses. Cache the inputs to that step instead, or let the registry be the cache.
- Repositories where `moon ci` already runs only a handful of affected tasks per PR. moon's own guidance: affected-only CI is the starting point; add a remote cache when CI time or cost becomes the bottleneck.

### Sizing and cost

- Size the cache to hold at least a few days of main-branch artifacts across all platforms you build on; LRU eviction handles the rest. Watch evictions: a cache that evicts today's artifacts has a hit rate near zero.
- One `instanceName` per repository; separate instances for experimental branches if they generate a lot of garbage.
- The bandwidth bill is usually larger than the storage bill. Co-locate the cache with CI runners (same region or same VPC).

## 8. Decision guide

| Situation                                                     | Recommendation                                                            |
| ------------------------------------------------------------- | ------------------------------------------------------------------------- |
| Small repository, CI under 10 minutes                         | local cache + `moon ci` affected runs; persist `.moon/cache` in CI if cheap |
| Medium repository, CI time dominated by a few builds          | remote cache for those builds (`cache: remote`/`true`), `local` for fast checks |
| Large monorepo, many parallel jobs, developers waiting on builds | remote cache with CI-only writes, `localReadOnly` for developers, sharded `moon ci` |
| Mixed macOS/Linux/Windows team with native outputs            | platform fingerprints on native tasks, or CI-only cache for those tasks   |
| Regulated environment                                         | self-hosted cache inside the network, mTLS or proxy auth, retention policy, audit of who holds write tokens |
