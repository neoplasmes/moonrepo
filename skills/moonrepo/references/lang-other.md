# Other languages

Docs: [guides/rust/handbook](https://moonrepo.dev/docs/guides/rust/handbook), [how languages are supported](https://moonrepo.dev/docs/how-it-works/languages), [tools proto installs](https://moonrepo.dev/docs/proto/tools), `moon toolchain info <id>`.

The task vocabulary never changes with the language. A new ecosystem brings a `.moon/tasks/<ecosystem>.yml`, pinned tools in `.prototools`, and projects selected through `language`, a toolchain or `tags`.

## Checklist for a new language

1. Pin the compiler, formatter, linter and test runner (`.prototools`; a proto TOML/WASM plugin when proto has none, see [wasm-plugins.md](wasm-plugins.md)).
2. Check whether moon has a toolchain (`moon toolchain info <id>`). Without one, the language still works: tasks just call the pinned binaries, and you add `/.prototools` and the lockfile as implicit inputs. `language` accepts any string.
3. Create `.moon/tasks/<ecosystem>.yml` with shared groups, `fmt`, `lint`, `lint-fix`, `typecheck`, `test-*` and the `check` proxy.
4. Give each project its file groups (see [patterns.md](patterns.md)) and `language`.
5. Register projects in `.moon/workspace.yml` and in the language's own workspace file (Cargo workspace, `go.work`, uv workspace).
6. Map fixed test layouts onto `test-unit`/`test-int`, never new task names.
7. Decide what platform data belongs in hashes: if the toolchain does not add os/arch and outputs are native, add a `fingerprint` check.

## One formatter entry point (optional)

Some repositories route every formatter through dprint so `fmt` is the same command everywhere and one config holds all formatting rules:

```jsonc
// dprint.jsonc, exec plugin
{
	"exec": {
		"commands": [
			{ "command": "gofmt", "exts": ["go"] },
			{ "command": "rustfmt --edition 2024 --emit stdout", "exts": ["rs"] },
			{ "command": "terraform fmt -", "exts": ["tf", "tfvars"] },
			{ "command": "ruff format --stdin-filename {{file_path}} -", "exts": ["py"] }
		]
	}
}
```

Gain: one `fmt` task, one cache, one place to configure. Cost: another tool in the chain, and editors must also be pointed at dprint. If the repository does not already do this, use each ecosystem's native formatter inside its own `fmt` task.

## Rust

One Cargo workspace at the root (`/Cargo.toml`, `/Cargo.lock`), each crate a project. Unit tests live inside `src/` under `#[cfg(test)]`; integration tests are Cargo's `tests/` folder.

`.moon/tasks/rust.yaml`:

```yaml
inheritedBy:
    languages: rust

implicitInputs:
    - /Cargo.toml
    - /Cargo.lock

fileGroups:
    test-int: []
    cfg-lint-rust:
        - /clippy.toml

tasks:
    lint:
        command: cargo clippy --all-targets --manifest-path $projectSource/Cargo.toml -- -D warnings
        inputs: &rust-lint-inputs
            - "@group(src)"
            - "@group(test-int)"
            - "@group(cfg)"
            - "@group(cfg-lint-rust)"

    lint-fix:
        command:
            cargo clippy --fix --allow-dirty --allow-staged --all-targets
            --manifest-path $projectSource/Cargo.toml
        inputs: *rust-lint-inputs

    typecheck:
        command: cargo check --all-targets --manifest-path $projectSource/Cargo.toml
        inputs:
            - "@group(src)"
            - "@group(test-int)"
            - "@group(cfg)"

    test-unit:
        command: cargo test --lib --manifest-path $projectSource/Cargo.toml
        inputs:
            - "@group(src)"
            - "@group(cfg)"

    test-int:
        command: cargo test --test '*' --manifest-path $projectSource/Cargo.toml
        inputs:
            - "@group(src)"
            - "@group(test-int)"
            - "@group(cfg)"
```

`lint-fix` uses an anchor and not `extends`, because `--fix` must stay before `--`. `fmt` is `cargo fmt --manifest-path $projectSource/Cargo.toml`; `check` is the usual proxy.

The Rust toolchain (`rust: {}` in `.moon/toolchains.yml`) installs the channel with rustup, installs `components`/`targets`, can sync `rust-toolchain.toml`, and adds the Rust version to hashes. `target/` is git-ignored, so it is never an input, but it is also huge: never list it as an output; list only the final artifact (`/target/release/<bin>`). Persist `~/.cargo` and `target/` in CI with a Rust-specific cache (`Swatinem/rust-cache`); moon's remote cache only carries declared outputs. Root-level `lock` for Cargo is `cargo update --workspace`. A crate compiled to WebAssembly adds a project `build` task whose `outputs` are the generated `pkg/` folder, and consumers depend on it with `cacheStrategy: outputs`.

## Terraform and other infrastructure

`language` accepts any value, so infrastructure projects say what they are. `plan` and `apply` read remote state and talk to cloud APIs: never cache them, and run them interactively locally. In CI, `plan` runs with `runInCI: true` on affected stacks and posts its output; `apply` runs only from a deploy workflow, never from `moon ci`. `.terraform/` and `.terraform.lock.hcl` handling: the lock file is committed and is an input of `lint`/`validate`; `.terraform/` is ignored.

```yaml
language: terraform
layer: configuration

tags:
    - terraform

fileGroups:
    src:
        - "**/*.{tf,tfvars}"

tasks:
    lint:
        command: tflint --recursive
        inputs:
            - "@group(src)"
            - /.tflint.hcl
        options:
            runFromWorkspaceRoot: false

    lint-fix:
        extends: lint
        args: --fix

    plan:
        command: terragrunt plan
        preset: utility
        options:
            runFromWorkspaceRoot: false

    apply:
        command: terragrunt apply
        preset: utility
        options:
            runFromWorkspaceRoot: false
            runInCI: false
```

## HTTP API tests with Hurl

Hurl scenarios are integration tests of one service (`test-int`) or end-to-end tests across services (`test-e2e` in `e2e/<suite>/`).

```yaml
fileGroups:
    test-int:
        - test/integration/**/*.hurl

tasks:
    test-int:
        command: hurl --test --variables-file test/integration/local.env @files(test-int)
        inputs:
            - "@group(src)"
            - "@group(test-int)"
            - test/integration/local.env
        deps:
            - api:build
        options:
            mutex: api-local-port
            runFromWorkspaceRoot: false
```

`mutex` keeps two tasks that bind the same port or database from running at the same time.

## Shell and script-only projects

- Bash scripts do not run on Windows outside WSL or Git Bash. A cross-platform repository writes multi-step logic in a language every developer already has: Nushell, Python (`uv run script.py`), Bun/Node, or Go (`go run ./tools/x`). Pin the interpreter in `.prototools`.
- Lint scripts as their own capability: `shellcheck @files(sh)` for Bash, `nu --ide-check` or a parse loop for Nushell, ruff for Python tools. A project that is only `tools/` usually belongs to the root project.
- A script that a task calls is an input of that task. Forgetting it means editing the script does not invalidate the cache.

## Anything else (Java, .NET, Swift, Ruby, Zig, C/C++)

- proto ships Java (v0.59), Swift (v0.61) and Zig (v0.62) plugins; moon has an unstable Ruby toolchain (v2.4). Others go through `.prototools` with a plugin, or a system install pinned by your CI image.
- Build systems with their own incremental caches (Gradle, MSBuild, CMake/Ninja, Bazel) should stay the source of truth for compilation. moon orchestrates them as tasks with narrow outputs (the final artifact), and its affected detection decides whether to call them at all. Do not try to mirror their internal graph in moon tasks.
- Project discovery by file glob (v2.5) works for any manifest: `src/**/*.csproj`, `**/build.gradle.kts`.
