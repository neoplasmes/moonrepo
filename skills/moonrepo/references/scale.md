# moon at scale: scenarios and trade-offs

Each scenario: the situation, what to do, and what it costs. Links point to the detailed references.

## 1. Hundreds of projects, dozens of teams

**Situation.** 300+ projects, many teams, people unfamiliar with moon editing configs.

**Do.**
- Make task vocabulary and file-group names a written contract (`AGENTS.md`/`ARCHITECTURE.md`) and check it in CI: a small script over `moon query tasks --json` that rejects unknown task names, cached tasks without inputs, catch-all globs and `inputs: []` on tasks meant for `--affected`.
- Put 90% of task definitions in `.moon/tasks/*.yml`, selected by toolchain, language, tag or layer. A project `moon.yml` should mostly be file groups, `dependsOn` and a couple of project tasks.
- Turn on `constraints.enforceLayerRelationships` and `tagRelationships` early; generate `CODEOWNERS` from `owners`.
- Use an explicit project map or `globFormat: source-path` to avoid id collisions; consider the daemon (v2.2+, unstable) once graph building is noticeable in every command.

**Costs.** Inheritance is powerful and opaque: someone editing `.moon/tasks/node.yml` changes 200 projects and invalidates their caches. Require review from a platform owner for `.moon/**` (CODEOWNERS), and teach `moon task <target> --json` as the first debugging step.

## 2. CI cost and duration

**Situation.** CI takes 40 minutes; most PRs touch one service.

