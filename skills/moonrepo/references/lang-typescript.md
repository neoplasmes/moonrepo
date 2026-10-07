# TypeScript and JavaScript

Docs: [guides/javascript/node-handbook](https://moonrepo.dev/docs/guides/javascript/node-handbook), [guides/javascript/bun-handbook](https://moonrepo.dev/docs/guides/javascript/bun-handbook), [guides/javascript/typescript-project-refs](https://moonrepo.dev/docs/guides/javascript/typescript-project-refs), `moon toolchain info javascript` / `node` / `typescript`.

Node, Bun or Deno; pnpm, npm, yarn or bun as the package manager. The examples use pnpm and Node; swap the executor (`pnpm exec`, `bunx`, `bun --bun`, `deno run`) to match the repository.

## Toolchains

```yaml
# .moon/toolchains.yml
javascript:
    packageManager: pnpm
    inferTasksFromScripts: false         # explicit tasks; see below
    syncProjectWorkspaceDependencies: true
    dependencyVersionFormat: workspace   # writes "workspace:*" for moon dependsOn edges
node: {}
pnpm: {}
typescript:
    syncProjectReferences: true          # tsconfig "references" from dependsOn
    routeOutDirToCache: false
```

| Setting                                         | Gain                                                        | Cost                                                                 |
| ----------------------------------------------- | ----------------------------------------------------------- | -------------------------------------------------------------------- |
| `inferTasksFromScripts`                         | zero-config migration from `package.json` scripts          | the real command hides behind a script; tasks get default `**/*` inputs |
| `syncProjectWorkspaceDependencies`              | `dependsOn` and `package.json` cannot drift                 | moon edits manifests during runs; CI may end up with a dirty tree    |
| `typescript.syncProjectReferences`              | `tsc --build` references stay correct                       | same: writes `tsconfig.json` files                                  |
| `rootPackageDependenciesOnly`                   | one-version policy, smaller lockfile churn                   | every dependency bump invalidates every Node task                    |
| `installDependencies` (default on)              | `pnpm install` runs automatically when the lockfile changes | an install inside `moon run` on every lockfile change; set `pipeline.installDependencies: false` if CI installs explicitly |

Rule of thumb: run sync features locally (`moon sync`), and in CI fail when they would change something (`git diff --exit-code` after `moon sync`), instead of letting CI rewrite files.

## What enters the hash

The Node toolchain adds the Node version and the resolved versions of the project's dependencies (from the lockfile with `hasher.optimization: accuracy`, the default). It does not add the OS or CPU. A JS bundle is platform-independent, so this is fine; a task whose output contains native binaries (`esbuild` binaries, `sharp`, Electron builds, `node_modules` themselves) needs a fingerprint check:

```yaml
checks:
    - check: fingerprint
      script: node -p "process.platform + '-' + process.arch"
      hash: stdout
```

Root files every Node task depends on go into `implicitInputs` of the ecosystem file: `/package.json`, `/pnpm-lock.yaml`, `/pnpm-workspace.yaml`, `/tsconfig.json` (or `tsconfig.base.json`), `/.npmrc`. Tool configs (`/eslint.config.js`, `/biome.json`, `/oxlint.config.ts`) go into `cfg-*` groups used only by that tool's tasks.

## Shared tasks

```yaml
# .moon/tasks/node.yml
inheritedBy:
    toolchains: [javascript]

implicitInputs:
    - /package.json
    - /pnpm-lock.yaml
    - /pnpm-workspace.yaml

fileGroups:
    src: []
    test-unit: []
    test-int: []
    cfg: []
    cfg-lint:
        - /eslint.config.js

taskOptions:
    runFromWorkspaceRoot: false

tasks:
    lint:
        command: pnpm exec eslint --max-warnings 0 .
        inputs: &lint
            - "@group(src)"
            - "@group(test-unit)"
            - "@group(test-int)"
            - "@group(cfg)"
            - "@group(cfg-lint)"

    lint-fix:
        command: pnpm exec eslint --fix .
        inputs: *lint

    fmt:
        command: pnpm exec prettier --write .
        inputs: *lint
        deps:
            - ~:lint-fix

    test-unit:
        command: pnpm exec vitest run --project unit
        inputs:
            - "@group(src)"
            - "@group(test-unit)"
            - "@group(cfg)"

    check:
        command: noop
        inputs:
            - "@group(src)"
            - "@group(test-unit)"
            - "@group(test-int)"
            - "@group(cfg)"
        deps:
            - ~:fmt
            - target: ~:typecheck
              optional: true        # plain JS projects have no typecheck
```

```yaml
# .moon/tasks/typescript.yml
inheritedBy:
    languages: [typescript]

tasks:
    typecheck:
        command: pnpm exec tsc --noEmit -p tsconfig.json
        inputs:
            - "@group(src)"
            - "@group(test-unit)"
            - "@group(test-int)"
            - "@group(cfg)"
            - /tsconfig.json
        options:
            runFromWorkspaceRoot: false
```

- Running from the project directory lets `pnpm exec` resolve the project's own binaries and nested config files; running from the root (`--dir`, `-p $projectSource`) is faster to reason about for single-root tools. Choose per tool, not per repository.
- `eslint .` from the project root lints only that project; pairing it with `options.affectedFiles: args` lints only changed files under `--affected`, but outside `--affected` moon passes `.`, so the command must accept `.`.
- Biome, oxlint and dprint are much faster than ESLint/Prettier; with fast tools you may skip caching lint entirely (`cache: local`) because a remote round trip costs more than the run.

## Builds and project references

```yaml
tasks:
    build:
        command: pnpm exec vite build
        inputs:
            - "@group(src)"
            - "@group(assets)"
            - "@group(cfg)"
            - $VITE_*                        # baked into the bundle
        outputs:
            - dist
        deps:
            - target: ^:build
              cacheStrategy: outputs
```

- `^:build` runs `build` in every dependency project that defines it and silently skips the others (verified), so define `build` only where a package ships compiled output. Source-only internal packages (consumed through TS path mapping or `exports` pointing at `.ts`) need no build and no `^:build` edge: the consumer's inputs cover them with `project://<id>?group=src`.
- With `tsc --build` and project references, `.tsbuildinfo` files are outputs. Commit to one model: either moon caches each package's `tsc` output (`outputs: [dist, tsconfig.tsbuildinfo]`), or one root `tsc --build` task owns type checking for everything. Mixing them double-builds.
- Frontend env vars (`VITE_*`, `NEXT_PUBLIC_*`) are compile-time constants. They must be inputs; otherwise a staging bundle can be hydrated in production.
- Next.js and other frameworks with their own caches (`.next/cache`): keep those out of `outputs`, and persist them separately in CI if needed.

## Dev servers and tests

```yaml
tasks:
    dev:
        command: pnpm exec vite
        preset: server
    start:
        command: node dist/server.js
        preset: server
        deps:
            - ~:build
    test-e2e:
        command: pnpm exec playwright test
        inputs:
            - "@group(e2e)"
            - "@group(cfg)"
        deps:
            - target: web:start
              type: wait               # v2.6
            - target: web:stop
              type: cleanup
        options:
            cache: false
```

- `preset: server` = persistent, not cached, streamed output, not in CI. Since v2.6 persistent tasks start as soon as their deps finish.
- Playwright, Cypress and browser downloads are not npm dependencies: pin them and install them in a `setup`-style task with `cache: false`, or cache `~/.cache/ms-playwright` in CI.

## Bun and Deno

- Bun: `bun: {}` toolchain plus `javascript.packageManager: bun`. `bun --bun <tool>` forces Bun instead of Node for a tool's shebang. Bun's lockfile `bun.lock` is the implicit input.
- Deno: `deno: {}`; `deno.json` tasks can be inferred with `inferTasksFromScripts`; the Deno toolchain hashes import maps.

## Profiling

`moon run --profile cpu <target>` and `--profile heap` record V8 profiles for Node-based tasks only (`.moon/cache/states/<project>/<task>/snapshot.cpuprofile`). Details: [observability.md](observability.md#task-profiling).

## Docker

pnpm workspaces have a canonical, verified recipe in [docker.md](docker.md#canonical-recipe-pnpm-workspace): `moon docker scaffold` for file lists, `pnpm fetch` on the lockfile, an offline filtered install, `moon run <app>:build --no-actions`, `pnpm deploy --prod`. Do not use `moon docker setup` (it runs an unfiltered `pnpm install`) or `moon docker prune` there. npm and yarn workspaces: read [guides/docker](https://moonrepo.dev/docs/guides/docker) and check what `moon docker setup` runs with `--log debug` before relying on it.