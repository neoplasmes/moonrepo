# Code generation with `moon generate`

moon renders templates (a folder with `template.yml` and Tera files) into the repository. Use it for structures the repository creates again and again: a service, a package, a page slice, an ADR, a migration pair. Everything in the feature table was run on moon 2.5.

## When to propose a template

Propose (do not create unasked) when:

- you are about to create a structure that already exists at least twice;
- you just created the second copy of something by hand;
- the user asks for "another one like X".

The proposal names the repeated paths, the variables the template would take, the destination and what it saves:

> `services/` has 6 Go services with the same shape: `cmd/<name>/main.go`, `internal/config`, `moon.yml`, `Dockerfile`. A `go-service` template with `name` and `withDatabase` would create them in one command: `moon generate go-service -- --name billing --with-database`. Capture it?

After the user agrees: create the template, generate one real instance, compare it with a hand-written one, run the repository's checks on it.

## What a template should contain

A template should create as little configuration as possible. Behaviour shared by all projects of a kind belongs in `.moon/tasks/*.yml` (selected by language, toolchain or tag), not in each generated `moon.yml`.

- Generated `moon.yml`: `language`, `layer`, `tags`, file groups, project-only tasks. Shared tasks arrive through inheritance.
- Reason: moon templates are one-shot. There is no `update` that re-applies a changed template to existing projects (unlike `copier update` or cruft). Whatever you copy into a project drifts; whatever you inherit stays current.
- Never generate empty placeholders (empty folders, `index.ts` that exports nothing); the code that needs them creates them.
- A template never edits existing files. Registering the project (`.moon/workspace.yml` map entry, `pnpm-workspace.yaml`, `go.work`, `[tool.uv.workspace]`) stays a manual step listed in the template `description`, unless workspace globs pick new projects up automatically, which is a good reason to use globs.

## Anatomy

```text
templates/
    frontend-widget/
        template.yaml
        widget.tsx.tera
        widget.module.css.tera
        index.ts.tera
```

`template.yaml`:

```yaml
title: Frontend widget
description: Widget slice with its CSS module and barrel.
destination: /apps/frontend/src/ui/widgets/[name | pascal_case]
variables:
    name:
        type: string
        default: ""
        required: true
        prompt: Widget name?
```

`widget.tsx.tera`:

```twig
{% set component = name | pascal_case %}
---
to: {{ component }}.widget.tsx
---
import styles from './{{ name | camel_case }}.widget.module.css';

export function {{ component }}() {
	return <section className={styles.root} />;
}
```

Run: `moon generate frontend-widget --defaults -- --name "order summary"` creates `apps/frontend/src/ui/widgets/OrderSummary/{OrderSummary.widget.tsx, orderSummary.widget.module.css, index.ts}`.

- `.tera` (or `.twig`) keeps editor highlighting and is stripped on output; real file names come from `to:` frontmatter or `[var]` path interpolation.
- A file whose content conflicts with Tera syntax (GitHub Actions `${{ }}`, Helm charts, Jinja) gets a `.raw` extension and is copied verbatim; or wrap the conflicting part in `{% raw %}...{% endraw %}`.
- Files with `partial` in their path are only for `{% include %}` and are never written. Binary assets are copied as is.
- Keep template folder names kebab-case; they are the template ids unless `id` is set.

## Template locations and sharing

```yaml
# .moon/workspace.yml
generator:
    templates:
        - ./templates                                      # default
        - ./templates/*                                    # glob (v1.31+)
        - git://github.com/acme/moon-templates#v3.2.0      # pinned tag (v1.23+)
        - npm://@acme/moon-templates#3.2.0                 # pinned package (v1.23+)
        - https://example.com/templates.tar.gz             # archive (v1.36+)
```

- Locations are searched in order; the first match by template id wins. Local templates can shadow shared ones on purpose.
- Pin `git://` and `npm://` locations to a tag or version. Remote templates are cached under `~/.moon/templates/`.
- A platform team publishing "golden path" templates in a separate repository gets one source of truth for many repositories. The cost: every consumer pins a version and has to bump it; changes to templates do not reach already generated projects.

## Variables

| Type      | Notes                                                                                     |
| --------- | ----------------------------------------------------------------------------------------- |
| `string`, `number`, `boolean` | `default` required; `prompt` makes it interactive; `required` rejects empty values |
| `enum`    | `values` list, optional `multiple: true` (renders an array)                                |
| `array`, `object` | JSON values; objects cannot be set from the command line                          |
| `internal: true` | computed or fixed values that the CLI must not override                            |
| `order`   | prompt order                                                                              |

Name variables in camelCase. Prefer `enum` for closed choices and `boolean` for optional files (rendered with `skip:` frontmatter). Command-line names match variable names: `--name x`, `--with-database` / `--no-with-database` for booleans.

## Running

```shell
moon templates                                   # list
moon template go-service                         # files and variables
moon generate go-service --to services/billing --defaults --dry-run -- --name billing
moon generate go-service --to services/billing -- --name billing
moon generate my-new-template --template         # scaffold a new template folder
```

- Always `--dry-run` first, into the real destination: it shows conflicts with existing files.
- `--defaults` skips prompts, `--force` overwrites. Agents and CI must pass both variables and `--defaults`, because prompts block.
- The moon MCP server exposes template discovery (`get_templates`, `get_template`, v2.3+).

## Tera and moon features

| Feature                          | Syntax                                                         |
| -------------------------------- | -------------------------------------------------------------- |
| Variable                         | `{{ name }}`                                                   |
| Case filters (moon)              | `pascal_case`, `camel_case`, `kebab_case`, `snake_case`, `upper_snake_case`, `upper_kebab_case`, `lower_case`, `upper_case`; function `variables()` |
| Path filters (moon)              | `path_join(part = "x")`, `path_relative(from = other)`         |
| Interpolation in paths           | `[name \| pascal_case]` in `destination` or file names          |
| Rename the output file           | frontmatter `to: {{ name \| pascal_case }}.page.tsx`            |
| Conditional file                 | frontmatter `skip: {{ not withModel }}`                        |
| Overwrite without prompting      | frontmatter `force: true`                                      |
| Current date                     | `{{ now() \| date(format="%Y-%m-%d") }}` (checked)              |
| Inherit another template         | `extends: [base]` in `template.yaml`                           |
| Shared fragments                 | any file with `partial` in its path, used with `{% include %}` |
| Always-available variables       | `dest_dir`, `dest_rel_dir`, `working_dir`, `workspace_root`    |

Commands: `moon templates` lists templates, `moon template <id>` shows files and variables, `moon generate <id> --to <dir> --defaults --dry-run -- --var value` renders without writing. Boolean variables accept `--flag` and `--no-flag`. The moon MCP server exposes `get_templates` and `get_template` for agents.

## Alternatives

| Tool                      | Pick it when                                                                        |
| ------------------------- | ----------------------------------------------------------------------------------- |
| `moon generate`           | templates live with the workspace, variables are simple, no update story needed     |
| copier / cruft            | generated projects must receive template updates later (`copier update`)            |
| Backstage scaffolder, internal portals | creation also provisions things outside the repo (repos, CI, cloud accounts) |
| Language generators (`go generate`, `sqlc`, `openapi-generator`, `protoc`) | output is derived from a schema and regenerated on every change: these are cached moon tasks with outputs, not templates |
