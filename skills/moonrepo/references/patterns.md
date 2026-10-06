# File pattern cookbook

Every pattern below was checked on moon 2.5.6 with a throwaway `echo @files(<group>)` task against a real file tree. Patterns are project-relative unless they start with `/`.

## How to think about a group

A file group answers one question: "which files play this role?". Build it from three parts:

1. **Roots**: the folders the role lives in, as an alternative when there are several: `{src,server}/`.
2. **Shape**: depth plus explicit extensions and suffixes: `**/*.{ts,tsx}`, `**/*.test.int.ts`.
3. **Exclusions**: `!` lines that remove files belonging to another role: tests, stories, generated files.

Groups of one project should partition its files by role: a file is in `src` or in a test group, never in both, unless a tool really needs both views (for example `css` and `css-modules`).

## Recipes

### Source without tests, stories and generated files

```yaml
src:
    - "{src,server}/**/*.{ts,tsx}"
    - "!src/**/*.{test,test.int,stories}.{ts,tsx}"
    - "!src/**/.int/**"
    - "!src/test/**"
    - "!src/**/*.gen.ts"
```

Matched: `server/server.ts`, `src/ui/pages/home/Home.page.tsx`, `src/ui/widgets/Card/Card.widget.tsx`. Not matched: `Home.test.tsx`, `load.test.int.ts`, `.int/home.test.int.ts`, `Card.stories.tsx`, `routeTree.gen.ts`, `src/test/setup.ts`.

### Unit and integration tests

```yaml
test-unit:
    - tests/**/*.test.ts
    - src/**/*.test.{ts,tsx}
    - src/test/**/*.ts
test-int:
    - src/**/*.test.int.{ts,tsx}
    - src/**/.int/**/*.{ts,tsx}
```

`*.test.{ts,tsx}` does not match `load.test.int.ts`: `*` cannot swallow the `.int` part because the pattern requires the name to end with `.test.ts` or `.test.tsx`. Test helpers (`src/test/`) belong to the test group and are excluded from `src`.

### Generated files and stories as their own roles

```yaml
gen:
    - src/**/*.gen.ts
stories:
    - src/**/*.stories.tsx
```

A generator task lists `gen` patterns as `outputs`; consumers depend on the generator task instead of hashing the generated files as sources.

### CSS: all, modules only, global only

```yaml
css:
    - src/**/*.css
css-modules:
    - src/**/*.module.css
css-global:
    - src/**/*.css
    - "!src/**/*.module.css"
```

There is no `!(...)` extglob; exclusion is always a separate `!` line.

### Several roots

```yaml
css:
    - "{src,.storybook}/**/*.css"
cfg:
    - "{package,tsconfig}.json"
    - "{vite,vitest}.config.ts"
    - index.html
```

An alternative must not contain `/`. `{src,server}/**/*.ts` works, but `{src/**/*.ts,server/*.ts}` fails with `glob::create` because moon splits the pattern at the first `/`. Write one line per root instead:

```yaml
src:
    - src/**/*.ts
    - server/*.ts
```

### Fixed-width names with repetition

```yaml
migrations:
    - migrations/<[0-9]:14>_*.{up,down}.sql
```

Matches `20261003120000_init.up.sql`, rejects `2026_bad.up.sql`. `<glob:n,m>` repeats a sub-glob n to m times; `<[0-9]:4>` means exactly four digits.

### Go: language-mandated suffixes

```yaml
src:
    - "{cmd,internal}/**/*.go"
    - "!**/*_test.go"
test-unit:
    - "{cmd,internal}/**/*_test.go"
    - "!**/*_int_test.go"
test-int:
    - "**/*_int_test.go"
    - test/integration/**/*.{go,hurl}
cfg:
    - go.{mod,sum}
```

### Rust: crate layout

```yaml
src:
    - src/**/*.rs
test-int:
    - tests/**/*.rs
bench:
    - benches/**/*.rs
cfg:
    - Cargo.toml
```

