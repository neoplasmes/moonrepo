# Python

The examples use uv. pip and Poetry differ only in the lock and install commands.

## Two ways to run Python under moon

| Approach                                                        | Gain                                                                  | Cost                                                                                           |
| --------------------------------------------------------------- | --------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------- |
| `unstable_python` + `unstable_uv` toolchains                    | moon installs Python and uv, creates the venv and runs `uv sync` before tasks, prunes `.venv` in Docker, infers `dependsOn` | **unstable**: settings and behaviour may change in minor releases; implicit `uv venv`/`uv sync` side effects on every run after a lockfile change |
| No Python toolchain: `python` and `uv` pinned in `.prototools`, tasks call `uv run --locked ...` | explicit, stable, same commands as outside moon                        | you write `setup` yourself, and nothing infers project edges                                  |

Both are valid. Pick the toolchain for new repositories that can follow moon releases; pick the explicit form for repositories that pin moon for a long time or where surprise installs are unacceptable.

```yaml
# .moon/toolchains.yml (toolchain approach)
unstable_python:
    packageManager: uv     # pip | poetry | uv | uv-pip
    venvName: .venv
unstable_uv:
    syncArgs: [--locked]
```

Observed on moon 2.5: the first task in a Python project ran `uv venv .venv` and then `uv sync` before the task itself.

## What enters the hash

The Python toolchain does **not** implement task-hash contributions (`moon toolchain info unstable_python`: `hash_task_contents` is off). Neither the Python version nor the platform is hashed automatically. Do it yourself:

```yaml
# .moon/tasks/python.yml
implicitInputs:
    - /.prototools          # python and uv versions
    - /uv.lock
    - /pyproject.toml       # workspace root, if any
    - file: /.python-version
      optional: true
```

and, for tasks whose outputs are platform-specific (wheels with C extensions, PyInstaller bundles, a packed `.venv`):

```yaml
checks:
    - check: fingerprint
      script: python -c "import sys, platform; print(sys.version_info[:2], platform.system(), platform.machine())"
      hash: stdout
```

Pure-Python checks (ruff, mypy, pytest) produce no outputs; the version in `.prototools` is enough.

## Workspace layout

A uv workspace keeps one lockfile and one venv at the root, with each package as a moon project:

```toml
# /pyproject.toml
[tool.uv.workspace]
members = ["services/*", "packages/*"]
```

```yaml
# .moon/workspace.yml
projects:
    - services/*/pyproject.toml
    - packages/*/pyproject.toml
```

- One lockfile means one dependency set: any `uv.lock` change invalidates every Python task. That is correct (any package could be affected) and cheap if tasks are fast.
- Separate projects with their own lockfiles isolate bumps but make shared libraries harder (path dependencies, version skew).
- Do not use a catch-all like `/python/**/*` as the input of every task: one edit anywhere invalidates everything. Each project hashes its own `src` and tests plus the shared lockfile.

## Shared tasks

```yaml
# .moon/tasks/python.yml
inheritedBy:
    languages: [python]

implicitInputs:
    - /.prototools
    - /uv.lock
    - /pyproject.toml

fileGroups:
    src: []
    test-unit: []
    test-int: []
    cfg: []
    cfg-lint-py:
        - /ruff.toml

taskOptions:
    runFromWorkspaceRoot: true

tasks:
    lint:
        command: uv run --locked ruff check --no-fix $projectSource
        inputs: &py-all
            - "@group(src)"
            - "@group(test-unit)"
            - "@group(test-int)"
            - "@group(cfg)"
            - "@group(cfg-lint-py)"

    lint-fix:
        command: uv run --locked ruff check --fix $projectSource
        inputs: *py-all

    fmt:
        command: uv run --locked ruff format $projectSource
        inputs: *py-all
        deps:
            - ~:lint-fix

    typecheck:
        command: uv run --locked mypy $projectSource/src
        inputs: *py-all

    test-unit:
        command: uv run --locked pytest $projectSource/tests/unit -p no:cacheprovider
        inputs:
            - "@group(src)"
            - "@group(test-unit)"
            - "@group(cfg)"

    test-int:
        command: uv run --locked pytest $projectSource/tests/integration -p no:cacheprovider
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
# root moon.yml
tasks:
    setup:
        command: uv sync --locked --all-packages
        options:
            cache: false
    lock:
        command: uv lock
        options:
            cache: false
```

- `--locked` fails when `uv.lock` is out of date with `pyproject.toml`; `--frozen` silently uses the stale lock. Use `--locked` everywhere, `--frozen` only in Docker layers where the project source is not copied yet.
- Ruff, mypy, pyright, ty and pytest find `pyproject.toml`/`ruff.toml` by walking up, so no `--config` with absolute paths (they would end up in the hash).
- `pytest -p no:cacheprovider` stops pytest from writing `.pytest_cache` into the project; it is ignored anyway, but it confuses people reading `git status` inside containers.
- Test selection by folder (`tests/unit`, `tests/integration`) is easier to hash than markers (`-m "not integration"`): the file groups then match exactly what runs.

## Long-running services

```yaml
tasks:
    dev:
        command: uv run --locked uvicorn app.main:create_app --factory --reload
        preset: server
        options:
            runFromWorkspaceRoot: false
            envFile: .env
```

## Docker

uv's own Docker recipe (`uv sync --locked --no-install-project` on the lockfile, then copy sources and sync again) already gives a cached dependency layer. `moon docker scaffold` adds value when one image needs several workspace packages and you do not want to list their `pyproject.toml` files by hand; since v2.5, `moon docker prune` removes `.venv` directories of unfocused projects. See [docker.md](docker.md).

## Notebooks and data tasks

Notebooks (`*.ipynb`) contain outputs and execution counts; hashing them makes every run of the notebook a cache miss. Strip outputs with `nbstripout` before commit, or exclude notebooks from code-quality groups and give them their own uncached tasks. Data pipelines that read from external storage cannot be cached by file inputs at all: use `cache: false`, or a `fingerprint` check over a data version (`dvc status`, a manifest hash).
