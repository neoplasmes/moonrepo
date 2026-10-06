# Docker

moon's Docker support is a set of file-shuffling commands that make Dockerfiles in a monorepo stop growing with the number of projects. It is optional, and for some ecosystems the wrong tool. This file says what the commands really do (verified on moon 2.5.5 in a pnpm + Go + uv workspace), when they pay off, and what breaks on Windows.

## What the commands do

| Command                         | Where it runs              | Effect                                                                                        |
| ------------------------------- | -------------------------- | --------------------------------------------------------------------------------------------- |
| `moon docker scaffold <ids...>` | repository (host or a build stage) | writes `.moon/docker/configs` and `.moon/docker/sources`                               |
| `moon docker setup`             | inside the image           | installs toolchains (via proto) and dependencies for the focused projects                     |
| `moon docker prune`             | inside the image           | deletes vendor directories (`node_modules`, `.venv`, `target`) and reinstalls production deps for focused projects |
| `moon docker file <id>`         | repository                 | generates a multi-stage `Dockerfile` for one project                                          |

Verified details the docs gloss over:

- The skeleton folders are `.moon/docker/configs` and `.moon/docker/sources`. (Parts of the Docker guide still say `workspace`; the generated Dockerfile and the CLI use `configs`.)
- `configs` contains the manifests of **every** project in the workspace (`package.json`, `go.mod`, `pyproject.toml`), the root lockfiles and `.moon/*.yml`, not only those of the focused project. Any manifest change anywhere invalidates the dependency layer of every image.
- `.prototools` is **not** copied into `configs` by default. Without it `moon docker setup` cannot see your pinned versions. Add it explicitly:

  ```yaml
  # .moon/workspace.yml
  docker:
      scaffold:
          configsPhaseGlobs:
              - .prototools
  ```
- `sources` contains the focused projects plus their `dependsOn` projects, with **all** files of each project by default (tests, docs, fixtures). Narrow it per project, so a README edit does not rebuild the image (verified):

  ```yaml
  # apps/web/moon.yml
  docker:
      scaffold:
          sourcesPhaseGlobs:
              - src/**/*
              - package.json
              - tsconfig.json
  ```
- Scaffolding reads tracked files: uncommitted new files are missing from the skeleton.
- `moon docker prune` refuses to run without `dockerManifest.json` at the root (`app::docker::missing_manifest`), so running it on a developer machine by mistake is harmless.
- `moon docker file` produced `FROM golang:<version from .prototools>`, installed moon with `curl -fsSL https://moonrepo.dev/install/moon.sh | bash` (latest, unpinned), then `scaffold` → `setup` → `prune`. Pin moon: `curl -fsSL https://moonrepo.dev/install/moon.sh | bash -s -- 2.6.0`. A mismatch between the moon that scaffolds and the moon in the image is a classic source of "works locally, fails in Docker".
- Inside a build without `.git`, moon disables task caching and hashes no inputs (verified). `RUN moon run app:build` therefore always runs, and cannot use the remote cache, unless `.git` is in the build context.

## The generated Dockerfile, annotated

```dockerfile
FROM node:24-bookworm-slim AS base
WORKDIR /app
RUN apt-get update && apt-get install -y --no-install-recommends curl ca-certificates git xz-utils \
 && rm -rf /var/lib/apt/lists/*
RUN curl -fsSL https://moonrepo.dev/install/moon.sh | bash -s -- 2.6.0
ENV PATH="/root/.moon/bin:/root/.proto/bin:/root/.proto/shims:$PATH"

FROM base AS skeleton
# Whole repository: .dockerignore matters
COPY . .
RUN moon docker scaffold web

FROM base AS build
COPY --from=skeleton /app/.moon/docker/configs .
# Cached until a manifest, lockfile or .moon config changes anywhere in the workspace
RUN moon docker setup
COPY --from=skeleton /app/.moon/docker/sources .
RUN moon run web:build
# Production dependencies only
RUN moon docker prune

FROM node:24-bookworm-slim AS start
WORKDIR /app
COPY --from=build /app /app
CMD ["node", "apps/web/dist/server.js"]
```

