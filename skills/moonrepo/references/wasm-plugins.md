# WASM plugins: when they are worth it

Docs: [guides/wasm-plugins](https://moonrepo.dev/docs/guides/wasm-plugins), [guides/extensions](https://moonrepo.dev/docs/guides/extensions), [config/extensions](https://moonrepo.dev/docs/config/extensions), [proto/non-wasm-plugin](https://moonrepo.dev/docs/proto/non-wasm-plugin), [proto/wasm-plugin](https://moonrepo.dev/docs/proto/wasm-plugin), [proto/plugins](https://moonrepo.dev/docs/proto/plugins).

moon and proto load plugins compiled to WebAssembly (Extism runtime, wasmtime underneath). There are three kinds, and for most needs there is a cheaper option than writing one. Read the decision table first.

> The plugin APIs are marked experimental: breaking changes can land in non-major releases. moon has not yet adopted the reworked path API from proto v0.60, so moon plugins still use the older `VirtualPath` helpers. Pin moon when you depend on a custom plugin.

## The three kinds

| Kind                       | Configured in                       | What it can do                                                                                       |
| -------------------------- | ----------------------------------- | ---------------------------------------------------------------------------------------------------- |
| **proto tool plugin**      | `.prototools` `[plugins.tools]`     | resolve versions, download, verify, install and locate a tool's binaries                            |
| **moon toolchain plugin** (v2) | `.moon/toolchains.yml` (`plugin:`) | everything a built-in toolchain does: detect projects, infer `dependsOn` and aliases, add data to task hashes, install dependencies, set up the environment, parse lockfiles, sync config files, Docker scaffold/prune, plus tool installation through proto |
| **moon extension**         | `.moon/extensions.yml`              | a `moon ext <id> -- args` command; since v2 also hooks: `extend_project_graph`, `extend_task_command`, `extend_task_script`, `extend_command`, `sync_project`, `sync_workspace` |

`moon toolchain info <id>` and `moon extension info <id>` print which APIs a plugin implements, which is the quickest way to learn what a built-in one does before writing your own.

## Decision table

| You want                                                                 | Cheapest working option                                                  | WASM is justified when                                                                 |
| ------------------------------------------------------------------------ | ------------------------------------------------------------------------ | -------------------------------------------------------------------------------------- |
| Pin a CLI that proto does not know                                       | built-in or community plugin (proto v0.62 resolves community ids with no config); backend `npm:`, `cargo:`, `asdf:` (asdf: not on Windows); a **TOML plugin** | versions come from an API with logic, the archive layout differs per version, post-install steps are needed |
| Run custom logic as part of the workflow                                 | a script under `tools/` called by a task (Python, Nushell, Bun, Go)      | the logic must run inside moon's lifecycle (graph, hash, sync), or ship to many repositories as one versioned binary with no interpreter |
| Add tasks to projects automatically based on their files                 | `.moon/tasks/*.yml` with `inheritedBy: { files: [...] }` or tags         | the tasks depend on file contents (e.g. one `test-int` per compose file, one `gen` task per OpenAPI spec found) → extension with `extend_project_graph` |
| First-class support for a language moon does not have (Gradle, .NET, Elixir, an in-house build tool) | `language: <x>` + tasks calling the pinned tool               | you need dependency inference from manifests, lockfile-aware hashing (only projects whose resolved deps changed are invalidated), automatic installs, Docker prune → toolchain plugin |
| Enforce an org-wide policy (licenses, ownership, forbidden deps)         | a lint task with a script, run in CI                                     | the same check must run identically in hundreds of repositories, versioned and sandboxed → extension |
| Keep a generated file in sync (editor settings, CI matrix, docs index)   | a task with `runInSyncPhase: true` (v2.1+)                               | it needs the full project graph and must run on every `moon sync` everywhere → extension with `sync_workspace` |
| Inject environment into every task (tracing ids, proxy settings)         | `.moon/tasks/all.yml` `env` (v2.5+)                                      | values are computed per task → extension with `extend_task_command`                   |
| Migrate from Nx or Turborepo                                             | built-in extensions `moon ext migrate-nx`, `migrate-turborepo`           | never; use them, then fix by hand                                                      |

Rule of thumb: if a shell or Python script in `tools/` plus a task solves it, do that. Write a WASM plugin when the logic must live **inside** moon (graph, hashing, sync, install) or must be **distributed** as one pinned artifact to many repositories.

## proto TOML plugins (no WASM)

Most "pin this CLI" needs are a static file:

```toml
# tools/proto/helm.toml
name = "helm"
type = "cli"

[platform.linux]
download-file = "helm-v{version}-{os}-{arch}.tar.gz"
checksum-file = "helm-v{version}-{os}-{arch}.tar.gz.sha256sum"
archive-prefix = "{os}-{arch}"
exe-path = "helm"

[platform.macos]
download-file = "helm-v{version}-{os}-{arch}.tar.gz"
checksum-file = "helm-v{version}-{os}-{arch}.tar.gz.sha256sum"
archive-prefix = "{os}-{arch}"
exe-path = "helm"

[platform.windows]
download-file = "helm-v{version}-{os}-{arch}.zip"
checksum-file = "helm-v{version}-{os}-{arch}.zip.sha256sum"
archive-prefix = "{os}-{arch}"
exe-path = "helm.exe"

[install]
download-url = "https://get.helm.sh/{download_file}"
checksum-url = "https://get.helm.sh/{checksum_file}"

[install.arch]
x64 = "amd64"
arm64 = "arm64"

[install.os]
macos = "darwin"

[resolve]
git-url = "https://github.com/helm/helm"
```

```toml
# .prototools
helm = "3.19.0"

[plugins.tools]
helm = "file://./tools/proto/helm.toml"
```

- Declare every platform the team uses. A plugin without `[platform.windows]` silently makes the tool uninstallable for Windows developers.
- Always set checksums. A TOML plugin without them downloads and runs whatever the URL serves.
- `[resolve] git-url` lists versions from tags; `manifest-url` from a JSON endpoint. A fixed `versions = [...]` list works but needs editing on every bump.

## Using third-party WASM plugins

Locators:

```yaml
# .moon/extensions.yml
policy:
    plugin: github://acme/moon-policy@v1.4.0        # pinned release asset
    forbiddenDeps: [left-pad]                       # plugin-specific settings (camelCase)
local-tool:
    plugin: file://./tools/plugins/local_tool.wasm
pinned-url:
    plugin: https://plugins.example.com/x-1.2.0.wasm
```

- Always pin (`@v1.4.0`). An unpinned `github://` locator takes the latest release and caches it for 7 days: builds change without a commit.
- `github://` uses the GitHub API; set `GITHUB_TOKEN` in CI to avoid rate limits.
- The WASM sandbox restricts **file system** access to whitelisted virtual paths (`/workspace`, `/userhome`, `/proto`, `/moon`), but host functions let plugins **execute commands** (`exec_command`), make HTTP requests and set environment variables. A third-party plugin is code you run with your permissions. Review it like a dependency.
- WASI limits: no `chmod`, so plugins cannot unpack archives with modes; that work goes through host functions.
- Debug with `MOON_DEBUG_WASM=true` and `--log trace`.

## Writing a plugin

Follow the docs, they track the PDK API: [guides/wasm-plugins](https://moonrepo.dev/docs/guides/wasm-plugins) (concepts, host functions, building), [guides/extensions](https://moonrepo.dev/docs/guides/extensions) (extension APIs), [proto/wasm-plugin](https://moonrepo.dev/docs/proto/wasm-plugin) (tool APIs). Reference implementations: `moonrepo/plugins`, `moonrepo/moon-extensions`; `moonrepo/build-wasm-plugin` builds, optimises and publishes releases. Gotchas worth knowing up front:

- Target `wasm32-wasip1`, `crate-type = ["cdylib"]`; release builds need `wasm-opt`/`wasm-strip` or the files are large.
- Read the host os/arch with `get_host_environment()`; never assume Linux.
- Paths are virtual (`/workspace`, `/userhome`); convert before logging or passing to commands. moon still uses the pre-proto-v0.60 path API.
- Test with a `file://` locator before publishing; consumers pin a release tag.

## Cost summary

| Cost                          | Notes                                                                    |
| ----------------------------- | ------------------------------------------------------------------------ |
| Skills                        | Rust, Extism PDK, moon/proto PDKs, WASI quirks                           |
| API churn                     | experimental; expect to update plugins on moon minor upgrades           |
| Distribution                  | release pipeline, versioning, pinning in every consumer                  |
| Debuggability                 | logs only through host functions; no debugger in the sandbox            |
| Payoff                        | one artifact, identical on every OS, integrated in graph/hash/sync, no interpreter needed |

For a single repository, a WASM plugin rarely pays off. For a platform team serving tens of repositories, a toolchain plugin for the company's main in-house stack, or an extension that encodes org policy, can.
