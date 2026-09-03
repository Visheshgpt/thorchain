# Cronos EVM — AKS Deployment Guide

> **Chain:** `cronosmainnet_25-1` · **Daemon:** `cronosd` v1.7.8 · **Backend:** RocksDB
> Confirm your context with `kubectl config current-context` before applying.

---

## 1. Node Overview

| Field | Value |
|---|---|
| Chain ID | `cronosmainnet_25-1` |
| Stack | Cosmos SDK v0.53.4 + evmos/ethermint |
| Image | `ubuntu:22.04` + upstream release binary (SHA256-verified) |
| DB backend | `rocksdb` (matches prod) — switchable via `DB_BACKEND` |
| Bootstrap | Cronos native `rocksdb-pruned` snapshot |
| Pruning | `custom`, keep-recent 100, interval 10 |
| Namespace | `cronos` · pod `node-cronos-0` · container `cronosd` |
| Data | `/data/.cronos` on a 256Gi StandardSSD_LRS disk |

### Ports

| Port | Container | Exposure |
|---|---|---|
| P2P | 26656 | Internal LB `node-cronos-lb` (5332) |
| CometBFT RPC | 26657 | Internal LB `node-cronos-rpc-lb` (2332) |
| Cosmos REST | 1317 | Internal LB (4332) |
| gRPC | 9090 | Internal LB (9090) |
| EVM JSON-RPC | 8545 / 8546 | **disabled** — see §7 |

---

## 2. Deploy

```powershell
kubectl apply -f 00-node_cronos_namespace.yaml
kubectl apply -f 01-node_cronos_storageclass.yaml
kubectl apply -f 02-node_cronos_pvc.yaml
kubectl apply -f 03-node_cronos_statefulset.yaml
kubectl apply -f 04-node_cronos_service.yaml
```

---

## 3. Is it running?

```powershell
kubectl get pods -n cronos -w
```

`Init:0/3` → `Init:2/3` → `Running` → `1/1`.

**During the snapshot download (~20–25 min):**

```powershell
kubectl logs -n cronos node-cronos-0 -c restore-snapshot -f
```

First line must be `PVC confirmed: /dev/… 256G`. Then `selected …rocksdb-pruned…`,
`extract verified`, `DONE`.

**Once Running:**

```powershell
kubectl logs -n cronos node-cronos-0 -c cronosd -f
```

`ABCI Handshake App Info` must show a **real height near 91.5M**. `height=0` means the
store is empty and it will fall through to `InitChain` — see §8.

---

## 4. If it is NOT running

```powershell
kubectl describe pod node-cronos-0 -n cronos | Select-String "Exit Code|Reason"
```

```powershell
kubectl logs -n cronos node-cronos-0 -c cronosd --previous | Select-Object -First 30
```

> The error prints at the **top**. `--tail` shows you the cosmos usage dump that follows it.

| Symptom | Cause | Fix |
|---|---|---|
| `exit=139` (SIGSEGV) | RocksDB cannot **create** a store — only open one | ensure the snapshot actually restored; else `DB_BACKEND=goleveldb` |
| `x/gov InitGenesis` nil deref | fell through to `InitChain` — store empty | §8 |
| `Init:Error` on `restore-snapshot` | download or extract failed | `kubectl logs … -c restore-snapshot` |
| `FATAL: … NOT on a persistent volume` | `volumeMount` missing | see §8 |
| `tls: certificate signed by unknown authority` | CA bundle missing | `install-cronosd` stages it; check `CA bundle staged` |

---

## 5. RPC verification

The main container is a bare `ubuntu:22.04` — **no `curl`, no `wget`**. Use port-forward.

```powershell
kubectl port-forward -n cronos node-cronos-0 26657:26657 1317:1317
```

Leave that running; use a second window below. On Windows write **`curl.exe`** — bare
`curl` is a PowerShell alias for `Invoke-WebRequest`.

### Alive

```powershell
curl.exe -s http://localhost:26657/health
```
→ `{"jsonrpc":"2.0","id":-1,"result":{}}`

