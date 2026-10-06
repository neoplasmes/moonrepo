# Observability: webhooks, OpenTelemetry, reports, profiling

Four ways to get data out of moon runs. They answer different questions; most organisations need at most two.

| Mechanism            | Scope                     | Answers                                                         | Setup cost                     |
| -------------------- | ------------------------- | --------------------------------------------------------------- | ------------------------------ |
| Run reports          | one run, local or CI      | what ran, what failed, what was cached                          | none (JSON in `.moon/cache`)   |
| Webhooks             | CI only                   | push events to Slack, a metrics collector, a dashboard          | an HTTPS endpoint you own      |
| OpenTelemetry (v2.5) | any run                   | traces of the whole pipeline, metrics per task (v2.6), cache hit rate | an OTLP collector/backend |
| `--dump`, `--profile` | one run, local           | why moon or one Node task is slow                               | none                           |

## Run reports

- `moon ci` writes `.moon/cache/ciReport.json`; `moon run`/`exec`/`check` write `.moon/cache/runReport.json`. They are overwritten by the next run: upload them as CI artifacts if you want history.
- `moonrepo/run-report-action` turns the report into a PR comment and a workflow summary.
- Cheapest way to start measuring: a CI step that uploads the report and a small job that aggregates durations and cache statuses.

## Webhooks

```yaml
# .moon/workspace.yml
notifier:
    webhookUrl: https://hooks.internal.example.com/moon/8f3c1d   # or MOON_WEBHOOK_URL
    webhookAcknowledge: false
```

- Sent **only in CI environments**, as POST requests with a JSON payload: `type`, `environment` (provider, branch, base branch, PR id/URL, revision, pipeline URL), `event`, `createdAt`, `uuid` (this run batch), `trace` (set `MOON_TRACE_ID` to correlate several moon invocations in one pipeline).
- Events: `pipeline.started|finished|aborted`, `action.started|finished`, `task.running|ran`, `toolchain.installing|installed`, `dependencies.installing|installed`, `project.syncing|synced`, `workspace.syncing|synced`, `environment.initializing|initialized`. `pipeline.finished` carries `cachedCount`, `passedCount`, `failedCount`, `duration`, `baselineDuration` and `estimatedSavings`.
- `pipeline.finished` is not sent when the pipeline crashes; listen to `pipeline.aborted` too.
- `webhookAcknowledge: true` waits for every request and fails on non-2xx. It slows the pipeline down by one round trip per event; leave it off unless the receiver is a gate.
- Payloads are not signed. Treat the URL as a secret: put it in `MOON_WEBHOOK_URL` from CI secrets rather than in `workspace.yml` (which is also in every task hash), use an unguessable path, and validate on the receiver side that `environment.provider`/`revision` look plausible.
- One event per action means hundreds of requests for a big run. Receivers should accept in batches or be a queue, not a chat API: forward only `pipeline.*` to Slack/Discord, everything else to storage.

Typical uses: a Slack message on `pipeline.aborted` or failures on main; a table of `task.ran` durations per target to find slow tasks; `estimatedSavings` from `pipeline.finished` to show what caching saves.

## OpenTelemetry (v2.5+, metrics v2.6+)

```shell
export OTEL_EXPORTER_OTLP_ENDPOINT=https://otel-collector.internal:4318
export OTEL_EXPORTER_OTLP_PROTOCOL=http/protobuf       # or grpc
moon --otel --otel-service-name moon-ci ci              # or MOON_OTEL=true
```

- Traces: the whole pipeline as a span tree (graph building, actions, task execution, plugin calls). The same data as `--dump`, but in Grafana, Honeycomb or Datadog, across runs.
- Metrics (v2.6): `moon.task.runs`, `moon.task.duration` (labels: target, project, task, toolchains, status, flaky), `moon.action.*`, `moon.operation.*`. Cache hit rate = runs with status `cached` or `cached-from-remote` over all runs.
- `--otel-logs` exports log events too, filtered by `--log`.
- Cardinality: `target` labels scale with the number of tasks. With thousands of tasks, aggregate or drop the label in the collector.
- proto has its own OTEL support (v0.58+) for tool installs.

Webhooks or OTEL? If you already run an OTEL backend, OTEL gives more with less code. Webhooks fit when you want custom side effects (notifications, PR comments, writing to your own database) or have no OTEL stack.

## Terminal notifications

```yaml
notifier:
    terminalNotifications: failure     # always | failure | success | task-failure
```

Desktop notifications for local runs (Linux via D-Bus, macOS via the terminal app, Windows 10+). Useful for long local builds; noise for short ones. A personal preference: prefer suggesting it over committing it.

## moon's own profile: `--dump`

`moon run <target> --dump` (or `MOON_DUMP=true`) writes a trace profile of moon itself into the working directory. Open it in `chrome://tracing` or ui.perfetto.dev. Use it when graph building, hashing or hydration is slow, not when the task's own command is slow.

## Task profiling

`moon run --profile cpu <target>` and `moon run --profile heap <target>` record V8 profiles of the task's process:

- Only for Node-based tasks (moon runs the tool through Node with profiling flags). Not for Bun/Deno, not for tasks inferred from `package.json` scripts, not for Go, Rust or Python.
- Output: `.moon/cache/states/<project>/<task>/snapshot.cpuprofile` or `snapshot.heapprofile`. Load it in Chrome DevTools: `chrome://inspect` → "Open dedicated DevTools for Node" → Profiler or Memory tab. Bottom-up view finds the heaviest functions, top-down shows call paths, the flame chart shows time.
- Typical findings: a lint rule or TypeScript plugin dominating `lint`/`typecheck`, a bundler plugin dominating `build`, memory growth in a test runner.

For other languages use their profilers inside the task: `go test -cpuprofile`, `py-spy record -- <cmd>`, `cargo flamegraph`. Make those separate uncached tasks or ad-hoc runs, not flags on the normal task.