- The skeleton stage makes the result independent of the host: scaffolding happens in Linux, inside the build. Prefer it over running `moon docker scaffold` on the host and copying `.moon/docker/*` (the "non-staged" variant), which depends on the host's moon version, line endings and file modes.
- `COPY . .` sends the whole repository to the builder. `.dockerignore` must exclude `.git` (unless you want caching inside the build), `.moon/cache`, `.moon/docker`, `node_modules`, `.venv`, `target`, build outputs.
- Toolchains are installed twice if the base image already has them (Node in `node:*`, Go in `golang:*`). Use a plain base (`debian:bookworm-slim`) and let `moon docker setup` install pinned versions, or keep the language image and set `MOON_TOOLCHAIN_FORCE_GLOBALS=true` (or `moon docker file --no-toolchain`) to use the image's binaries. Alpine needs `MOON_TOOLCHAIN_FORCE_GLOBALS=true`: Node has no musl builds for proto to install.
- `docker.file.template` (v2.0+) renders your own Tera template instead of the built-in one, so every service's Dockerfile can be regenerated from one place.

## When it pays off

| Ecosystem / shape                                          | moon docker?  | Why                                                                                     |
| ---------------------------------------------------------- | ------------- | --------------------------------------------------------------------------------------- |
| Node/TS app depending on several workspace packages        | **yes**       | replaces hand-maintained `COPY packages/*/package.json` lists; prune gives prod-only `node_modules` |
| Python service depending on several uv workspace members   | often         | same for `pyproject.toml`s; v2.5 prune removes unfocused `.venv`s. A single-package service is simpler with uv's own recipe |
| Go service                                                 | rarely        | one `go mod download`, static binary: build it as a moon task, copy into distroless (see [lang-go.md](lang-go.md)) |
| Rust service                                               | sometimes     | dependency layer caching is valuable (`cargo-chef` does the same job more precisely); v2.5 stopped leaving empty `lib.rs`/`main.rs` in skeletons |
| Static frontend (SPA)                                      | no            | build with moon (cached), copy `dist/` into an nginx/caddy image                        |
| Single-project repository                                  | no            | the problem it solves does not exist                                                    |

## Three patterns

**A. moon inside the image** (above). Reproducible from a clean checkout, no host tools needed beyond Docker. Costs: moon and proto in the build stage, toolchain downloads on cold builds, no moon cache inside the build without `.git`, and the dependency layer invalidated by any manifest in the workspace.

**B. Build outside, package inside.** moon builds the artifact in CI (affected-only, remote-cached), the Dockerfile only copies it:

```yaml
tasks:
    image:
        command: docker build -f Dockerfile -t $IMAGE:$GIT_SHA .
        inputs:
            - Dockerfile
            - "@group(src)"          # otherwise the task is never affected and `moon ci` skips it
            - "@group(cfg)"
        deps:
            - ~:build
        options:
            cache: false
            runFromWorkspaceRoot: false
        checks:
            - docker info          # v2.4+: fail fast when no daemon is reachable
```

