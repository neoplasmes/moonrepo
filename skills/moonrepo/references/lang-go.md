# Go

Go has strong opinions about layout and its own build and test cache. moon adds project-level orchestration, affected detection and cross-machine caching on top; the trick is to not fight Go's cache.

## Toolchain

```yaml
# .moon/toolchains.yml
go:
    workspaces: true                 # read go.work / go.work.sum when present
    inferRelationships: true         # dependsOn from imports (go list --deps)
    inferRelationshipsFromTests: false
    tidyOnChange: false              # keep `go mod tidy` an explicit `lock` task
```

- The Go toolchain adds the Go version **and the host os/arch/libc** to every task hash (verified in a hash manifest: `{"toolchain":"go","version":"1.25.1","contents":[{"os":"linux","arch":"x64","libc":"gnu"}]}`). A binary built on macOS arm64 is therefore never replayed on Linux CI. Tasks that do not run under the Go toolchain (a project without `language: go`, or `toolchains: [system]`) lose this protection.
- `inferRelationships` keeps `dependsOn` in sync with imports. In a single huge module it runs `go list --deps` over everything; restrict it with `inferRelationshipsPackages` if graph building gets slow.
- `bins` installs tools with `go install`; prefer pinning tools elsewhere (below), because `bins` versions are not part of task hashes unless you add them.

## Layout: one module or many

| Layout                                                 | Gain                                                        | Cost                                                                            |
| ------------------------------------------------------ | ----------------------------------------------------------- | ------------------------------------------------------------------------------- |
| One module per project + `go.work`                     | isolated dependency bumps; affected detection per service  | many `go.mod` files to keep consistent; `go.work` must be committed or every tool must know about it |
| One root `go.mod`, projects are directories            | one dependency set, trivial refactors across services       | any `go.sum` change invalidates every Go task; `go test ./...` scope must be narrowed per project |

With one root module the shared files are implicit inputs and commands are scoped to the project directory:

```yaml
# .moon/tasks/go.yml (single root module)
inheritedBy:
    languages: [go]
implicitInputs:
    - /go.mod
    - /go.sum
taskOptions:
    runFromWorkspaceRoot: true
tasks:
    test-unit:
        command: go test ./$projectSource/...
```

With modules per project, run tasks inside the project (`runFromWorkspaceRoot: false`) and add `/go.work` and `/go.work.sum` as optional implicit inputs.

## Shared tasks (module per project)

```yaml
# .moon/tasks/go.yml
inheritedBy:
    languages: [go]

implicitInputs:
    - file: /go.work
      optional: true
    - file: /go.work.sum
      optional: true

fileGroups:
    test-unit: []
    test-int: []
    cfg-lint-go:
        - /.golangci.yml

taskOptions:
    runFromWorkspaceRoot: false

tasks:
    fmt:
        command: golangci-lint fmt ./...
        inputs: &go-all
            - "@group(src)"
            - "@group(test-unit)"
            - "@group(test-int)"
            - "@group(cfg)"
            - "@group(cfg-lint-go)"
        deps:
            - ~:lint-fix

    lint:
        command: golangci-lint run ./...
        inputs: *go-all

    lint-fix:
        extends: lint
        args: --fix

    typecheck:
        command: go vet ./...
        inputs: *go-all

    test-unit:
        command: go test ./...
        inputs:
            - "@group(src)"
            - "@group(test-unit)"
            - "@group(cfg)"

    test-int:
        command: go test -count=1 -tags integration -run '^TestInt' ./...
        inputs:
            - "@group(src)"
            - "@group(test-int)"
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
            - ~:typecheck
```

```yaml
# apps/<service>/moon.yml
language: go
layer: application

fileGroups:
    src:
        - "{cmd,internal,pkg}/**/*.go"
        - "!**/*_test.go"
    test-unit:
        - "{cmd,internal,pkg}/**/*_test.go"
        - "!**/*_int_test.go"
    test-int:
        - "**/*_int_test.go"
        - testdata/**/*
    cfg:
        - go.{mod,sum}
```