**This does not mean synced** — CometBFT returns 200 unconditionally.

### Synced — the one that matters

```powershell
curl.exe -s http://localhost:26657/status | ConvertFrom-Json | % { $_.result.sync_info }
```

```
latest_block_height   : 91552143      <- near the network tip
earliest_block_height : 91530000      <- snapshot point, not 1
catching_up           : False         <- DONE
```

### Right chain and version

```powershell
curl.exe -s http://localhost:26657/abci_info
```
→ `"data":"cronos"`, `"version":"v1.7.8"`

Compare with the network:
```powershell
curl.exe -s https://rpc.cronos.com/abci_info
```

> The network runs **v1.7.9**, which has **no GitHub release** — only a git tag, so there is
> no binary. Its diff from v1.7.8 is `disable evidence slash` / `disable evidence p2p`.
> Fine for this node; a production deployment would need a v1.7.9 build.

### Peers

```powershell
curl.exe -s http://localhost:26657/net_info | ConvertFrom-Json | % { $_.result.n_peers }
```
`0` means it can never sync.

### App layer — catches a half-configured node

```powershell
curl.exe -s http://localhost:1317/cosmos/base/tendermint/v1beta1/syncing
```
→ `{"syncing":false}`

```powershell
curl.exe -s "http://localhost:1317/cosmos/bank/v1beta1/supply/by_denom?denom=basecro"
```
→ `{"amount":{"denom":"basecro","amount":"19391644598827388197999…"}}`

Proves the ABCI application is queryable, not merely that blocks arrive.

### Blocks and balances

```powershell
curl.exe -s http://localhost:1317/cosmos/base/tendermint/v1beta1/blocks/latest | ConvertFrom-Json | % { $_.block.header.height }
curl.exe -s http://localhost:26657/block?height=91550000 | ConvertFrom-Json | % { $_.result.block.header.time }
curl.exe -s http://localhost:1317/cosmos/bank/v1beta1/balances/<crc-or-cro-address>
curl.exe -s http://localhost:1317/cosmos/staking/v1beta1/pool
curl.exe -s http://localhost:26657/num_unconfirmed_txs
```

> **Pruned + snapshot-restored:** anything below `earliest_block_height` returns an error.
> That is correct, not a fault.

### Pass criteria

| # | Check | Pass |
|---|---|---|
| 1 | `/health` | `{}` |
| 2 | `/abci_info` | `cronos` `v1.7.8` |
| 3 | `n_peers` | `> 0` |
| 4 | `latest_block_height` | near the network tip, **not 0** |
| 5 | `catching_up` | `false` |
| 6 | `syncing` | `{"syncing":false}` |
| 7 | `supply/by_denom?denom=basecro` | returns an amount |
| 8 | logs | no `panic`, `AppHash`, `CONSENSUS FAILURE` |

```powershell
kubectl logs -n cronos node-cronos-0 -c cronosd --tail=4000 | Select-String "panic|AppHash|CONSENSUS FAILURE|OOMKilled"
```

Empty output is the pass.

---

## 6. Sync Proof

Evidence that this node reached the network tip. Take the screenshot from:

```powershell
curl.exe -s http://localhost:26657/status | ConvertFrom-Json | % { $_.result.sync_info }
```

showing `latest_block_height` and `catching_up: False`, alongside the live network height
from `https://cronoscan.com/` or:

```powershell
curl.exe -s https://rpc.cronos.com/abci_info
```

Save the image as `screenshots/cronos-latest-height.png`:

![Cronos latest block height](./screenshots/cronos-latest-height.png)

| Field | Value |
|---|---|
| Date captured | `YYYY-MM-DD` |
| Node height | |
| Network height | |
| `catching_up` | |
| Gap (blocks) | |

---

## 7. Operational switches

All on the `restore-snapshot` init container. **Never mount the PVC from a throwaway pod** —
that routes around the guard protecting real chain data.