Fast and cache-friendly. Costs: the artifact must be runnable in the image (static Go binary, platform-independent JS bundle; native modules or cgo need building for the image's platform), and the build environment is the CI runner, not the image.

**C. Ecosystem tools, no moon in the image.** `pnpm deploy --filter app --prod`, uv's `--no-install-project` layering, `cargo-chef`. Use when the image must not depend on moon, or the team already knows these recipes.

Large organisations usually mix: B for Go/Rust/static assets, A or C for Node/Python services. Registry layer caching (`--cache-to type=registry,mode=max`) complements all three.

## Remote cache inside a Docker build (advanced)

To let `RUN moon run app:build` hit the remote cache: keep `.git` in the context (or in the skeleton stage only), pass the token with a BuildKit secret, never with `ARG`/`ENV`:

```dockerfile
RUN --mount=type=secret,id=moon_token \
    MOON_REMOTE_TOKEN="$(cat /run/secrets/moon_token)" moon run app:build
```

Costs: `.git` makes the context large and changes every commit (keep it out of later layers), and the hash inside the build includes the container's platform data, so artifacts built on the CI host and inside the image may not be interchangeable. Pattern B is usually simpler.

## Compose and other container tasks

```yaml
tasks:
    compose-up:
        command: docker compose -f compose.yaml up -d --wait
        options:
            cache: false
            runFromWorkspaceRoot: false
            runInCI: false           # CI starts services itself, or use v2.6 wait/cleanup deps
        checks:
            - docker compose version
    compose-down:
        command: docker compose -f compose.yaml down --remove-orphans
        options:
            cache: false
            runFromWorkspaceRoot: false
```

- Paths in `compose.yaml` stay relative to the compose file. Never pass `$workspaceRoot` into bind mounts: on Windows it expands to `C:\...`, which a Docker engine inside WSL cannot mount.
- One stack per project, with `name:` set in the compose file, so two projects' stacks never collide; a `mutex` on tasks sharing a port or database.

## Windows

There is no native Linux container runtime on Windows. A developer has one of: Docker Desktop (WSL2 backend), Docker Engine installed inside a WSL2 distro, Rancher Desktop or Podman Desktop, or nothing. Decide per repository whether Docker tasks are supported on Windows hosts at all.

| Problem                                       | What happens                                                                 | Mitigation                                                                            |
| --------------------------------------------- | ---------------------------------------------------------------------------- | ------------------------------------------------------------------------------------- |
| No Docker available                           | Docker tasks fail with confusing shell errors                                | `checks: [docker info]` (v2.4+) for a clear failure; `runInCI`/`os` to keep them out of default runs |
| moon runs in Windows, Docker engine in WSL    | `docker` not on the Windows `PATH`; `$workspaceRoot` is a Windows path       | run the whole workflow inside WSL, or use Docker Desktop's Windows CLI; relative paths in compose |
| Tasks written for Bash                        | Windows tasks run in `pwsh` by default                                       | cross-platform scripts (Nushell, Python, Bun), or `options.os` variants of the task  |
| CRLF line endings (`core.autocrlf=true`)      | shell scripts copied into Linux images fail with `bash\r: not found`         | `.gitattributes`: `* text=auto eol=lf` (at least `*.sh text eol=lf`)                  |
| Executable bit lost                           | `permission denied` for entrypoints                                          | `git update-index --chmod=+x`, or `COPY --chmod=755`, or `RUN chmod +x`               |
| Repository on `/mnt/c` used from WSL          | very slow file access: hashing, `docker build` context upload, watchers       | keep the checkout in the WSL filesystem (`~/src/...`), open it with an IDE's WSL mode |
| Repository on the Windows filesystem, bind-mounted into containers | slow I/O and missing file events for dev servers          | same: put the repository in WSL, or use dev containers                                |
| Host scaffolding (non-staged Dockerfiles)     | skeleton carries host line endings and modes                                  | multi-stage Dockerfiles that scaffold inside Linux                                     |
| Line endings and hashes                       | working-tree bytes differ from Linux checkouts, so Windows machines may never hit Linux-produced cache entries | enforce LF with `.gitattributes`; compare with `moon hash` between machines     |
| Windows containers                            | the generated Dockerfile requires Bash in the base image                      | not supported by `moon docker file`; write those Dockerfiles by hand                  |
| Docker Desktop licensing                      | paid subscription required in larger companies                                | Rancher Desktop, Podman, Docker Engine in WSL; `moon docker *` only writes files, so any OCI builder works |
| WSL memory limits                             | builds killed with exit 137                                                   | raise `memory=` in `%UserProfile%\.wslconfig`                                         |

`moon docker scaffold`, `setup` and `prune` do not call Docker at all; they work with BuildKit, Buildah, Kaniko or Podman. Only tasks you write (`image`, `compose-*`) depend on a Docker CLI.