golangci-lint finds `/.golangci.yml` by walking up from the module directory, so no `--config` flag is needed. Avoid `--config $workspaceRoot/...`: tokens are expanded before hashing (verified: the manifest stores `/tmp/moonlab/.golangci.yml`), so every machine with a different checkout path gets a different hash and never hits the remote cache. When a tool cannot discover its config, pass a path relative to the working directory inside the command; `..` is acceptable there, never in `inputs`.

### Working with Go's own caches

- `go test` caches per package in `GOCACHE`. moon decides per project; Go then skips unchanged packages inside that project. Do **not** add `-count=1` to unit tests: you would throw away the finer-grained cache.
- Integration tests touch databases and networks Go cannot see. Use `-count=1` there, and a `mutex` or a v2.6 `wait`/`cleanup` stack (see [decomposition.md](decomposition.md)).
- moon's remote cache does not carry `GOCACHE` or `GOMODCACHE`. In CI persist them separately (`actions/setup-go` with `cache: true`, keyed on `go.sum`). Remote cache and Go cache are complementary: moon skips whole projects, Go makes misses cheap.
- Integration tests keep Go's `_test.go` suffix: `*_int_test.go` with `//go:build integration`. A `.int/` or `_int/` folder does not work: `./...` skips directories starting with `.` or `_`.

## Pinning Go tools

| Method                                         | Notes                                                                                  |
| ---------------------------------------------- | -------------------------------------------------------------------------------------- |
| `.prototools` + a proto plugin                 | one place for every tool; `/.prototools` is already an implicit input. golangci-lint needs a TOML or WASM proto plugin ([wasm-plugins.md](wasm-plugins.md)). |
| `tool` directives in `go.mod` (Go 1.24+), `go tool <name>` | version lives in `go.mod`/`go.sum`, already hashed. Good for `stringer`, `sqlc`, `oapi-codegen`, `mockgen`. golangci-lint discourages this because its dependencies leak into your module. |
| toolchain `bins`                               | simplest, but versions are invisible to task hashes                                    |

## Builds and binaries

```yaml
tasks:
    build:
        command: go build -trimpath -buildvcs=false -ldflags "-s -w" -o bin/<service> ./cmd/<service>
        inputs:
            - "@group(src)"
            - "@group(cfg)"
        outputs:
            - bin/<service>
        env:
            CGO_ENABLED: "0"
            GOOS: linux
            GOARCH: amd64
```

- `-trimpath` removes absolute paths from the binary, so the output does not depend on the checkout location. `-buildvcs=false` stops Go from embedding the commit: otherwise a cached binary from commit A is replayed at commit B and reports the wrong revision.
- Need the version in the binary? Release builds pass it with `-ldflags -X main.version=$VERSION` and list `$VERSION` as an input (or run uncached). Development builds should not, or every commit is a cache miss.
- An explicit `GOOS`/`GOARCH` makes the artifact identical on every machine, so a deployable binary can be built once and shared through the remote cache. A binary meant to run on the developer's machine should not pin them.
- With `CGO_ENABLED=1` the result depends on the C toolchain and libc too. Keep cgo builds in a container or add a fingerprint check (`cc --version`).

## Docker

For Go, `moon docker scaffold` brings little: dependencies are one `go mod download`, and the result is a static binary. The usual pattern is a `build` task as above and a thin image:

```dockerfile
FROM gcr.io/distroless/static-debian12
COPY apps/<service>/bin/<service> /app
ENTRYPOINT ["/app"]
```

with an `image` task that depends on `~:build`. The binary comes from the moon cache, the image build takes seconds. See [docker.md](docker.md) for the trade-offs against building inside Docker.

## Generated code

`sqlc`, `oapi-codegen`, `protoc`, `swag`: a `gen-*` (or project-specific) task with precise inputs (`queries/**/*.sql`, `openapi.yaml`) and the generated files as `outputs`. Consumers depend on it with `cacheStrategy: outputs`. Decide whether generated files are committed:

- Committed: reviewers see them, `go build` works without moon; add a CI check that regenerating changes nothing.
- Not committed: they must be git-ignored, so they are never inputs; every consumer must depend on the generator task.
