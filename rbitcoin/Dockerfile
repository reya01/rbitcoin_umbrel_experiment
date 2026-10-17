# syntax=docker/dockerfile:1
#
# rbitcoin v0.8.0 — https://github.com/reardencode/rbitcoin
#
# One Dockerfile, two build targets (referenced by docker-compose.yml):
#   node : the rbitcoin full node — upstream v0.8.0 release binaries, verified
#          fail-closed against SHA256SUMS.linux-musl, on alpine:3.20
#   ui   : tiny nginx page showing connection info, rendered from env at start
#
# Build:
#   docker build --target node -t rbitcoin-node:0.8.0 .
#   docker build --target ui   -t rbitcoin-ui:0.8.0 .

ARG RBITCOIN_VERSION=v0.8.0
ARG RELEASE_BASE=https://github.com/reardencode/rbitcoin/releases/download

##############################################
# Stage 1 — download + verify release binaries (fail closed)
##############################################
FROM alpine:3.20 AS fetch
ARG RBITCOIN_VERSION
ARG RELEASE_BASE
RUN apk add --no-cache curl
WORKDIR /release
RUN <<EOF
set -eux
base="${RELEASE_BASE}/${RBITCOIN_VERSION}"
curl -fsSLO "${base}/rbitcoin-node-x86_64-linux"
curl -fsSLO "${base}/rbitcoin-cli-x86_64-linux"
curl -fsSLO "${base}/SHA256SUMS.linux-musl"
# Extra pin: known size of the v0.8.0 node binary.
[ "$(stat -c %s rbitcoin-node-x86_64-linux)" = "17064208" ]
# The sums file names binaries without the platform suffix — rename to match.
cp rbitcoin-node-x86_64-linux rbitcoin-node
cp rbitcoin-cli-x86_64-linux rbitcoin-cli
# Fail-closed verification: the sums file is "<hash>  <short name>" — require
# exactly the two musl lines, then sha256sum -c must pass. Any mismatch aborts.
[ "$(grep -cE '^[0-9a-f]{64}  rbitcoin-(node|cli)$' SHA256SUMS.linux-musl)" = "2" ]
grep -E '^[0-9a-f]{64}  rbitcoin-(node|cli)$' SHA256SUMS.linux-musl > SHA256SUMS
sha256sum -c SHA256SUMS
mkdir -p /out
install -m 0755 rbitcoin-node /out/rbitcoin-node
install -m 0755 rbitcoin-cli /out/rbitcoin-cli
EOF

##############################################
# Target: node — rbitcoin runtime
# alpine:3.20 is used only for the POSIX shell entrypoint; the release
# binaries are statically linked (musl) and need no other packages.
##############################################
FROM alpine:3.20 AS node
LABEL org.opencontainers.image.title="rbitcoin" \
      org.opencontainers.image.version="0.8.0" \
      org.opencontainers.image.description="Full Bitcoin node in Rust with built-in Electrum, Esplora and Core-compatible RPC servers" \
      org.opencontainers.image.source="https://github.com/reardencode/rbitcoin" \
      org.opencontainers.image.licenses="Apache-2.0 OR MIT"

COPY --from=fetch /out/rbitcoin-node /usr/local/bin/rbitcoin-node
COPY --from=fetch /out/rbitcoin-cli /usr/local/bin/rbitcoin-cli

COPY --chmod=0755 <<'EOF' /usr/local/bin/rbitcoin-entrypoint
#!/bin/sh
set -eu

DATA_DIR="${RBITCOIN_DATA_DIR:-/data/rbitcoin}"
ELECTRUM_PORT="${RBITCOIN_ELECTRUM_PORT:-50001}"
ESPLORA_PORT="${RBITCOIN_ESPLORA_PORT:-3000}"
RPC_PORT="${RBITCOIN_RPC_PORT:-8332}"
COOKIE_FILE="${RBITCOIN_RPC_COOKIE_FILE:-${DATA_DIR}/.cookie}"

mkdir -p "${DATA_DIR}"

# Core-style RPC cookie: a single "user:pass" line with NO trailing newline
# (rbitcoin refuses to start if the file has one). Regenerated on every start.
RPC_USER="${RBITCOIN_RPC_USER:-__cookie__}"
RPC_PASS="$(od -An -N32 -tx1 /dev/urandom | tr -d ' \n')"
printf '%s:%s' "${RPC_USER}" "${RPC_PASS}" > "${COOKIE_FILE}"
chmod 600 "${COOKIE_FILE}"

echo "rbitcoin: datadir ${DATA_DIR}; electrum :${ELECTRUM_PORT}, esplora :${ESPLORA_PORT}, rpc :${RPC_PORT}"

# Indexes: --sp-tweaks (silent payments) and --block-filter-index (BIP158,
# needed by Wasabi) are built during IBD.
set -- rbitcoin-node \
  --datadir "${DATA_DIR}" \
  --network mainnet \
  --sh-index \
  --sp-tweaks \
  --block-filter-index \
  --electrum-listen "0.0.0.0:${ELECTRUM_PORT}" \
  --esplora-listen "0.0.0.0:${ESPLORA_PORT}" \
  --rpc-listen "0.0.0.0:${RPC_PORT}" \
  --rpc-cookie-file "${COOKIE_FILE}"

