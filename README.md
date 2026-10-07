# moonrepo

[![skills.sh](https://skills.sh/b/neoplasmes/moonrepo)](https://skills.sh/neoplasmes/moonrepo)

An [agent skill](https://agentskills.io) for working with [moon](https://moonrepo.dev) (moonrepo): task running, caching, CI, remote cache, Docker and code generation in monorepos. It is language-agnostic, with detailed guides for **TypeScript/JavaScript**, **Go** and **Python**, and shorter ones for Rust, Terraform and others.

It works with Claude Code, Codex, Cursor, OpenCode and the other agents supported by the [skills CLI](https://github.com/vercel-labs/skills).

## Install

```bash
pnpx skills add neoplasmes/moonrepo
# or
npx skills add neoplasmes/moonrepo
```

Non-interactive, for Claude Code, in your user directory:

```bash
npx skills add neoplasmes/moonrepo --skill moonrepo -a claude-code -g -y
```

Update later with `npx skills update moonrepo`.

## What is inside

The agent reads `SKILL.md` (about 150 lines: the mental model, rules and a table of references) and opens a reference file only when the task needs it.

| Reference | Topic |
| --- | --- |
| `patterns.md`, `decomposition.md` | file groups and globs, task inheritance, deps versus inputs |
| `workspace.md` | projects, toolchains and proto, constraints, shared config, daemon, MCP, VCS hooks |
| `lang-typescript.md`, `lang-go.md`, `lang-python.md`, `lang-other.md` | per-language tasks, hashing pitfalls, Docker notes |
| `caching.md` | what is hashed, outputs, CI cache persistence, remote cache, who may write to it |
| `ci.md` | `moon ci`, affected detection, merge queues, sharding, execution plans |
| `docker.md` | `moon docker scaffold/setup/prune`, when it pays off, Windows and WSL2 |
| `debugging.md`, `observability.md` | `moon hash` diffs, trace logs, webhooks, OpenTelemetry, profiling |
| `codegen.md`, `wasm-plugins.md` | `moon generate` templates, when WASM plugins are worth it |
| `migration.md`, `scale.md` | porting from scripts, Nx and Turborepo; large-monorepo trade-offs |

The skill keeps rules, recipes and verified gotchas, and sends the agent to moon's documentation (raw MDX on GitHub, `llms.txt`) and CLI (`--help`, `moon toolchain info`, `moon task --json`) for reference details, so it does not go stale with every moon release. Claims marked "(verified)" were reproduced on a real workspace with the version stated; the pnpm Docker recipe was built and run end to end.

## Repository layout

```text
skills/moonrepo/        the skill that gets installed (SKILL.md and references/)
.prototools             pins the dev tooling (cocogitto)
.proto/plugins/         proto plugin for cocogitto
cog.toml                conventional commit and changelog config
```

Only `skills/moonrepo/` is installed into your agent; the rest is repository tooling.

## Contributing

Tools come from [proto](https://moonrepo.dev/proto):

```bash
proto install        # installs cocogitto from .prototools
cog commit feat "add a section about X"
cog verify "$(git log -1 --format=%B)"
```

Commits follow [Conventional Commits](https://www.conventionalcommits.org) and are created with `cog commit`.

## License

Free to use, including in commercial projects, but you may not sell it or present it as your own. See [LICENSE](LICENSE). This is a custom source-available license, not an OSI open-source license.
