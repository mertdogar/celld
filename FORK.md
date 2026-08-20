# About this fork

`mertdogar/celld`, branch `fork/worker-websockets` — a fork of
[denoland/celld](https://github.com/denoland/celld) at **v0.3.0**, run by the
`ommaworks/infra` Worker pool as the local stand-in for Cloudflare Workers.
History starts at the v0.0.1 snapshot; upstream is the `upstream` remote.

`celld --version` prints upstream's number (`celld 0.3.0`), so the binary does
not reveal the fork. A config carrying `observability` does: this fork accepts
and ignores platform metadata with a note, stock v0.3.0 refuses the deploy.

## Divergence from v0.3.0, in order

- **WebSockets a stateless Worker can serve** (`6c6de26`) — upstream's v0.3.0
  work joins sockets whose far end is a cell (Durable Object subrequests, and
  `wsTarget` carried across service bindings); a stateless Worker's own `101`
  still reaches the client bare. Here it binds an ingress pump too, registered
  as `waitUntil` work so the socket outlives the handler's return. New since
  v0.3.0: `__bindWorkerSocket` skips a pair whose `_loopback` link is live —
  upstream's same-isolate path already delivers those, and binding the host
  too would give one pair two delivery paths. Known gap, narrower now: a `101`
  that a *Worker* serves still loses its socket across a service binding
  (cell-owned sockets cross fine).
- **Cloudflare platform metadata accepted and ignored** (`3ba49cb`) —
  `observability`, `upload_source_maps`, `placement`, … are dropped with a
  note naming them instead of refusing the deploy; every key that changes what
  a Worker can do still fails loudly. None of the eleven ignored keys became a
  real feature in v0.3.0 (`triggers` did, and was never in the list).
- **Wrangler's `alias`** (`97b59c7`) — exact-match specifier aliases, passed
  to esbuild; refuses absolute replacement paths and `no_bundle`.
- **Unprefixed node builtins** (`7af8ae8`) — bare `fs`/`path`/`stream/web`
  imports and dependencies' `require("fs")` resolve the way Wrangler's
  `nodejs_compat_v2` resolves them. Reworked at the v0.3.0 rebase: upstream
  now externalizes bare builtins (via its own `BARE_NODE_BUILTINS`, which this
  fork extends with ten submodule spellings — esbuild's `--alias:` does not
  match subpaths the way `--external:` does), but externalizing alone leaves a
  dependency's `require("fs")` throwing at module evaluation, so the fork's
  CJS shim machinery (`write_builtin_shims`) survives on top of upstream's
  list.
- **A real `node:fs` surface** (`684c284`) — enumerable named exports and
  Node-shaped `ENOENT` (`code`/`errno`/`syscall`); still no filesystem.
  Load-bearing for the shims above: a CJS-routed `import * as fs` enumerates
  through this proxy or gets an empty namespace.
- **`node:process` resolves to the installed global** (`a3df7c6`).
- **`nodejs_compat_populate_process_env`** (`2002eb7`) — honored with
  Cloudflare's date gate (on by default from 2025-04-01); string `vars` are
  copied into `process.env` before the module evaluates.
- **Multipart upload for objects over 5 MiB** (`0660588`) — a 13.7 MB Worker
  bundle deploys instead of dying on the S3 emulator's single-PUT ceiling;
  reuses the LTX client's 5 MiB threshold. Since v0.3.0 this also covers the
  write-behind log tier's bundle flush, which uploads every dirty cell's L0
  segments through the same `Bucket::put` with no size cap of its own.
- **Docs caught up to the fork** (`0de5712`) — this file, and
  [docs/cloudflare-compat.md](docs/cloudflare-compat.md).
- **SQL turns on cells over the public listener** (`02fdbaf`) — a platform
  surface at `POST /__celld/sql/<scope>`, gated on `CELLD_SQL_GATE`, run as a
  `CellJob::Sql` turn with the output gate proven before the reply. It
  resolves through `RuntimeManager::cell_scope`, so it inherits upstream's
  `valid_cell_scope` charset gate. New at the v0.3.0 rebase: a D1 scope is
  refused with 403 — upstream walls D1 off from every non-HMAC path (its
  `/do/` route refuses them; `/__d1/` is signed), and this gate is a shared
  secret, not that signature, so it inherits the refusal rather than widening
  the D1 surface.
- **Offline SQLite restore** (`1c18ecb`) — a one-shot read-only
  `celld restore SCOPE --bucket …` that reads the highest non-empty epoch
  straight from the bucket, takes no lease and wakes no Worker. Reworked at
  the v0.3.0 rebase: upstream deleted the epoch-seal protocol (its removal
  comment in `activate()` argues a permanent cap turned ordering slips into
  permanent loss), so the command's seal half went with it, and an `az://`
  bucket is accepted. Caveat, printed in its help: on a fleet of two or more
  nodes a write acknowledged after peer fsync may not have reached the bucket
  yet, so the output can trail the fleet's acknowledged truth; a single node
  acknowledges only bucket-proven writes. Upstream's own in-process
  `LtxRepl::restore_snapshot` uses the same read basis.
- **Trace ids on the console line** (`949a9c7`) — `op_log` hoists
  `current_trace_ids` out of the telemetry guard so its `cell_console` stdout
  line carries `trace=`/`span=` beside the Parquet log row it already writes.
  Always stamped, `-` when there is no trace. `telemetry.rs` and the Parquet
  schema stay byte-identical to upstream (`op_log` itself was untouched by
  v0.3.0).
- **The operator API behind `CELLD_OPERATOR_GATE`** (`82c2951`) — upstream's
  internal listener serves the unauthenticated operator routes (`/state`,
  `POST /shutdown`, `/do/<cell>`, `/cell/<id>`, `/evict/<cell>`) on the same
  address peers forward to, and tells you to firewall it. Under the pool that
  address is the pod IP, which puts every child's operator API one `fetch()`
  away from every other pod and from tenant JS on the same pod. So the fork
  gates every non-peer path on the internal listener: unset → 503, set →
  `x-celld-operator-gate` must match, else 403. Peer paths keep their HMAC
  and are untouched. v0.3.0's two new internal routes (`/__log/append|seal|
  tail` for the write-behind log, `/__d1/<scope>`) are both `/__`-prefixed
  and both HMAC-signed, so the gate's rule still covers exactly the
  unauthenticated set — upstream's own `/do/` comment ("This route has no
  authentication") now corroborates the threat model.

## What the v0.3.0 rebase required

v0.3.0 landed as one squashed commit (97 files, +18,642/−5,262): the
replicated write-behind log, D1, cron triggers, Azure Blob Storage, a
per-isolate V8 heap limit, and `main.rs` dissolved into `actor.rs` and
friends. Fifteen of the sixteen files the fork touches changed under it.
Two fork commits died, seven needed conflict resolution, and four compile
breaks surfaced with no conflict marker at all.

**Dropped in upstream's favour, deliberately.** `setInterval` (`a41691d`):
upstream's implementation is better than the fork's — it arms the next round
*before* invoking the callback, so a throwing callback no longer kills the
interval; it allocates a fresh host op per round with identity-tested
cancellation; and `clearTimeout`/`clearInterval` are one function because
callers cross them. The rustfmt commit over the fork's builtin list
(`45e75d2`) went with it, since the fork now extends upstream's list instead
of hoisting its own.

**The websocket rule held.** *Do not re-inline a read loop into a
`tokio::select!`* — `fastwebsockets::read_frame` is not cancel-safe, v0.2.1
existed to remove exactly that fault, and the fork's footprint in
`pump_cell_socket`'s caller stays the two `deliver_*` calls plus the
ingress-queue teardown branch. Upstream inserted a close-flush wait
(`ws_await_flushes`) between those hunks; it is harmless here because a
Worker ingress socket is never hibernatable, so the wait returns on its
first check.

**The four marker-less breaks**, all caught by rebasing with
`git rebase --exec 'cargo check -p celld --locked'`:

- `ws_registry()` now returns `Arc<Mutex<WsRegistry>>` by value, so
  `op_ws_accept_worker`'s single-statement lock was E0716 (temporary freed
  while borrowed) — the two-statement form upstream uses everywhere fixes it.
- `await_egress_gate` was re-typed from `Option<(String, u64)>` to the new
  `EgressGate` enum (upstream's issue-#144 fix), so `await_sql_gate` now
  passes `EgressGate::Wrote(scope, position)`.
- `StorageBackend` gained `Azure`, making `run_restore`'s backend match
  non-exhaustive (E0004) — it gained an `azure_replica_store` arm.
- Upstream removed `use std::time::Instant` from `ltx_repl.rs` under its new
  `disallowed_methods` regime; `run_restore` is an offline operator path
  outside the World, so it carries the explicit allow that convention
  prescribes.

**Conflicts of note.** `turn_begin_cell` gained `recover_heap` on the exact
line the SQL interception occupies — the SQL turn early-returns first, since
it never enters V8. The `live_cas` test module upstream deleted came back as
`live_bucket` carrying only the multipart round-trip test (the only live
coverage of the fork-only `put_body`); the CAS probe test stays deleted, as
upstream decided — `probe_cas` itself survives upstream and is still
exercised by `fleet.rs`. Everything else was anchor-sharing: upstream put
`D1_CLASS`/`read_crons`/`celld d1` in the same slots the fork put
`IGNORED_PLATFORM_KEYS`/`read_alias`/`celld restore`, and both sides now
coexist.

## Downstream

Fork-visible surface changes land in
[docs/cloudflare-compat.md](docs/cloudflare-compat.md); everything else — the
README included — tracks upstream to keep merges small. The infra repo pins
the pool's build by short SHA at three sites — `k8s/base/worker-pool.yaml`,
`k8s/base/celld-deploy.yaml` and `k8s/base/bridge.yaml` — and rebuilds via
`k8s/build-celld.sh`. (`cli/omma.mjs` used to be a fourth and no longer
carries a celld image: deploys go through Bridge.) Bridge's build reads its
tag out of `celld-deploy.yaml`, so that manifest is re-pinned *before* Bridge
is built.

v0.3.0 defaults the pool must decide on rather than inherit blindly, joining
the two from v0.2.1 (`CELLD_STORAGE_PROBE`, on by default;
`CELLD_TRUST_FORWARDED_HEADERS`, off by default):

- `CELLD_DURABILITY` now defaults to `fleet` — the write-behind log,
  acknowledging after peer fsync. A one-node pool has no peers and keeps
  bucket-proven acks, so today's behavior is unchanged, but the default
  activates the moment a second node joins. `bucket` is the explicit opt-out.
- `CELLD_V8_HEAP_LIMIT_MB` — per-isolate heap limit, default 128 MiB. Near
  the limit celld refuses new hibernatable WebSockets and oversized SQL
  materialization, then resumes; decide whether Mastra's isolates need more.
- D1 (`d1_databases` + `celld d1`) and cron `triggers` are now real features
  a tenant config can enable — decide whether the pool exposes them, and
  note the fork's SQL surface refuses D1 scopes either way.
- A v0.2.1 fleet can roll forward one node at a time, per upstream's notes.