Rust unit tests live inside `src/` (`#[cfg(test)]`), so `test-unit` is empty and the `test-unit` task hashes `src`.

### Python: src layout and pytest

```yaml
src:
    - src/**/*.py
    - "!src/**/test_*.py"
    - "!src/**/*_test.py"
test-unit:
    - tests/unit/**/*.py
    - tests/conftest.py
test-int:
    - tests/integration/**/*.py
cfg:
    - pyproject.toml
```

`.venv/`, `__pycache__/`, `.pytest_cache/` and `.ruff_cache/` are git-ignored, so they never become inputs even under a broad glob. Do not rely on that for files you create yourself: an untracked, non-ignored scratch file inside `src/` is hashed.

### Terraform and other single-extension trees

```yaml
src:
    - "**/*.{tf,tfvars}"
```

Allowed because the extensions are explicit. `.terraform/` is ignored by git, so it is never hashed.

### Workspace-relative configs

```yaml
cfg-lint:
    - /oxlint.config.ts
cfg-css:
    - /stylelint.config.js
    - /postcss.config.js
cfg-lint-go:
    - /.golangci.yml
cfg-lint-py:
    - /ruff.toml
```

Declared once in `.moon/tasks/*.yaml`, referenced only by the tasks of that tool.

## Inputs beyond groups

| Need                                                         | Input                                                     |
| ------------------------------------------------------------ | --------------------------------------------------------- |
| A file that may be absent without a warning                  | `- file: /go.work` with `optional: true`                  |
| Re-run only when a file's content matches a regex            | `- file: /.env.example` with `content: "^API_"`           |
| Files of another project, by its group                       | `project://design-system?group=css`                       |
| Files of every dependency project                            | `- project: "^"` with `group: src`                        |
| An environment variable                                      | `$API_URL`, or `$VITE_*` for a family                     |
| Output of a command (tool version, platform)                 | task `checks` with `check: fingerprint` (v2.4+)           |
| Everything except a few files of a known tree                | a positive pattern, then `!` lines; never `**/*` alone    |

Environment variable inputs hash the variable's value, so the task is stale whenever the value differs between runs or machines. Use them only for variables that change the result, and always for variables that are baked into outputs (frontend `VITE_*`/`NEXT_PUBLIC_*`, Go `-ldflags -X` values): otherwise a cached artifact built with one value is replayed for another.

## Anti-patterns and their fixes

| Wrong                                            | Why                                                         | Right                                                       |
| ------------------------------------------------ | ----------------------------------------------------------- | ----------------------------------------------------------- |
| `src/**/*`                                       | Hashes READMEs, snapshots, stray files                      | `src/**/*.{ts,tsx}`                                         |
| `**/*.{js,jsx,mjs,cjs,ts,tsx,mts,cts}`           | Catch-all extension list over the whole project             | the extensions the project really has, under its roots     |
| `*.config.{js,ts}`                               | Picks up root-owned configs copied by mistake, any new tool | `{vite,vitest}.config.ts`                                   |
| `{src,server}/*.test.ts`                         | Only direct children, deeper tests are silently skipped     | `{src,server}/**/*.test.ts`                                 |
| `!**/*.test.{js,ts,tsx}` with no positive line   | An exclusion alone matches nothing                          | a positive pattern first, then the exclusion                |
| `../../../.golangci.yaml` in a command           | Climbs out of the project, breaks on moves                  | input `/.golangci.yaml`, command `$workspaceRoot/.golangci.yaml` |
| `**/*` in an e2e project                         | Hashes reports, traces and fixtures output                  | `{specs,fixtures}/**/*.ts`                                  |
| `/python/**/*` with `!` lines for `.venv` etc.   | One edit anywhere in the tree invalidates every Python task | per-project `src`/`test-*` groups plus `/uv.lock`           |
| `/rust/**/*` for a crate task                    | Same, plus `target/` leaks in when not ignored              | the crate's `src/**/*.rs`, `Cargo.toml`, `/Cargo.lock`      |