| Env | Effect |
|---|---|
| `DB_BACKEND` | `rocksdb` or `goleveldb`. Drives **both** the snapshot source and `db_backend`/`app-db-backend`, so they cannot diverge. |
| `RESET_DATA=<new value>` | Wipe `data/` and re-restore. A **token, not a boolean** — recorded on the volume once honoured, so a crashloop cannot re-wipe. |
| `ADOPT_EXISTING=true` | Use data already on the volume as-is. **The switch for a prod-data migration.** Never deletes. |

Changing `DB_BACKEND` requires a new `RESET_DATA` — the on-disk format changes.

```powershell
kubectl set env statefulset/node-cronos -n cronos -c restore-snapshot DB_BACKEND=goleveldb RESET_DATA=switch-1
kubectl set env statefulset/node-cronos -n cronos -c configure DB_BACKEND=goleveldb
kubectl delete pod node-cronos-0 -n cronos
```

> `kubectl set env` mutates the live StatefulSet, not the YAML. A later `kubectl apply`
> resets it to the file's values. The file is the source of truth.

**EVM JSON-RPC is disabled** (`[json-rpc] enable = false`), matching the source compose which
exposed only 26657/26656/1317. To enable: set it true, `address = "0.0.0.0:8545"`, and add
the ports to the container and Services.

---

## 8. Node-Specific Notes — lessons that cost real time

### Cannot start from genesis — ever

The mainnet genesis (sha256 `58f17545…`, verified) is 19,442 bytes and carries the **2021-era
gov layout**: `deposit_params` / `voting_params` / `tally_params`. cosmos-sdk v0.47
consolidated those into a single `params` object, and v1.7.8 runs **v0.53.4**, so
`x/gov.InitGenesis` executes:

```go
err = k.Params.Set(ctx, *data.Params)     // genesis.go:21
```

`data.Params` is `nil` → nil pointer dereference → SIGSEGV. **A snapshot is mandatory.**

### RocksDB opens but cannot create

Measured, same host, same clean directory:

```
--db_backend=rocksdb    -> Segmentation fault (core dumped), exit 139
--db_backend=goleveldb  -> reaches "starting node with ABCI CometBFT in-process"
```

But with a **restored** RocksDB store it starts fine. The release binary is built with
`grocksdb_no_link`, so its RocksDB linkage differs from a self-built image. Practical rule:
**RocksDB works only when a snapshot is restored first.** If the store is ever empty, it
segfaults.

### `df` reporting `overlay` means the PVC is not mounted

`restore-snapshot` lost its `volumeMount` twice. Without it the entire restore runs against
the container's ephemeral filesystem: 22 GB downloads, `extract verified` prints, `DONE`
prints — and the PVC stays empty, so the node opens an empty store and falls through to
`InitChain`. Every log line looks like success.

The container now **fails fast** if `/data/.cronos` is not on a persistent volume, and prints
`PVC confirmed: /dev/… 256G` when it is.

### State sync does not work for this chain

Tried and abandoned. The only reachable light-client RPC (`rpc.cronos.com`) retains ~278k
blocks, and peers offer snapshots *below* that window, so the light client cannot fetch
`snapshot_height + 1` and every snapshot is rejected with `light block not found`. Not peer
luck — structural. Cronos' own docs recommend native snapshots when state sync will not take.

### The main container has an empty trust store

`ubuntu:22.04` ships no CA certificates, and the init containers' `apt-get` lands in *their*
filesystems. Without staging, the first HTTPS call fails with
`x509: certificate signed by unknown authority`. `install-cronosd` copies the bundle to
`/data/certs/` and the main container sets `SSL_CERT_FILE`.

### `minimum-gas-prices`

`46000000000000basecro`, per Cronos' own docs. `0.025basecro` (the CronosPOS value) is wrong
here — `basecro` is 18-decimal on Cronos EVM, so `0.025` is effectively zero.

### Migration to production

Prod runs `--pruning nothing` (archive) on RocksDB. This node is pruned on RocksDB, so the
backend matches — set `ADOPT_EXISTING=true` to have it adopt a restored prod disk. Size the
production cluster for archive (multi-TB), not for this.
