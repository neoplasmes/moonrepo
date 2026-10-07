# Docker

Docs: [guides/docker](https://moonrepo.dev/docs/guides/docker), [commands/docker/scaffold](https://moonrepo.dev/docs/commands/docker/scaffold), [setup](https://moonrepo.dev/docs/commands/docker/setup), [prune](https://moonrepo.dev/docs/commands/docker/prune), [file](https://moonrepo.dev/docs/commands/docker/file), settings in [config/workspace#docker](https://moonrepo.dev/docs/config/workspace#docker) and [config/project#docker](https://moonrepo.dev/docs/config/project#docker).

This file gives one canonical recipe (pnpm workspaces), the verified facts behind it, and Windows caveats. For other ecosystems it gives pointers only: read the docs and the ecosystem's own Docker guidance.

## Verdict per command

| Command                  | Use it?                         | Why (verified on moon 2.5.5 and 2.6.0)                                                                 |
| ------------------------ | ------------------------------- | ------------------------------------------------------------------------------------------------------ |
| `moon docker scaffold`   | **yes**                         | computes from the project graph which manifests and which sources an image needs; replaces hand-written `COPY` lists that drift |
| `moon docker setup`      | **no** for pnpm                 | runs a plain `pnpm install` for the whole workspace (debug log: `Running command bash -c "pnpm install"`), not the focused subtree |
| `moon docker prune`      | **no** for pnpm                 | `pnpm deploy --prod` does the same job more precisely and produces a self-contained folder              |
| `moon docker file`       | draft only                      | installs moon unpinned (`curl ... moon.sh \| bash`), ignores `.prototools`, assumes Bash in the base image |

## Canonical recipe: pnpm workspace

Verified end to end with moon 2.6.0, pnpm 10.18.0, Node 24, Docker 29 on a workspace where `server` depends on a workspace package and an npm package, next to an unrelated project.

```dockerfile
# Build from the workspace root: docker build -f apps/server/Dockerfile .

ARG NODE_VERSION=24.10.0
ARG PNPM_VERSION=10.18.0
ARG MOON_VERSION=2.6.0

# moon project id and pnpm package name of the image's application
ARG MOON_PROJECT=server
ARG PNPM_PACKAGE=server

#### moon: pinned moon on a glibc image
FROM node:${NODE_VERSION}-bookworm-slim AS moon
ARG MOON_VERSION
RUN apt-get update \
 && apt-get install -y --no-install-recommends ca-certificates curl git xz-utils \
 && rm -rf /var/lib/apt/lists/* \
 && curl -fsSL https://moonrepo.dev/install/moon.sh | MOON_INSTALL_DIR=/usr/local/bin bash -s -- ${MOON_VERSION}

#### skeleton: moon computes which manifests and sources the image needs
FROM moon AS skeleton
WORKDIR /repo
COPY . .
ARG MOON_PROJECT
RUN --mount=type=cache,id=moon-home,target=/root/.moon \
    --mount=type=cache,id=proto-home,target=/root/.proto \
    --mount=type=cache,id=wasmtime,target=/root/.cache/wasmtime \
    moon docker scaffold ${MOON_PROJECT}

#### base: node + pinned pnpm + moon
FROM moon AS base
ARG PNPM_VERSION
ENV CI=true \
    PNPM_HOME=/pnpm \
    MOON_TOOLCHAIN_FORCE_GLOBALS=true
ENV PATH="$PNPM_HOME:$PATH"
RUN corepack enable && corepack prepare pnpm@${PNPM_VERSION} --activate
WORKDIR /app

#### fetch: download every package in the lockfile; depends on the lockfile only
FROM base AS fetch
COPY pnpm-lock.yaml pnpm-workspace.yaml ./
RUN pnpm fetch

#### build: offline install of the app's subtree, then build through moon
FROM fetch AS build
ARG MOON_PROJECT
ARG PNPM_PACKAGE
COPY --from=skeleton /repo/.moon/docker/configs/ ./
RUN pnpm install --offline --frozen-lockfile --filter "${PNPM_PACKAGE}..."
COPY --from=skeleton /repo/.moon/docker/sources/ ./
RUN --mount=type=cache,id=moon-home,target=/root/.moon \
    --mount=type=cache,id=proto-home,target=/root/.proto \
    --mount=type=cache,id=wasmtime,target=/root/.cache/wasmtime \
    moon run ${MOON_PROJECT}:build --no-actions
RUN pnpm --filter "${PNPM_PACKAGE}" deploy --prod --legacy /out

#### runtime: no moon, no pnpm, no build tools
FROM node:${NODE_VERSION}-alpine AS runtime
ENV NODE_ENV=production
WORKDIR /app
COPY --from=build --chown=node:node /out ./
USER node
CMD ["node", "dist/index.js"]
```

```text
# .dockerignore
.git
**/node_modules
**/dist
.moon/cache
.moon/docker
```

```yaml
# .moon/workspace.yml
docker:
    scaffold:
        configsPhaseGlobs:     # not copied by default
            - .prototools
            - .npmrc
```

```yaml
# apps/server/moon.yml: keep tests, docs and fixtures out of the image context
docker:
    scaffold:
        sourcesPhaseGlobs:
            - src/**/*
            - package.json
            - tsconfig.json
            - moon.yml
```

### Measured behaviour

| Change                                                | What rebuilds                                         | Time (warm builder) |
| ----------------------------------------------------- | ----------------------------------------------------- | ------------------- |
| source file of an unrelated project                   | only the skeleton stage; every later layer `CACHED`   | under 10 s total    |
| `package.json` of an unrelated project (no lockfile change) | install (offline, ~1 s), build, deploy; `pnpm fetch` stays cached | ~6 s total |
| `pnpm-lock.yaml`                                      | everything from `pnpm fetch` on                        | network-bound       |
| the app's sources                                     | build and deploy                                       | build-bound         |

### Why each piece is there

- **Scaffold only computes file lists.** `configs` gets the manifests of *every* project in the workspace, not only the app's subtree, and `configsPhaseGlobs` can only add files, never remove a toolchain's manifests (verified: setting it on the unrelated project changed nothing). The `pnpm fetch` layer is what makes this harmless: the expensive download depends on the lockfile alone, and a foreign manifest change only re-runs a ~1 s offline install.
- **`pnpm install --offline --filter "<pkg>..."` instead of `moon docker setup`**: installs only the app and its workspace dependencies, from the store `pnpm fetch` filled. `moon docker setup` would install the whole workspace.
- **`moon run <project>:build` instead of repeating `pnpm exec tsup && ...`**: [MOONREPO FIRST](../SKILL.md#moonrepo-first). moon orders `^:build` dependencies itself; a hand-written chain in the Dockerfile is a second source of truth.
  - `--no-actions` skips toolchain setup and `InstallDependencies`; without it moon would run its own unfiltered `pnpm install`.
  - `MOON_TOOLCHAIN_FORCE_GLOBALS=true` makes moon use the image's Node and pnpm instead of downloading them through proto.
  - There is no `.git` in the context, so moon's own cache is disabled inside the build (verified). Docker's layer cache does that job here.
  - If the repository pins the task shell (`unixShell: nu`), install that shell in `base`, or the build task will not start.
- **`pnpm deploy --prod --legacy`** produces a self-contained folder with production dependencies.
  - It respects the lockfile: `^2.0.0` locked to `2.0.0` stayed `2.0.0` (verified).
  - It cannot run `--offline`: it re-reads registry metadata and fails with `ERR_PNPM_NO_OFFLINE_META` (verified).
  - Non-legacy deploy requires `injectWorkspacePackages: true`, which changes local development (workspace packages are copied, not linked). Keep `--legacy` unless the repository already injects.
- **moon installed to `/usr/local/bin`** because the cache mount covers all of `~/.moon`. Two verified traps:
  - Mounting only `~/.moon/plugins` breaks moon: it downloads into `~/.moon/temp` and renames into `plugins`, which fails across file systems (`fs::rename ... Invalid cross-device link (os error 18)`).
  - Mounting all of `~/.cache` hides corepack's pnpm (`~/.cache/node/corepack`). Mount only `~/.cache/wasmtime`, where compiled WASM plugins live.
- **The three cache mounts** turn moon's startup inside the build from ~8–10 s (plugin download plus WASM compilation, measured) into ~0.7 s. Ephemeral CI builders start with empty mounts and pay the 8–10 s once per build; export the BuildKit cache if that matters.
- **No `# syntax=docker/dockerfile:1` line**: it downloads the frontend from Docker Hub on every build, and a network blip fails the build (seen). Docker 23+ supports `--mount` without it.
- **Alpine only in `runtime`**: moon and proto need glibc; the runtime image needs neither.
- **Versions**: the `ARG` defaults duplicate `.prototools`. Renovate updates `.prototools` natively ([guides/renovate](https://moonrepo.dev/docs/guides/renovate)); Dockerfile `ARG`s need a `# renovate:` annotation or a custom manager. Either set that up, or let the `image` task call a small script in `tools/` that reads `.prototools` and passes `--build-arg`s. The task itself:

```yaml
tasks:
    image:
        command: docker build -f Dockerfile -t $IMAGE:$GIT_SHA $workspaceRoot
        inputs:
            - Dockerfile
            - "@group(src)"          # without inputs the task is never affected and `moon ci` skips it
            - "@group(cfg)"
            - /pnpm-lock.yaml
            - /.prototools
        deps:
            - ~:build
        options:
            cache: false
            runFromWorkspaceRoot: false
        checks:
            - docker info            # v2.4+: fail fast when no daemon is reachable
```

- **Private registries**: copy `.npmrc` into the `fetch` stage and pass tokens with `RUN --mount=type=secret,id=npmrc,target=/app/.npmrc pnpm fetch`, never with `ARG`/`ENV`.

### Adapting the recipe

- Another app: change `MOON_PROJECT`, `PNPM_PACKAGE` and `CMD`. Everything else stays.
- Next.js standalone, static SPAs: keep `skeleton`/`fetch`/`build`; replace `deploy` and `runtime` with copying `.next/standalone` or `dist/` into the runtime image (nginx/caddy for static files).
- npm or yarn workspaces: no `pnpm fetch` equivalent with the same guarantees. Read the docs, and expect `moon docker setup`/`prune` to be closer to useful there; verify what they run with `--log debug` before trusting them.

## Verified facts about the `moon docker` commands

- Skeleton folders are `.moon/docker/configs` and `.moon/docker/sources`. Parts of the Docker guide still say `workspace`.
- With `.git` present, scaffold copies tracked files only. Without `.git` (the usual case inside a build), it walks the file system, so `.dockerignore` decides what ends up in `sources` (an untracked file was copied).
- `.prototools` is not copied into `configs` unless listed in `docker.scaffold.configsPhaseGlobs`.
- `sourcesPhaseGlobs` narrows `sources` per project; the default is the whole project folder.
- `moon docker prune` refuses to run without `dockerManifest.json` at the root, so running it on a developer machine by mistake is harmless.
- Without `.git`, moon disables task caching and hashes no inputs.

## Other ecosystems (pointers, not prescriptions)

- **Go, Rust, static frontends**: build the artifact as a cached moon task, copy it into a minimal image (distroless, scratch, nginx). See [lang-go.md](lang-go.md#docker). `cargo-chef` for Rust dependency layers.
- **Python**: uv's own Docker guidance (lockfile-only `uv sync --no-install-project` layer, then sources). Since v2.5 `moon docker prune` removes unfocused `.venv`s; check what `setup` runs with `--log debug` before using it.
- Anything else: read [guides/docker](https://moonrepo.dev/docs/guides/docker) for the current behaviour of the installed version, then check `moon toolchain info <id>` for `scaffold_docker`/`prune_docker` support.

## Compose and container tasks

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
- One stack per project with `name:` set in the compose file; a `mutex` on tasks sharing a port or database.

## Windows

There is no native Linux container runtime on Windows. A developer has one of: Docker Desktop (WSL2 backend), Docker Engine inside a WSL2 distro, Rancher Desktop or Podman Desktop, or nothing. Decide per repository whether Docker tasks are supported on Windows hosts at all.

| Problem                                       | What happens                                                                 | Mitigation                                                                            |
| --------------------------------------------- | ---------------------------------------------------------------------------- | ------------------------------------------------------------------------------------- |
| No Docker available                           | Docker tasks fail with confusing shell errors                                | `checks: [docker info]` (v2.4+) for a clear failure; `runInCI`/`os` to keep them out of default runs |
| moon runs in Windows, Docker engine in WSL    | `docker` not on the Windows `PATH`; `$workspaceRoot` is a Windows path       | run the whole workflow inside WSL, or use Docker Desktop's Windows CLI; relative paths in compose |
| Tasks written for Bash                        | Windows tasks run in `pwsh` by default                                       | cross-platform scripts (Nushell, Python, Bun), or `options.os` variants of the task  |
| CRLF line endings (`core.autocrlf=true`)      | shell scripts copied into Linux images fail with `bash\r: not found`         | `.gitattributes`: `* text=auto eol=lf` (at least `*.sh text eol=lf`)                  |
| Executable bit lost                           | `permission denied` for entrypoints                                          | `git update-index --chmod=+x`, or `COPY --chmod=755`, or `RUN chmod +x`               |
| Repository on `/mnt/c` used from WSL          | very slow file access: hashing, `docker build` context upload, watchers       | keep the checkout in the WSL filesystem (`~/src/...`), open it with an IDE's WSL mode |
| Repository on the Windows filesystem, bind-mounted into containers | slow I/O and missing file events for dev servers          | same: put the repository in WSL, or use dev containers                                |
| Host scaffolding (non-staged Dockerfiles)     | skeleton carries host line endings and modes                                  | scaffold inside the build (the recipe's `skeleton` stage)                             |
| Line endings and hashes                       | working-tree bytes may differ from Linux checkouts, so Windows machines may never hit Linux-produced cache entries (not verified) | enforce LF with `.gitattributes`; compare with `moon hash` between machines |
| Windows containers                            | the generated Dockerfile requires Bash in the base image                      | not supported by `moon docker file`; write those Dockerfiles by hand                  |
| Docker Desktop licensing                      | paid subscription required in larger companies                                | Rancher Desktop, Podman, Docker Engine in WSL; `moon docker *` only writes files, so any OCI builder works |
| WSL memory limits                             | builds killed with exit 137                                                   | raise `memory=` in `%UserProfile%\.wslconfig`                                         |