# Optional: set RBITCOIN_TOR_CONTROL to a Tor control endpoint so the node
# publishes its own onion for the Electrum port (--tor-control). Off by
# default: the app-level tor hidden services already provide onion access.
if [ -n "${RBITCOIN_TOR_CONTROL:-}" ]; then
  set -- "$@" --tor-control "${RBITCOIN_TOR_CONTROL}"
fi

# v0.8.0 ships x86_64-linux-musl binaries only — fail loudly on other arches.
[ "$(uname -m)" = "x86_64" ] || { echo "rbitcoin v0.8.0: x86_64-linux-musl only" >&2; exit 1; }

exec "$@"
EOF

USER 1000:1000
EXPOSE 50001 3000 8332
ENTRYPOINT ["rbitcoin-entrypoint"]

##############################################
# Target: ui — minimal connection-info page (nginx:alpine)
##############################################
FROM nginx:alpine AS ui

COPY --chmod=0644 <<'EOF' /etc/nginx/conf.d/rbitcoin.conf.template
server {
    listen ${RBITCOIN_UI_PORT};
    server_name _;
    root /usr/share/nginx/html;
    index index.html;
}
EOF

COPY --chmod=0644 <<'EOF' /usr/share/nginx/html/index.html.template
<!doctype html>
<html lang="en">
<head>
<meta charset="utf-8">
<meta name="viewport" content="width=device-width, initial-scale=1">
<title>rbitcoin</title>
<style>
body{font:16px/1.5 system-ui,sans-serif;max-width:44em;margin:2rem auto;padding:0 1rem}
code{background:#f2f2f2;padding:.1em .35em;border-radius:3px;word-break:break-all}
h1{font-size:1.5rem}h2{font-size:1.05rem;margin-top:1.6rem}
.warn{background:#fff4e0;border-left:4px solid #e8930c;padding:.75rem 1rem}
</style>
</head>
<body>
<h1>rbitcoin <small>v0.8.0</small></h1>
<p>Standalone full Bitcoin node in Rust &mdash; pure-Rust consensus, multi-peer
IBD, with built-in <strong>Electrum</strong>, <strong>Esplora</strong> and
<strong>Core-compatible RPC</strong> servers. It does <em>not</em> run on top of
the Bitcoin (bitcoind) app.</p>

<h2>Electrum server &mdash; TCP ${RBITCOIN_ELECTRUM_PORT} (protocol &le; 1.6)</h2>
<p>Electrum, Sparrow, Liana, Cake Wallet, BlueWallet, BitBox App, Envoy:<br>
<code>${RBITCOIN_HOST}:${RBITCOIN_ELECTRUM_PORT}</code><br>
Tor: <code>${RBITCOIN_ELECTRUM_ONION}:${RBITCOIN_ELECTRUM_PORT}</code></p>

<h2>Esplora API &mdash; HTTP ${RBITCOIN_ESPLORA_PORT}</h2>
<p>BDK, Alby Hub (mempool.space/electrs drop-in):<br>
<code>http://${RBITCOIN_HOST}:${RBITCOIN_ESPLORA_PORT}</code><br>
Tor: <code>http://${RBITCOIN_ESPLORA_ONION}:${RBITCOIN_ESPLORA_PORT}</code></p>

<h2>RPC &mdash; ${RBITCOIN_RPC_PORT}, localhost only (no Tor)</h2>
<p>Wasabi, LND, CLN &mdash; Core 0.19-compatible:<br>
<code>http://127.0.0.1:${RBITCOIN_RPC_PORT}</code>, cookie auth. The cookie file
is <code>data/rbitcoin/.cookie</code> on the Umbrel host, regenerated at every
start.</p>

<p class="warn"><strong>Heads-up:</strong> initial block download takes days and
the full-archive datadir grows beyond 700&nbsp;GB (plus silent-payments and BIP158
block-filter indexes). Shutdown can take a while (large store flush).</p>
</body>
</html>
EOF

COPY --chmod=0755 <<'EOF' /docker-entrypoint.d/10-rbitcoin-ui.sh
#!/bin/sh
# Runs via the nginx image's docker-entrypoint.d mechanism before nginx starts.
set -eu
: "${RBITCOIN_UI_PORT:=3012}"
: "${RBITCOIN_HOST:=localhost}"
: "${RBITCOIN_ELECTRUM_PORT:=50001}"
: "${RBITCOIN_ESPLORA_PORT:=3000}"
: "${RBITCOIN_RPC_PORT:=8332}"
: "${RBITCOIN_ELECTRUM_ONION:=not available yet}"
: "${RBITCOIN_ESPLORA_ONION:=not available yet}"
envsubst '${RBITCOIN_UI_PORT}' \
  < /etc/nginx/conf.d/rbitcoin.conf.template > /etc/nginx/conf.d/default.conf
envsubst '${RBITCOIN_HOST} ${RBITCOIN_ELECTRUM_PORT} ${RBITCOIN_ESPLORA_PORT} ${RBITCOIN_RPC_PORT} ${RBITCOIN_ELECTRUM_ONION} ${RBITCOIN_ESPLORA_ONION}' \
  < /usr/share/nginx/html/index.html.template > /usr/share/nginx/html/index.html
EOF

EXPOSE 3012
