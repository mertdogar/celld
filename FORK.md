# About this fork

`mertdogar/celld`, branch `fork/v0.5.1` — a fork of
[denoland/celld](https://github.com/denoland/celld) at **v0.5.1**, run by the
`ommaworks/infra` Worker pool as the local stand-in for Cloudflare Workers.
History starts at the v0.0.1 snapshot; upstream is the `upstream` remote. The
previous fork, `fork/worker-websockets` at v0.3.0, is kept as a backup branch
and is not built any more.

`celld --version` prints upstream's number (`celld 0.5.1`), so the binary does
not reveal the fork. A config carrying `observability` does: this fork accepts
and ignores platform metadata with a note, stock v0.5.1 refuses the deploy.

## Divergence from v0.5.1, in order

Seven commits, cherry-picked from the v0.3.0 fork one at a time with
`cargo check -p celld --locked` after each.

- **Cloudflare platform metadata accepted and ignored** (was `3ba49cb`) —
  `observability`, `upload_source_maps`, `placement`, `workers_dev`,
  `preview_urls`, `routes`/`route`, `account_id`, `dev`,
  `compatibility_domains`, `keep_vars`, `minify` are dropped with a note
  naming them instead of refusing the deploy; every key that changes what a
  Worker can do still fails loudly. Upstream's `SUPPORTED_KEYS` grew
  `define`, `rules`, `worker_loaders`, `containers` since v0.3.0; none of the
  ignored keys became real. The unit test lives beside upstream's
  `deploy_contract` module.
- **Wrangler's `alias`** (was `97b59c7`) — exact-match specifier aliases,
  passed to esbuild as `--alias:` before the fixed flags. Refuses absolute
  replacement paths. Reworked at the v0.5.1 rebase: upstream now carries a
  `BundleConfig` (`define` + `rules`) into `run_esbuild`, so `alias` rides
  beside it as a second argument, and the `no_bundle` refusal names all three
  keys in one message.
- **`nodejs_compat_populate_process_env`** (was `2002eb7`) — honored with
  Cloudflare's date gate (on by default from 2025-04-01); string `vars` are
  copied into `process.env` before the module evaluates. Upstream v0.5.1
  still has no equivalent: `CELLD_VAR_` and `CELLD_VARS_FILE` were removed
  in favour of `vars`, but `vars` reach only `env`, and a dependency reading
  `process.env.X` at module scope sees nothing.
- **SQL turns on cells over the public listener** (was `02fdbaf`) — a
  platform surface at `POST /__celld/sql/<scope>`, gated on `CELLD_SQL_GATE`,
  run as a `CellJob::Sql` turn with the output gate proven before the reply.
  Reworked at the v0.5.1 rebase in three places: `EgressGate::Wrote` now
  carries a `Channel` and the activation epoch, so `SqlTurn` gained `epoch`
  and the gate is awaited on `Channel::Response`; `storage::write_position`
  became fallible (it refuses an unfinished `RETURNING` cursor), so the turn
  fails instead of guessing a position; and the D1-only refusal became
  upstream's `reserved_class_of`, so KV, Queue and Workflow cells are refused
  the same way `/do/` refuses them. `request_payload` takes the node's body
  limit.
- **`node:process` resolves to the installed global** (was `a3df7c6`) —
  re-picked after its drop-verification failed on stock v0.5.1: `import
  proc from "node:process"` there is a `__nodeStubFor` proxy, so
  `proc.env.X` is a proxy too and `proc !== globalThis.process`. The fork
  maps the specifier onto `__celld.process`, which harness.js sets to the
  global right after installing it.
- **Trace ids on the console line** (was `949a9c7`) — `op_log` stamps
  `trace=`/`span=` on its `cell_console` stdout line beside the Parquet log
  row it already writes. Always stamped, `-` when there is no trace. Since
  v0.5.1 the ids are the *recording* ids (`TraceContext::recording_ids`):
  an unsampled trace stamps `-`, matching what the Parquet tier holds.

- **Offline SQLite restore** (was `1c18ecb`) — `celld restore SCOPE
  --bucket …` reads a cell straight from the bucket, takes no lease and
  wakes no Worker. Reworked at the v0.5.1 rebase: v0.4.1 lets a large cell
  hand off through its LTX chain, so the newest epoch need not open with a
  snapshot. The command now lists every non-empty epoch and restores through
  upstream's `EpochChain`, the same read basis the node's own
  `restore_snapshot` uses; `celld dev`'s local store is refused. The report
  line is unchanged: `epoch eN (as-uploaded through txid N)`.

## Dropped at the v0.5.1 rebase

Each of these had become upstream's job. Ticket 01 of the infra map
`.scratch/celld-v0.5.1/` records the live verification for each.

- **WebSockets a stateless Worker can serve** (`6c6de26`) — upstream's
  `worker_websocket_task` carries a top-level Worker's `101` through
  `__ws_next`; the fork's ingress pump and `__bindWorkerSocket` are gone.
- **Multipart upload over 5 MiB** (`0660588`) — upstream writes large
  objects through `object_store` multipart.
- **Unprefixed node builtins and the `node:fs` surface** (`7af8ae8`,
  `684c284`) — v0.4.1 added real `node:fs` and `node:os`, v0.5.0 resolves
  minified builtin imports (#201), v0.5.1 fixes mutable builtins and the
  `process` fields dependencies read at load. Verified: bare `fs` and `path`
  imports resolve and `fs` enumerates. (`node:process` was also dropped, and
  came back — see above.)
- **The operator API behind `CELLD_OPERATOR_GATE`** (`82c2951`) — the
  internal listener is still unauthenticated upstream, but the pool now
  binds each child's `--internal-listen` and `--advertise` to loopback, so
  nothing outside the container can reach it and there is nothing to gate.
- `setInterval` (`a41691d`) and the builtin-list rustfmt (`45e75d2`) were
  already dropped at v0.3.0.

## Downstream

Fork-visible surface changes land in
[docs/cloudflare-compat.md](docs/cloudflare-compat.md); everything else — the
README included — tracks upstream to keep merges small. The infra repo pins
the pool's build by short SHA at three sites — `k8s/base/worker-pool.yaml`,
`k8s/base/celld-deploy.yaml` and `k8s/base/bridge.yaml` — and rebuilds via
`k8s/build-celld.sh`. Bridge's build reads its tag out of
`celld-deploy.yaml`, so that manifest is re-pinned *before* Bridge is built.

What the pool inherits from v0.4.0–v0.5.1 without a fork change, and what it
decided:

- A node **polls `deploy/current.json`** every `CELLD_DEPLOY_POLL_S` (30 s)
  and adopts a new Version in place; `POST /reload` on the internal listener
  forces it. The harness forwards its own `/reload` there instead of
  restarting the child.
- A node with **no Pointer waits** instead of exiting.
- Health moved to **`/.well-known/celld/health`**.
- `CELLD_VAR_*` and `CELLD_VARS_FILE` **refuse to start**; the harness sets
  neither.
- KV, R2, Queues, Workflows, D1 and cron are real and land inside the
  Worker's own prefix. `CELLD_DURABILITY` defaults to `fleet`; a one-child
  fleet has no peers, so acks stay bucket-proven.
- `CELLD_MAX_RSS_MB`, cgroup-aware pressure, `CELLD_V8_HEAP_LIMIT_MB`
  (128 MiB), `CELLD_MAX_REQUEST_BODY_BYTES` (1 GiB), `CELLD_OTEL` (`1` for
  the bucket, or an OTLP URL).
