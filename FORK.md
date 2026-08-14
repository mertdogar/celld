# About this fork

`mertdogar/celld`, branch `fork/worker-websockets` — a fork of
[denoland/celld](https://github.com/denoland/celld) at v0.2.0, run by the
`ommaworks/infra` Worker pool as the local stand-in for Cloudflare Workers.
History starts at the v0.0.1 snapshot; upstream is the `upstream` remote.

`celld --version` prints upstream's number (`celld 0.2.0`), so the binary does
not reveal the fork. A config carrying `observability` does: this fork accepts
and ignores platform metadata with a note, stock v0.2.0 refuses the deploy.

## Divergence from v0.2.0, in order

- **WebSockets a stateless Worker can serve** (`f4153c7`) — upstream binds a
  `WebSocketPair` to the host transport only on the Durable Object path; here a
  stateless Worker's `101` binds an ingress pump too, registered as `waitUntil`
  work so the socket outlives the handler's return. Known gap: a `101` crossing
  a service binding still loses its socket.
- **Cloudflare platform metadata accepted and ignored** (`fcbd6d3`) —
  `observability`, `upload_source_maps`, `placement`, … are dropped with a note
  naming them instead of refusing the deploy; every key that changes what a
  Worker can do still fails loudly.
- **Wrangler's `alias`** (`1542f5c`) — exact-match specifier aliases, passed to
  esbuild; refuses absolute replacement paths and `no_bundle`.
- **Unprefixed node builtins** (`19f7674`) — bare `fs`/`path`/`stream/web`
  imports and dependencies' `require("fs")` resolve the way Wrangler's
  `nodejs_compat_v2` resolves them.
- **A real `node:fs` surface** (`05940e3`) — enumerable named exports and
  Node-shaped `ENOENT` (`code`/`errno`/`syscall`); still no filesystem.
- **`node:process` resolves to the installed global** (`5151f46`).
- **`nodejs_compat_populate_process_env`** (`80cf4b9`) — honored with
  Cloudflare's date gate (on by default from 2025-04-01); string `vars` are
  copied into `process.env` before the module evaluates.
- **Multipart upload for objects over 5 MiB** (`c88e7b5`) — a 13.7 MB Worker
  bundle deploys instead of dying on the S3 emulator's single-PUT ceiling;
  reuses the LTX client's 5 MiB threshold.

Fork-visible surface changes land in
[docs/cloudflare-compat.md](docs/cloudflare-compat.md); everything else — the
README included — tracks upstream to keep merges small. The infra repo pins the
pool's build by short SHA (`k8s/base/worker-pool.yaml`, `k8s/base/celld-deploy.yaml`,
`cli/omma.mjs`) and rebuilds via its `k8s/build-celld.sh`.
