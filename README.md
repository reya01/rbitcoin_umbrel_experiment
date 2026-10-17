# rbitcoin-umbrel

An [Umbrel](https://umbrel.com/) app package for [rbitcoin](https://github.com/reardencode/rbitcoin) — a full Bitcoin node in Rust by **reardencode**. All credit for rbitcoin itself goes entirely to **reardencode** and the rbitcoin contributors; this repository only packages their work for Umbrel.

I prefer rbitcoin's new approach: it is written in a memory-safe programming language, it does away with the traditional UTXO set in favor of a map-free relational archive, and it ships a transaction indexer (Electrum + Esplora) built right into the node — one binary, no separate indexer to run, and no bitcoind dependency.

> ⚠️ **This package was vibe coded and is meant solely for fun experimentation. Do not use it with real funds.** rbitcoin is pre-1.0 experimental software, and this packaging is a hobby project by an amateur — neither is audited or production-ready.

## What it runs

- **rbitcoin v0.8.0** full node (mainnet), binaries verified against the official `SHA256SUMS.linux-musl` at build time
- Built-in **Electrum server** (TCP 50001) — Electrum, Sparrow, Liana, Cake Wallet, BlueWallet, BitBox App, Envoy
- Built-in **Esplora API** (HTTP 3000) — BDK wallets, Alby Hub
- **Core-compatible RPC** (8332, cookie auth, localhost-only) — Wasabi, LND, CLN
- **Tor hidden services** for the Electrum and Esplora listeners (RPC deliberately stays off Tor)

The node is standalone: it does **not** depend on (or use) the Bitcoin (bitcoind) Umbrel app.

## Install on your personal Umbrel

These are the explicit, from-scratch steps. You need SSH access to your Umbrel and about **700+ GB of free disk** (the full archive plus indexes). x86_64 only — v0.8.0 ships no ARM binaries.

**1. Build the two images on the Umbrel host** (from any directory with Docker):

```bash
git clone https://github.com/xjgsfqfznm-maker/rbitcoin-umbrel.git
cd rbitcoin-umbrel/rbitcoin
docker build --target node -t rbitcoin-node:0.8.0 .
docker build --target ui   -t rbitcoin-ui:0.8.0 .
```

The build downloads the v0.8.0 release binaries from GitHub and fails hard if their SHA-256 doesn't match the official checksums.

**2. Copy the package into your Umbrel apps directory:**

```bash
# from the repo root, on the Umbrel host:
sudo cp -r rbitcoin ~/umbrel/apps/
```

**3. Restart Umbrel so it picks up the local app:**

```bash
sudo ~/umbrel/scripts/stop
sudo ~/umbrel/scripts/start
```

**4. Install it:** open your Umbrel home screen — **rbitcoin** appears in the App Store's local/community section (or under "Apps" with an "install from local" path depending on your Umbrel version). Click **Install**.

**5. Wait for initial block download.** This takes **days** on ordinary connections (it is multi-peer and does not need bitcoind). Track progress in the app logs:

```bash
docker logs umbrel-rbitcoin-rbitcoin-1 --tail 20
```

**6. Connect a wallet.** Open the rbitcoin app page in Umbrel — it shows your actual host ports and Tor onion addresses (Umbrel assigns the ports dynamically; don't assume 50001/3000):

- Electrum / Sparrow / Liana / Cake / BlueWallet / BitBox / Envoy → the **Electrum** address shown (plain TCP)
- BDK wallets / Alby Hub → the **Esplora** `http://` URL shown
- Wasabi / LND / CLN → SSH into the Umbrel, `127.0.0.1:<rpc-port>`; the RPC cookie file is at `~/umbrel/app-data/rbitcoin/data/rbitcoin/.cookie` (one `user:pass` line, no trailing newline)

**Uninstall:** remove the app from Umbrel, then `sudo rm -rf ~/umbrel/app-data/rbitcoin` to reclaim the disk.

## Notes & caveats

- The node runs with `--sh-index --sp-tweaks --block-filter-index` (scripthash, silent payments, and BIP158 filter indexes build during IBD).
- `stop_grace_period` is 30 minutes — the node flushes a large store on shutdown; let it stop cleanly.
- RPC is deliberately **not** exposed over Tor; it is loopback-only with cookie auth.
- If the node crash-loops on first boot with a permissions error, run `sudo chown -R 1000:1000 ~/umbrel/app-data/rbitcoin/data`.

## Links

- rbitcoin website: **[rBitcoin.org](https://rbitcoin.org/)**
- rbitcoin source & docs: [github.com/reardencode/rbitcoin](https://github.com/reardencode/rbitcoin)
- rbitcoin releases: [v0.8.0](https://github.com/reardencode/rbitcoin/releases/tag/v0.8.0)

## Donate to reardencode

If you like rbitcoin, support its author directly:

- **Silent payments (on-chain):** `sp1qqwkt0aetchmm5gzucxych7l7kxaevetpx7npe54e236zqv73jevr5qucy5pgnut9fm7syjmlqv9ykm3qzzpfc0v3j737plzvvuy8k08a4scn62wl`
- **Lightning (Strike):** `reardencode@strike.me`

## License

MIT — see [LICENSE](LICENSE). rbitcoin itself is Apache-2.0 OR MIT under its own license.
