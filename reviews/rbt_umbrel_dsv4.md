## Blocking issues

1. **docker-compose.yml — `image: ghcr.io/reardencode/rbitcoin-ui:v0.8.0@sha256:PLACEHOLDER` and `image: ghcr.io/reardencode/rbitcoin-node:v0.8.0@sha256:PLACEHOLDER`**
   These digests are invalid and will fail to pull. For local install, reference locally built images:
   ```yaml
   image: rbitcoin-ui:0.8.0
   image: rbitcoin-node:0.8.0
   ```

2. **docker-compose.yml — `ports:` on the `rbitcoin` service publishes host ports**
   Umbrel OS does not allow host port publishing in app compose files. Remove:
   ```yaml
   ports:
     - "${APP_RBITCOIN_ELECTRUM_PORT}:${APP_RBITCOIN_ELECTRUM_PORT}"
   ```
   The node listens on the static IP `$APP_RBITCOIN_NODE_IP`; LAN access is via that IP, not host port mapping.

3. **docker-compose.yml — `app_proxy` `APP_PORT: 3012` but `app` service has no `APP_PORT` env**
   The `app` (UI) container listens on `${RBITCOIN_UI_PORT}` (default 3012), but `app_proxy` forwards to `$APP_RBITCOIN_IP:3012`. The `app` service sets `RBITCOIN_UI_PORT: 3012` in its environment, so this works, but the `app` service must be on `$APP_RBITCOIN_IP` (it is). No fix needed — but verify `app_proxy` `APP_HOST` matches the `app` service IP.

4. **torrc.template — hardcoded ports 50001/3000 instead of `$APP_RBITCOIN_ELECTRUM_PORT`/`$APP_RBITCOIN_ESPLORA_PORT`**
   If the user changes ports via env, Tor hidden services break. Use:
   ```
   HiddenServicePort ${APP_RBITCOIN_ELECTRUM_PORT} $APP_RBITCOIN_NODE_IP:${APP_RBITCOIN_ELECTRUM_PORT}
   HiddenServicePort ${APP_RBITCOIN_ESPLORA_PORT} $APP_RBITCOIN_NODE_IP:${APP_RBITCOIN_ESPLORA_PORT}
   ```

5. **docker-compose.yml — `rbitcoin` service has no `depends_on` for `tor`**
   The node can start without Tor (it doesn't use `--tor-control` by default), so this is not a hard breaker, but the UI page shows Tor onion addresses that won't exist until Tor starts. Add:
   ```yaml
   depends_on:
     - tor
   ```

## Should-fix

1. **docker-compose.yml — `app` service `RBITCOIN_HOST: ${DEVICE_DOMAIN_NAME}`**
   `DEVICE_DOMAIN_NAME` is not an Umbrel-injected variable. Use `APP_RBITCOIN_IP` or `localhost`:
   ```yaml
   RBITCOIN_HOST: ${APP_RBITCOIN_IP}
   ```

2. **docker-compose.yml — `app` service missing `APP_RBITCOIN_ELECTRUM_HIDDEN_SERVICE` and `APP_RBITCOIN_ESPLORA_HIDDEN_SERVICE` env vars**
   These are referenced in the UI template but not set in compose. Umbrel OS injects `APP_<ID>_<SERVICE>_HIDDEN_SERVICE` only if the torrc.template defines a hidden service with that name. The torrc uses `app-$APP_ID-electrum` and `app-$APP_ID-esplora`, so the injected vars are `APP_RBITCOIN_ELECTRUM_HIDDEN_SERVICE` and `APP_RBITCOIN_ESPLORA_HIDDEN_SERVICE`. These are set in the `app` service environment, so this is fine — but verify the torrc `HiddenServiceDir` names match the env var suffixes (`electrum`/`esplora`).

3. **Dockerfile — `USER 1000:1000` but entrypoint writes to `${DATA_DIR}` before `exec`**
   The volume `${APP_DATA_DIR}/data:/data` is mounted at `/data`. The entrypoint runs as UID 1000 and does `mkdir -p "${DATA_DIR}"` and writes the cookie file. If the host directory is owned by root (typical Umbrel mount), this fails. Add `user: "1000:1000"` to the `rbitcoin` service in compose (already present) and ensure the volume is writable by UID 1000. The Fulcrum package handles this by running as `1000:1000` and mounting the data dir; the entrypoint must not assume the dir exists. Add `chown -R 1000:1000 "${DATA_DIR}"` or document that the user must `chown` the host dir.

4. **Dockerfile — `stop_grace_period: 30m` is excessive**
   Fulcrum uses 5m. 30m will delay Umbrel shutdown/restart significantly. Use 5m unless rbitcoin specifically requires longer.

5. **docker-compose.yml — `rbitcoin` service `init: true`**
   This is fine, but the entrypoint already `exec`s the node as PID 1. `init: true` adds tini, which is harmless but unnecessary. Keep or remove.

6. **docker-compose.yml — `app` service `restart: on-failure` but no `init: true`**
   The nginx UI container runs the nginx entrypoint, which handles signals. Fine as-is.

7. **umbrel-app.yml — `port: 3012`**
   This is the UI port via app_proxy. Correct.

8. **umbrel-app.yml — `implements: [electrs]`**
   Correct for an Electrum server app.

9. **Dockerfile — `EXPOSE 50001 3000 8332` in node target**
   These are the default ports, but the entrypoint uses env vars. Harmless.

10. **Dockerfile — `COPY --chmod=0755 <<'EOF' /usr/local/bin/rbitcoin-entrypoint`**
    The heredoc syntax is valid in BuildKit. Fine.

11. **docker-compose.yml — `app` service `RBITCOIN_UI_PORT: 3012`**
    This is set, but the nginx template uses `${RBITCOIN_UI_PORT}` in the `listen` directive. The `envsubst` in the entrypoint script substitutes it. Fine.

12. **torrc.template — RPC not exposed over Tor**
    This is intentional and documented. Fine.

## Deferred-by-design

- Store submission requires gallery images, submitter, submission PR, and digest-pinned images — noted in README TODO.
- `alpine:3.20` and `nginx:alpine` base images not digest-pinned — store requirement.
- `SHA256SUMS.linux-musl` not digest-pinned — store requirement.