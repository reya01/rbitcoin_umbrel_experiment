## Blocking issues (would fail to install or start)

**1. `docker-compose.yml` — both images are `ghcr.io/reardencode/rbitcoin-(node|ui):v0.8.0@sha256:PLACEHOLDER`**
`sha256:PLACEHOLDER` is not a valid digest, so `docker compose pull/config` fails with an invalid-reference error and the install aborts. Even without the digest, nothing is pushed to ghcr yet (per your own TODOs), and the README's local build tags (`rbitcoin-node:0.8.0`, `rbitcoin-ui:0.8.0`) don't match the refs compose uses — so a successful local build would be ignored anyway.
**Fix:** for direct install, set `image: rbitcoin-node:0.8.0` and `image: rbitcoin-ui:0.8.0` (exactly the README build tags) and build on the Umbrel host *before* installing. Registry push + real digest pinning stays on the store TODO.

## Should-fix (likely breakage or UX harm on a real install)

**1. `torrc.template` — hidden services forward to hardcoded `50001`/`3000`, but the node listens on OS-assigned ports.**
Umbrel OS assigns `APP_RBITCOIN_ELECTRUM_PORT` / `APP_RBITCOIN_ESPLORA_PORT` dynamically (same mechanism as `APP_FULCRUM_NODE_PORT` — it is *not* 50001). Compose passes those to the node via `RBITCOIN_*_PORT`, so the node listens on the assigned ports while both onions forward to closed 50001/3000 → Electrum-over-Tor and Esplora-over-Tor are dead, and the UI's `onion:port` display also disagrees with the hardcoded virtual port.
**Fix** (mirrors Fulcrum's torrc exactly):
```
HiddenServicePort $APP_RBITCOIN_ELECTRUM_PORT $APP_RBITCOIN_NODE_IP:$APP_RBITCOIN_ELECTRUM_PORT
HiddenServicePort $APP_RBITCOIN_ESPLORA_PORT $APP_RBITCOIN_NODE_IP:$APP_RBITCOIN_ESPLORA_PORT
```
Also update the header comment — the OS substitutes the `*_PORT` vars too, not just `$APP_ID`/`$APP_RBITCOIN_NODE_IP`.

**2. `docker-compose.yml` — Esplora is not host-published, but the UI and README advertise LAN access.**
Only the Electrum port has a `ports:` mapping. `http://<umbrel>:<esplora-port>` (the advertised BDK/Alby Hub path) reaches nothing.
**Fix:** add to `rbitcoin.ports`: `- "${APP_RBITCOIN_ESPLORA_PORT}:${APP_RBITCOIN_ESPLORA_PORT}"`.

**3. `docker-compose.yml` — RPC is not host-published at all, but the UI/README promise `127.0.0.1:<rpc-port>` on the host.**
The node listens on `0.0.0.0:8332` *inside* the container; with no mapping, host localhost (Wasabi/LND/CLN per the docs) can't reach it.
**Fix:** add `- "127.0.0.1:${APP_RBITCOIN_RPC_PORT}:${APP_RBITCOIN_RPC_PORT}"`. Bind 127.0.0.1, not 0.0.0.0 — that matches the "localhost only, cookie auth" messaging.

**4. `Dockerfile` — x86_64-only binaries, no architecture guard.**
v0.8.0 ships linux-musl x86_64 only; on an arm64 Umbrel (Raspberry Pi) the node container crash-loops with `exec format error`. The README mentions it, but nothing fails loudly at runtime.
**Fix:** confirm the target host is x86_64; add before the `exec` in the entrypoint: `[ "$(uname -m)" = "x86_64" ] || { echo "rbitcoin v0.8.0: x86_64-linux-musl only" >&2; exit 1; }`.

**5. `README.md` — "Test" section documents fixed host ports 50001/3000/8332.**
On a real install all three are OS-assigned (`APP_RBITCOIN_*_PORT`), and 3000/8332 aren't even published until fixes 2–3 land.
**Fix:** drop the fixed ports; tell the user to copy the actual host:port combinations from the app's UI page.

**6. Minor first-boot verifications for the smoke test:**
- a) The UI bakes the onion addresses into static HTML at container start. Umbrel OS starts tor first and injects `APP_RBITCOIN_*_HIDDEN_SERVICE` before the rest (same mechanism Fulcrum relies on), but if the page ever renders "not available yet" it stays that way until the app is restarted — the static render can't recover.
- b) uid 1000 writing `/data/rbitcoin` (mkdir + cookie) relies on the OS pre-creating `${APP_DATA_DIR}/data` writable by 1000 — same assumption as the working Fulcrum/Electrs packages. If the node crash-loops on `mkdir`/cookie write, pre-create on the host: `mkdir -p ~/umbrel/app-data/rbitcoin/data/rbitcoin && chown -R 1000:1000 ~/umbrel/app-data/rbitcoin/data`.
- c) Confirm the release asset names (`rbitcoin-node-x86_64-linux`, `rbitcoin-cli-x86_64-linux`) and the 17,064,208-byte pin before building — a mismatch aborts the build loudly (fail-closed, by design), but it's the first thing to check if the build fails.

## Deferred-by-design (store requirements — noted, not counted)
- Push both images to a registry and pin real `@sha256:` digests (compose TODOs already say this).
- Pin base images (`alpine:3.20`, `nginx:alpine`) by digest; the tor sidecar is already pinned.
- `gallery:` files (1.jpg–3.jpg) don't exist in the package; `submitter`/`submission` absent — store-only fields.
- Optional stronger supply-chain pin: digest-pin `SHA256SUMS.linux-musl` itself.
- Full on-device install + IBD smoke test before submitting to getumbrel/umbrel-apps.