**Do, in order of payoff.**
1. `moon ci` with full-history checkout: affected-only runs usually cut most of the time ([ci.md](ci.md)).
2. Fix cache keys: precise inputs, no absolute paths, pinned tools ([caching.md](caching.md)). Measure hit rate before adding infrastructure.
3. Remote cache, written only from main ([caching.md](caching.md#7-remote-cache)). Main keeps it warm; PRs mostly download.
4. Sharding (`--job/--job-total` or execution plans) once one runner is CPU-bound.
5. `priority: high` on long tasks on the critical path; `cache: local` for tasks faster than a remote round trip.

**Costs.** `--downstream direct` (the `moon ci` default) skips transitive dependents: a breaking change in a core library may only surface on main. Run `moon ci --downstream deep` on main and nightly, or for core libraries. Remote caches need ownership (storage, auth, eviction tuning) and become a production dependency of CI: when the cache is down, CI must still pass, just slower.

## 3. Trust boundaries around the remote cache

**Situation.** Open-source or many contributors; some PRs come from forks.

**Do.**
- Writes only from CI on protected branches. Internal PRs `MOON_CACHE=read`. Fork PRs have no token, so no remote at all.
- Developers read only (`remote.cache.localReadOnly: true`), with a read-scoped credential if the server supports it.
- A separate `instanceName` (or server) per trust zone: release builds never read artifacts written by less trusted pipelines.
- Release artifacts are built with `--cache off` or `--force` if your compliance story requires "built from source in this pipeline".

**Costs.** Read-only PRs do not populate the cache for each other; two PRs touching the same package both rebuild it. Usually acceptable; the alternative (PR writes) is a cache-poisoning vector.

## 4. Mixed OS fleet (macOS and Windows developers, Linux CI)

**Do.**
- Pin `unixShell` and `windowsShell` in `.moon/tasks/all.yml`; write multi-step logic in cross-platform scripts; `options.os` variants where a tool differs per OS.
- Enforce LF line endings with `.gitattributes` so hashes and scripts match across OSes.
- Native outputs: rely on toolchains that hash platform data (Go, Rust), add `fingerprint` checks elsewhere (Node native modules, Python wheels), or `cache: local`.
- Windows developers keep repositories inside WSL when they need Docker or Bash-heavy tooling; native Windows is supported by moon itself, not by every task you write. Decide and document which ([docker.md](docker.md#windows)).
- Run a Windows CI leg at least nightly if Windows is a supported dev platform, otherwise it rots.

**Costs.** Platform-specific hashes mean macOS developers never hit Linux-built artifacts for Go/Rust tasks; the remote cache helps them only if some macOS machine (a macOS CI leg, or developer writes) populates it.

## 5. Many deployable services and container images

**Do.**
- Per ecosystem: Go/Rust/static frontends build artifacts as cached moon tasks and copy them into minimal images; Node/Python multi-package services use `moon docker scaffold` or the ecosystem's own pruning ([docker.md](docker.md)).
- Image tasks have real inputs (Dockerfile + the groups they ship) so `moon ci` selects them only when affected. Tag images with the commit SHA; deploy affected services only.
- Pin moon inside images to the version in `.prototools`; add `.prototools` to `docker.scaffold.configsPhaseGlobs`; narrow `sourcesPhaseGlobs`.

**Costs.** `moon docker scaffold` copies every project's manifests into the dependency layer, so in a large workspace the install layer is invalidated by unrelated manifest changes; per-service `pnpm deploy`/uv layering can be more precise for big repositories. Building outside Docker ties the artifact to the CI runner's platform.

## 6. Merge queues and high commit rates

**Do.**
- Handle the `merge_group` event explicitly (`MOON_BASE`/`MOON_HEAD` from the event) and `push` with `MOON_BASE=${{ github.event.before }}` ([ci.md](ci.md#revisions)).
- Remote cache makes queue runs cheap: the PR already built the same hashes.
- Flaky tests: `retryCount` with OpenTelemetry's `flaky` label to find them; quarantine with `allowFailure` for advisory tasks, `expectFailure` (v2.6) for known-broken checks during migrations.

**Costs.** Retries hide flakiness unless someone watches the metric.

## 7. AI agents and many worktrees

**Situation.** Several agents work in parallel worktrees, each running `moon run` constantly.

**Do.**
- `experiments.casOutputsCache` + `cache.unstable_sharedWorktreeCache` (v2.5) so worktrees share outputs; or a remote cache with `localReadOnly`.
- The moon MCP server and the `debug-task` skill for agents; `--log trace` output is designed to be agent-readable since v2.4.
- Precise `inputs` matter more: agents touch many files, and catch-all globs turn every edit into full rebuilds.
- Pre-commit hooks and stop hooks run `--affected` sets of tasks with real inputs (see the `inputs: []` trap in [decomposition.md](decomposition.md#2-check-fix-and-the-check-proxy)).

**Costs.** Shared worktree cache is unstable; a corrupted shared CAS affects all worktrees (`cache.cas.verifyIntegrity` trades CPU for safety).

## 8. Corporate networks, air gaps, compliance

**Do.**
- Mirror moon and proto downloads (`moon.downloadUrl`, `manifestUrl`, proto `[settings.http]`, internal plugin registries); vendor WASM plugins as `file://` locators.
- `telemetry: false` if policy requires.
- Self-hosted remote cache inside the network, TLS-terminating proxy, retention policy; artifacts are not encrypted by moon.
- `PROTO_OFFLINE=1` on air-gapped runners with pre-baked tool caches.
- Pin every shared `extends` URL and template location to a commit or version.

**Costs.** Mirrors must be updated with every version bump; plan it into the upgrade procedure.

## 9. Upgrading moon across a big repository

**Do.**
- Pin moon in `.prototools`; upgrade through a PR that bumps it, runs the full graph (`moon check --all` or `moon ci` with a forced base), and reads the release notes' "behavior change" sections (examples: v2.3 dependency `cacheStrategy` defaults, v2.6 persistent tasks no longer run in CI by default).
- Upgrades usually invalidate caches (hash format, toolchain plugins). Schedule them when a cold CI run is acceptable, and warm the remote cache from main right after merging.
- `unstable_*` toolchains and experiments: read their changelog entries specifically.

**Costs.** One cold CI day per upgrade; plugin API changes for custom WASM plugins ([wasm-plugins.md](wasm-plugins.md)).

## 10. When moon is the wrong tool

- A single-language repository whose native build system already does incremental, cached, affected builds well (Gradle with a build cache, Bazel, Cargo workspaces alone, Nx fully adopted). Adding moon on top doubles the configuration.
- A repository where most "tasks" are deployments and infrastructure operations with side effects: moon's value is caching and affected detection; for those tasks you get neither.
- Teams unwilling to maintain inputs: moon with `**/*` everywhere is just a slower task runner.
