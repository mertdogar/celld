# About this fork

`mertdogar/celld`, branch `fork/worker-websockets` — a fork of
[denoland/celld](https://github.com/denoland/celld) at **v0.2.1**, run by the
`ommaworks/infra` Worker pool as the local stand-in for Cloudflare Workers.
History starts at the v0.0.1 snapshot; upstream is the `upstream` remote.

`celld --version` prints upstream's number (`celld 0.2.1`), so the binary does
not reveal the fork. A config carrying `observability` does: this fork accepts
and ignores platform metadata with a note, stock v0.2.1 refuses the deploy.

## Divergence from v0.2.1, in order

- **WebSockets a stateless Worker can serve** (`31a2d24`) — upstream binds a
  `WebSocketPair` to the host transport only on the Durable Object path; here a
  stateless Worker's `101` binds an ingress pump too, registered as `waitUntil`
  work so the socket outlives the handler's return. Known gap: a `101` crossing
  a service binding still loses its socket.
- **Cloudflare platform metadata accepted and ignored** (`0d52849`) —
  `observability`, `upload_source_maps`, `placement`, … are dropped with a note
  naming them instead of refusing the deploy; every key that changes what a
  Worker can do still fails loudly.
- **Wrangler's `alias`** (`16a0218`) — exact-match specifier aliases, passed to
  esbuild; refuses absolute replacement paths and `no_bundle`.
- **Unprefixed node builtins** (`b6084ce`, formatting in `45e75d2`) — bare
  `fs`/`path`/`stream/web` imports and dependencies' `require("fs")` resolve the
  way Wrangler's `nodejs_compat_v2` resolves them.
- **A real `node:fs` surface** (`e71e279`) — enumerable named exports and
  Node-shaped `ENOENT` (`code`/`errno`/`syscall`); still no filesystem.
- **`node:process` resolves to the installed global** (`efcd82a`).
- **`nodejs_compat_populate_process_env`** (`cdebf15`) — honored with
  Cloudflare's date gate (on by default from 2025-04-01); string `vars` are
  copied into `process.env` before the module evaluates.
- **Multipart upload for objects over 5 MiB** (`d27e8d7`) — a 13.7 MB Worker
  bundle deploys instead of dying on the S3 emulator's single-PUT ceiling;
  reuses the LTX client's 5 MiB threshold.
- **Docs caught up to the fork** (`cfc4e79`) — this file, and
  [docs/cloudflare-compat.md](docs/cloudflare-compat.md).
- **`setInterval` on the existing timer machinery** (`a41691d`) — the throwing
  stub and the no-op `clearInterval` become a real re-arming timer, which is
  what lets Mastra's hourly catalog refresh run.
- **SQL turns on cells over the public listener** (`fb6bc55`) — a platform
  surface at `POST /__celld/sql/<scope>`, gated on `CELLD_SQL_GATE`, run as a
  `CellJob::Sql` turn with the output gate proven before the reply. Since
  v0.2.1 it resolves through `RuntimeManager::cell_scope`, so it inherits
  upstream's `valid_cell_scope` charset gate and answers 400 for a cell id
  outside `[A-Za-z0-9_\-.:$]` or over 512 bytes.
- **Offline SQLite restore** (`079ba14`) — a one-shot read-only
  `celld restore SCOPE --bucket …` that reads the highest non-empty epoch
  straight from the bucket, takes no lease, writes no seal and wakes no Worker.

## What the v0.2.1 rebase required

Only two files conflicted, and one of them matters.

**`main/websocket.rs` — resolved in upstream's favour, deliberately.** v0.2.1
deleted every read loop that lived inside a `tokio::select!`, because
`fastwebsockets::read_frame` is not cancel-safe: dropping the future leaves its
buffer advanced past a consumed frame header and the stream never realigns.
The fork's `websocket_task` **was** such a loop, so keeping the fork's side
would have re-introduced the exact fault the release fixes. The fork is now
re-applied on top of upstream's `pump_cell_socket`, and its whole footprint in
that function is two calls — `deliver_ws_message` and `deliver_ws_closed` —
plus the teardown branch that unregisters an ingress queue instead of settling
cell bookkeeping. All Worker-vs-cell routing lives in those two `deliver_*`
helpers and in `js/websocket.rs`, which upstream does not touch.

*Do not re-inline that loop on the next rebase.* If `pump_cell_socket` changes
shape again, re-apply the two `deliver_*` calls onto whatever upstream ships.

**`bucket.rs`** — both sides refactored the same live test. Resolved by keeping
upstream's `probe_cas()` assertion **and** the fork's
`put_multipart_round_trips_against_real_bucket`, which is the only live check
that the store accepts `object_store`'s fixed-size parts and preserves user
metadata through `CompleteMultipartUpload`.

**One silent break, no conflict**: v0.2.1 gave `request_payload` a second
parameter (`trust_forwarded_headers`) and fixed its own call sites; the fork's
`handle_cell_sql` adds another that merges cleanly and fails to compile. Rebase
with `git rebase --exec 'cargo check -p celld --locked'` so this class of break
surfaces at the commit that causes it.

## Downstream

Fork-visible surface changes land in
[docs/cloudflare-compat.md](docs/cloudflare-compat.md); everything else — the
README included — tracks upstream to keep merges small. The infra repo pins the
pool's build by short SHA at four sites — `k8s/base/worker-pool.yaml`,
`k8s/base/celld-deploy.yaml`, `k8s/base/bridge.yaml` and `cli/omma.mjs` — and
rebuilds via `k8s/build-celld.sh`. Bridge's build reads its tag out of
`celld-deploy.yaml`, so that manifest is re-pinned *before* Bridge is built.

Two v0.2.1 defaults the pool must decide on rather than inherit blindly:
`CELLD_STORAGE_PROBE` (on by default; refuses to serve on a conditional-write
violation) and `CELLD_TRUST_FORWARDED_HEADERS` (off by default, which flips
`request.url` from `https:` to `http:` behind a TLS-terminating edge).
