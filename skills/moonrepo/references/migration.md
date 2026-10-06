# Migrating to moon

Port the intent, not the form. Existing scripts tell you what has to happen; moon tasks should say it with precise inputs and outputs. Migrate in two passes:

1. **Wrap.** Each existing entry point becomes a task with `cache: false` (or `inferTasksFromScripts` for `package.json`). Nothing breaks, CI switches to `moon run`/`moon ci`, people learn targets.
2. **Tighten.** Task by task: call the tool directly, add file groups, `inputs`, `outputs`, `deps`, move shared tasks into `.moon/tasks/*.yml`, enable caching, verify hits (see [debugging.md](debugging.md)).

Skipping pass 2 leaves you with a slower Makefile. Both passes serve [MOONREPO FIRST](../SKILL.md#moonrepo-first): at the end moon is the only entry point, and old scripts and Make targets are deleted or reduced to one-line `moon run` wrappers. Skipping pass 1 means a big-bang change nobody can review.

## Sources and what to do with them

| Source                                   | Approach                                                                                   |
| ---------------------------------------- | ------------------------------------------------------------------------------------------ |
| `package.json` scripts                   | pass 1: `javascript.inferTasksFromScripts: true`; pass 2: explicit tasks calling the tools, scripts removed |
| Makefile / justfile / Taskfile           | one moon task per target that matters; prerequisites become `deps`; variables become `env` or `args` |
| Nx                                       | `moon ext migrate-nx` (experimental) converts `nx.json`/`project.json`; `namedInputs` become file groups, `targetDefaults` become `.moon/tasks`. Expect manual fixes: executors become plain commands |
| Turborepo                                | `moon ext migrate-turborepo` converts `turbo.json` tasks; tasks still call package scripts until you tighten them |
| Bash scripts in CI YAML                  | each step becomes a task; CI calls `moon ci`                                                |
| Per-language tools (`go generate`, `cargo xtask`, `tox`/`nox`) | keep the tool, wrap its entry points as tasks with precise inputs           |

## Translation table

| Before                                                        | After                                                                              |
| ------------------------------------------------------------- | ---------------------------------------------------------------------------------- |
| `command: pnpm lint` (a package script)                       | the tool itself (`pnpm exec eslint .`); the script removed                         |
| formatter task with `cache: false`                            | inherited, cached `fmt` with real inputs                                           |
| one `lint` running two linters with `&&`                      | `lint` and `lint-css`, each with its `-fix` twin, each cached                      |
| `test`                                                        | `test-unit`, `test-int` or `test-e2e`                                              |
| `e2e`, `preview`, `tidy`                                      | `test-e2e`, `start`, `lock`                                                        |
| `toolchain: system` on every task                             | tools pinned in `.prototools`; default toolchain                                   |
| `script:` with `set -a; source .env`                          | `options.envFile`                                                                   |
| script that starts Compose, runs tests, stops Compose          | `compose-up` dep + test task + `compose-down` (v2.6: `wait`/`cleanup` deps)        |
| `../../../.golangci.yml` in a command                         | config discovery by the tool, or a workspace-relative input plus a relative path   |
| `inputs` repeated in every task                               | file groups plus a YAML anchor or `extends`                                        |
| per-project copies of `typecheck`, `lint`, `lint-fix`         | inherited from `.moon/tasks/*.yml`                                                 |
| `deps: [^:build]` everywhere                                  | only where a dependency ships build output, with `cacheStrategy: outputs`          |
| `"{src,server}/**/*"`, `**/*` inputs                          | role-based groups with explicit extensions                                         |
| `bash -c 'cd x && a && b && c'`                               | separate tasks with `deps`, or one script in `tools/`                              |

## Example 1: React SSR app (from a real repository)

Before:

```yaml
fileGroups:
  src: ["{src,server,vite}/**/*", "!**/*.test.{js,ts,tsx}"]
  test: ["{src,server}/*.test.{js,ts,tsx}", "test/**/*"]
  cfg: ["package.json", "tsconfig.json", "*.config.{js,ts}"]
tasks:
  build:        { command: "pnpm build", deps: ["^:build"], inputs: ["@group(src)", "@group(cfg)", "index.html"], outputs: ["dist"] }
  build-client: { command: "pnpm build:client", inputs: ["src/**/*", "vite.config.ts", "index.html"], outputs: ["dist/client"] }
  build-server: { command: "pnpm build:server", deps: ["^:build"], inputs: ["server/**/*", "src/**/*", "tsup.config.ts"], outputs: ["dist/server"] }
  fmt:          { command: "pnpm prettier", options: { cache: false } }
  lint:         { command: "pnpm lint" }
  test:         { command: "pnpm test" }
```

Problems:

- `{src,server,vite}/**/*` hashes every file, including styles, HTML and fixtures; `!**/*.test.*` misses test helpers.
- `{src,server}/*.test.*` has no `**`, so nested tests never invalidate `test`.
- `*.config.{js,ts}` picks up any config that appears later, including copies of root-owned ones.
- `build` repeats both sub-builds; the sub-builds are shortcuts that drift from it.
- Every command goes through `pnpm <script>`: the real command is invisible to moon and to the hash.

After:

```yaml
language: typescript
layer: application
tags: [css]
dependsOn: [primitive-server]

fileGroups:
    src:
        - "{src,server}/**/*.{ts,tsx}"
        - "!src/**/*.{test,test.int}.{ts,tsx}"
        - "!server/**/*.test.ts"
        - "!src/test/**"
    test-unit:
        - src/**/*.test.{ts,tsx}
        - server/**/*.test.ts
        - src/test/**/*.ts
    css:
        - src/**/*.css
    cfg:
        - "{package,tsconfig}.json"
        - "{vite,vitest}.config.ts"
        - index.html

tasks:
    build-client:
        command: pnpm exec vite build
        inputs: ["@group(src)", "@group(css)", "@group(cfg)"]
        outputs: [dist/client]
        options: { internal: true }

    build-server:
        command: pnpm exec vite build --ssr src/entry.server.tsx --outDir dist/server
        inputs: ["@group(src)", "@group(cfg)"]
        outputs: [dist/server]
        deps:
            - target: primitive-server:build
              cacheStrategy: outputs
        options: { internal: true }

    build:
        command: noop
        inputs: ["@group(src)", "@group(css)", "@group(cfg)"]
        deps: [client:build-client, client:build-server]

    dev:
        command: node --watch server/server.ts
        preset: server
        options: { envFile: .env }

    start:
        command: node dist/server/server.js
        preset: server
        deps: [client:build]
        options: { envFile: .env }
```

`fmt`, `lint`, `lint-fix`, `lint-css`, `typecheck` and `check` arrive by inheritance (TypeScript language, `css` tag).

## Example 2: Go service (`apps/go/auth`)

Before (abridged):

```yaml
fileGroups:
  src:
    - "{cmd,internal}/**/*.go"
    - "!**/*_test.go"
  test:
    - "**/*_test.go"

tasks:
  dev:
    script: "set -a; source .env; set +a; go run ./cmd/auth/main.go"
    toolchain: system
  tidy:
    command: "go mod tidy -v"
  fmt:
    command: "golangci-lint fmt --config ../../../.golangci.yaml ./..."
    options: { cache: false }
  lint:
    command: "golangci-lint run --config ../../../.golangci.yaml ./..."
  test:
    script: |
      docker compose -f docker-compose.test.yaml up -d ...
      go test -tags=integration -count=1 ./test/integration/... || RC=$?
      docker compose -f docker-compose.test.yaml down ...
  swagger:
    script: "swag init -d cmd/auth,internal/... -o docs ..."
    outputs: ["docs"]
```

After: the project keeps only what is specific to it. Formatting, linting, `typecheck`, `test-unit`, `test-int` and `check` come from `.moon/tasks/go.yml` (see [lang-go.md](lang-go.md)).

```yaml
language: go
layer: application

fileGroups:
    src:
        - "{cmd,internal}/**/*.go"
        - "!**/*_test.go"
    test-unit:
        - "{cmd,internal}/**/*_test.go"
        - "!**/*_int_test.go"
    test-int:
        - "**/*_int_test.go"
        - test/integration/**/*.go
    cfg:
        - go.{mod,sum}

tasks:
    lock:
        command: go mod tidy
        options:
            cache: false

    swagger:
        command:
            swag init --dir cmd/auth,internal/adapters/driving/http --generalInfo main.go
            --output docs --v3.1 --parseInternal --outputTypes json
        inputs:
            - "@group(src)"
            - "@group(cfg)"
        outputs:
            - docs/swagger.json

    test-int:
        deps:
            - auth:compose-up
        options:
            mutex: auth-test-stack

    dev:
        command: go run ./cmd/auth
        preset: server
        options:
            envFile: .env

    build:
        command: go build -o bin/auth ./cmd/auth
        inputs:
            - "@group(src)"
            - "@group(cfg)"
        outputs:
            - bin/auth
```

What changed and why:

- `fmt` became an inherited, cached task instead of a per-project `cache: false` copy.
- `tidy` became `lock`: it refreshes `go.sum`, which is this ecosystem's lockfile.
- The Bash test script that started and stopped Compose became an inherited `test-int` with a dep on `compose-up` and a `mutex`. Starting and stopping a stack are their own tasks, so a failed test never hides a teardown problem; on v2.6 the stop becomes a `cleanup` dependency.
- `swagger` is a generator: precise inputs, a file output, cached.

## After migrating

- Remove the old entry points (scripts, Make targets) or make them one-line wrappers around `moon run`, so there is one source of truth.
- Update CI to `moon ci` and compare durations and coverage for a week before deleting the old pipeline.
- Write the repository's conventions (task names, layers, where shared tasks live) into `AGENTS.md`/`ARCHITECTURE.md`; this skill's defaults are a starting point, not the contract.
