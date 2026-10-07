# CI with moon

Docs: [guides/ci](https://moonrepo.dev/docs/guides/ci), [commands/ci](https://moonrepo.dev/docs/commands/ci), [commands/exec](https://moonrepo.dev/docs/commands/exec), [guides/exec-plan](https://moonrepo.dev/docs/guides/exec-plan), [concepts/affected](https://moonrepo.dev/docs/concepts/affected), [version matrices](https://moonrepo.dev/docs/guides/open-source).

CI follows [MOONREPO FIRST](../SKILL.md#moonrepo-first): the pipeline calls `moon ci` or `moon run`, never `pnpm test`, `go test`, `make` or the tools themselves. A CI step that repeats a task's command is a second source of truth and bypasses affected detection and the cache.

## What `moon ci` does

`moon ci [targets...]` = `moon exec` with `--affected --ci --on-failure=continue --summary=detailed --upstream=deep --downstream=direct`:

1. Computes changed files between a base and a head revision.
2. Selects tasks whose inputs match (all tasks, or only the targets you pass), filtered by `runInCI`.
3. Adds their dependencies (deep) and the direct dependents' tasks.
4. Installs toolchains and dependencies, runs everything in parallel, continues past failures, prints a summary, writes `.moon/cache/ciReport.json`.

`--downstream direct` is a deliberate trade-off: a change in `lib` runs tasks of projects that depend on `lib` directly, not of projects that depend on them. If a transitive break matters (lib → sdk → app), run `moon ci --downstream deep` on main or nightly, or for specific targets.

## `runInCI`

| Value                 | Meaning                                                                  |
| --------------------- | ------------------------------------------------------------------------ |
| `true` / `affected`   | default: run when affected                                               |
| `always`              | run on every CI run, affected or not (license scans, smoke checks)       |
| `false`               | never in CI (`dev`, interactive helpers, local-only fix tasks)           |
| `only`                | only in CI, never locally                                                |
| `skip`                | not in CI, but keeps dependency edges valid for tasks that do run there  |

- A task that runs in CI cannot depend on one that does not (`run_in_ci_mismatch`). Fix with `runInCI: true` or `skip` on the dependency.
- Tasks named `dev`, `start`, `serve` and persistent tasks are not run in CI by default; since v2.6 persistent tasks never run unless `runInCI` is enabled, and then only for tasks that depend on them.
- Fix tasks (`lint-fix`, `fmt`) mutate files. In CI, run read-only variants, or run the fix and fail on `git diff --exit-code`. Pick one and document it.

## Revisions

- **Full history is required.** `actions/checkout` with `fetch-depth: 0` and `filter: blob:none` (full commit graph, file contents on demand). GitLab: `GIT_DEPTH: 0`. A shallow clone makes affected detection wrong or fail.
- moon detects base and head from the CI provider. Override with `--base/--head` or `MOON_BASE`/`MOON_HEAD` (highest precedence).
- **Push to the default branch compares with `HEAD~1`.** A push of several commits (rebase-merge, direct pushes) checks only the last one. Set `MOON_BASE=${{ github.event.before }}` on push events.
- **Merge queues** (`merge_group` event) check out a temporary branch. Set `MOON_BASE=${{ github.event.merge_group.base_sha }}` and `MOON_HEAD=${{ github.event.merge_group.head_sha }}`.
- First push of a new branch: `github.event.before` is all zeros; fall back to the default branch.

## GitHub Actions

```yaml
name: CI
on:
    pull_request:
    push:
        branches: [main]
    merge_group:

concurrency:
    group: ci-${{ github.ref }}
    cancel-in-progress: ${{ github.event_name == 'pull_request' }}

jobs:
    ci:
        runs-on: ubuntu-latest
        timeout-minutes: 30
        env:
            MOON_REMOTE_TOKEN: ${{ secrets.MOON_REMOTE_TOKEN }}
            # PRs read the remote cache, main writes it
            MOON_CACHE: ${{ github.event_name == 'push' && 'read-write' || 'read' }}
        steps:
            - uses: actions/checkout@v4
              with:
                  fetch-depth: 0
                  filter: blob:none

            - uses: moonrepo/setup-toolchain@v0
              with:
                  auto-install: true          # proto install from .prototools; caches ~/.proto

            - name: Set base for pushes and merge queues
              run: |
                  if [ "${{ github.event_name }}" = "push" ] && [ "${{ github.event.before }}" != "0000000000000000000000000000000000000000" ]; then
                    echo "MOON_BASE=${{ github.event.before }}" >> "$GITHUB_ENV"
                  fi
                  if [ "${{ github.event_name }}" = "merge_group" ]; then
                    echo "MOON_BASE=${{ github.event.merge_group.base_sha }}" >> "$GITHUB_ENV"
                    echo "MOON_HEAD=${{ github.event.merge_group.head_sha }}" >> "$GITHUB_ENV"
                  fi

            - run: moon ci --output-style buffer-only-failure --summary minimal   # --output-style: v2.6+

            - uses: moonrepo/run-report-action@v1
              if: success() || failure()
              with:
                  access-token: ${{ secrets.GITHUB_TOKEN }}
```

- `moonrepo/setup-toolchain` installs proto and moon (version from `.prototools`) and caches `~/.proto` keyed on `.prototools` and `.moon/toolchain*.yml`. Language package caches (pnpm store, Go, uv, Cargo) are separate steps.
- `moonrepo/run-report-action` posts the run summary to the PR. `.moon/cache/ciReport.json` can be uploaded as an artifact for later analysis.
- Branch protection with affected-only jobs: a required check that is skipped counts as pending. Make one aggregating job (`needs: [ci]`, `if: always()`) the required check, or keep a single `moon ci` job.

## GitLab CI

```yaml
ci:
    image: ubuntu:24.04
    variables:
        GIT_DEPTH: "0"
        MOON_BASE: $CI_MERGE_REQUEST_DIFF_BASE_SHA
    before_script:
        - bash <(curl -fsSL https://moonrepo.dev/install/proto.sh) --yes
        - export PATH="$HOME/.proto/bin:$HOME/.proto/shims:$PATH"
        - proto install
    script:
        - moon ci
    cache:
        key: proto-$CI_COMMIT_REF_SLUG
        paths: [.proto-cache/]
    rules:
        - if: $CI_PIPELINE_SOURCE == "merge_request_event"
        - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH
```

Point `PROTO_HOME` at a project-relative directory if you want GitLab to cache it (GitLab caches only paths inside the project).

## Splitting the work

| Technique                                   | Use when                                                               | Trade-off                                                   |
| ------------------------------------------- | ---------------------------------------------------------------------- | ----------------------------------------------------------- |
| One `moon ci` job                           | default; fits most repositories                                        | one runner's CPU is the limit                               |
| `moon ci :lint :typecheck` / `moon ci :build :test-unit` in separate jobs | different runner sizes or permissions per kind of task | dependencies are recomputed per job; without a remote cache a shared `build` runs in each |
| `moon ci --job $i --job-total $n` (matrix)  | many independent affected tasks                                        | moon splits by target, not by duration; heavy tasks can land in one shard. Needs a remote cache or each shard rebuilds shared deps |
| Execution plan with partitioned `targets.jobs` (v2.1+, `moon exec --plan plan.json --job $i`) | you know which groups are heavy and want stable, reviewed partitions | the partition list must be maintained |
| Per-OS matrix (`ubuntu`, `windows`, `macos`) | the product ships on several platforms                                | 3x cost; run the non-Linux legs on main or nightly, not every PR |

Execution plans keep long CLI flag sets in a reviewed JSON file:

```json
{
    "affected": { "base": "origin/main", "source": "remote" },
    "pipeline": { "ci": true, "onFailure": "continue", "jobTotal": 3 },
    "graph": { "upstream": "deep", "downstream": "direct" },
    "targets": { "jobs": [[":build"], [":test-unit"], [":lint", ":typecheck"]] }
}
```

## Toolchain version matrices

`MOON_<TOOLCHAIN>_VERSION` overrides a toolchain version for one run (`MOON_NODE_VERSION=22`, `MOON_GO_VERSION=1.24.5`). Libraries that support several runtime versions run a matrix with it; applications pin one version and do not.

## Persistent services in CI (v2.6)

```yaml
tasks:
    serve:
        command: pnpm exec vite preview
        preset: server
        options:
            runInCI: true          # only started for tasks that depend on it
    test-e2e:
        deps:
            - target: ~:serve
              type: wait
```

Before v2.6 start services with CI steps (`docker compose up -d --wait`) and tear them down with `if: always()`.

## Hygiene checks worth adding

- Config drift: `moon sync` then `git diff --exit-code` (TypeScript references, `package.json` workspace deps, CODEOWNERS, hooks).
- Generated files: run the generator, then `git diff --exit-code`.
- Lockfiles: `pnpm install --frozen-lockfile`, `uv sync --locked`, `go mod tidy -diff` (Go 1.23+), `cargo metadata --locked`.
- `moon query projects` and `moon task <target> --json` in a debug step when a pipeline behaves differently from local runs. See [debugging.md](debugging.md).

## Observability

Webhooks (`MOON_WEBHOOK_URL`), OpenTelemetry (`MOON_OTEL`, `OTEL_EXPORTER_OTLP_ENDPOINT`) and run reports are covered in [observability.md](observability.md). Cache hit rate per target is the metric to watch after enabling a remote cache.
