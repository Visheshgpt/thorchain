# Kava AKS Deployment Guide

> **Chain:** `kava_2222-10` · **Daemon:** `kava` v0.28.0 · **Backend:** RocksDB
> **Cluster:** confirm with `kubectl config current-context` before applying

---

## 1. Node Overview

| Field | Value |
|---|---|
| **Blockchain** | Kava (Cosmos SDK v0.47.15 + evmos/ethermint v0.21.0) |
| **Chain ID** | `kava_2222-10` |
| **Image** | `kava/kava:v0.28.0-rocksdb` |
| **DB backend** | `rocksdb` — **matches the production VM** |
| **Bootstrap** | state sync from `kava-rpc.polkachu.com` |
| **Pruning** | `custom`, keep-recent 100, interval 10 |
| **Namespace** | `kava` · **Pod** `node-kava-0` · **Container** `kava-node` |
| **Data mount** | `/kava/.kava` |
| **Block time** | ~5.71 s (~15,100 blocks/day) |

### Port Map

| Port | Container | Exposure |
|---|---|---|
| P2P | `26656` | Internal LB (`node-kava-lb`) |
| CometBFT RPC | `26657` | Internal LB (`node-kava-rpc-lb`) |
| Cosmos REST | `1317` | Internal LB (`node-kava-rpc-lb`) |
| gRPC | `9090` | Internal LB (`node-kava-rpc-lb`) |

> **EVM JSON-RPC (8545/8546) is disabled**, matching the source compose file. Kava does have
> an EVM co-chain. To enable: set `[json-rpc] enable = true` + `address = "0.0.0.0:8545"` in
> the `configure` init container, then add the ports to the container and Services.

---

## 2. Why this differs from the production compose file

**Image.** `rocksdb:9.3.3` is a *local build tag* built on the VM from the Kava repo's
`Dockerfile-rocksdb`. It exists on no registry, so a pod can never pull it.
`kava/kava:v0.28.0-rocksdb` is the public equivalent.

**Why v0.28.0 and not v0.28.2** — `kava/kava` publishes no rocksdb tag past v0.28.0
(goleveldb only). Two patch releases behind is safe here, from the release notes:

- v0.28.1 — *"non-breaking and optional… not necessary if you do not use a ledger"*
- v0.28.2 — *"Bump cometbft to v0.37.18"*

Neither touches consensus, so there is no app-hash halt risk.

**RocksDB, deliberately.** Prod runs `db_backend = "rocksdb"`, and a goleveldb binary
**cannot open a RocksDB store**. Testing on goleveldb would validate the manifests while
rehearsing a configuration that could never accept the prod data.

**Pruned, not archive.** Prod runs `--pruning nothing`. Kava's docs put archive at ~6 TB disk
and 128 GB RAM. This is a *test* node whose job is to prove the stack runs.

**State sync, not a snapshot restore.** The Polkachu tarball restores a 44 GB store
(32.4 GB `application.db` + 8.4 GB `state.db`) which could not be opened in the memory
available. State sync fetches only current state — no big download, no huge store, and
CometBFT starts the RPC listener *before* syncing, so startup is not a silent 40-minute wait.

---

## 3. Apply Order

```bash
kubectl apply -f 00-namespace.yaml
kubectl apply -f 01-storageclass.yaml
kubectl apply -f 02-pvc.yaml
kubectl apply -f 03-statefulset.yaml
kubectl apply -f 04-service.yaml
```

> **Updating a running node:** a StatefulSet will **not** roll a pod that is not Ready
> (default `podManagementPolicy: OrderedReady`). During the startup window `kubectl apply`
> silently does nothing. Always follow it with:
> ```bash
> kubectl delete pod node-kava-0 -n kava
> ```

---

## 4. Kubernetes-level checks

```powershell
kubectl get pods -n kava
kubectl get pvc -n kava
kubectl get svc -n kava -o wide
kubectl rollout status statefulset/node-kava -n kava --timeout=1800s

# Logs — container name is REQUIRED (3 containers in this pod)
kubectl logs node-kava-0 -n kava -c prepare-statesync
kubectl logs node-kava-0 -n kava -c configure
kubectl logs node-kava-0 -n kava -c kava-node --tail=100
kubectl logs node-kava-0 -n kava -c kava-node -f

kubectl describe pod node-kava-0 -n kava
kubectl top pod node-kava-0 -n kava --containers
kubectl get pod node-kava-0 -n kava -o jsonpath="{.status.containerStatuses[0].restartCount}{'\n'}"
```

**Expected startup:** `prepare-statesync` (seconds) → `configure` prints
`state sync trust point: height=... hash=...` and `db_backend = "rocksdb"` → `kava-node`
discovers snapshots and applies 232 chunks over ~10 min.

> `Select-String` buffers a `-f` stream in PowerShell — a live follow piped into it prints
> nothing. Use `--tail N | Select-String ...` instead, or follow without the pipe.

---

## 5. RPC Verification — proving the node actually works

`Running 1/1` proves nothing. A node can be Running, Ready, and serving nothing. These go
from "is the process alive" to "is it serving real chain data". Stop at the first failure.

The rocksdb image is **Ubuntu-based and ships `curl` and `jq`** — so in-pod commands can use
both directly. On Windows write **`curl.exe`**; bare `curl` is a PowerShell alias.

```powershell
$NS="kava"; $POD="node-kava-0"; $C="kava-node"
```

### Step 1 — Is the process alive?

```powershell
kubectl exec -n kava node-kava-0 -c kava-node -- curl -s http://localhost:26657/health
```

`{"jsonrpc":"2.0","id":-1,"result":{}}` = the RPC server answers.

**This does NOT mean synced.** CometBFT's `/health` returns an empty `200` unconditionally
and never inspects sync state. It is what the readiness probe uses, which is why
`Ready 1/1` is not evidence of sync.

### Step 2 — Right chain, right version?

```powershell
kubectl exec -n kava node-kava-0 -c kava-node -- curl -s http://localhost:26657/abci_info
```

Expect `"data":"kava"` and `"version":"0.28.0"`. Compare against the live network:

```powershell
curl.exe -s https://rpc.data.kava.io/abci_info
```

If the network has moved to a consensus-breaking release and this node has not, it will sync
until the upgrade height and then **halt on a mismatched app hash** — healthy-looking and
permanently stuck.

### Step 3 — Connected?

```powershell
kubectl exec -n kava node-kava-0 -c kava-node -- sh -c "curl -s http://localhost:26657/net_info | jq -r .result.n_peers"
```

**Zero peers means it can never sync.** One or two is too thin and will stall — see §8.

### Step 4 — Is it synced? *(the one that matters)*

```powershell
kubectl exec -n kava node-kava-0 -c kava-node -- sh -c "curl -s http://localhost:26657/status | jq -r .result.sync_info"
```

```json
"latest_block_height": "22292945",
"earliest_block_height": "22292001",     <- state-sync point, not genesis
"catching_up": false                      <- SYNCED
```

`catching_up: false` with a height near the network tip is the definition of done.
`latest_block_height: "0"` means state sync has not completed yet.

**Height delta over 60s** — run, wait, up-arrow, compare:

```powershell
kubectl exec -n kava node-kava-0 -c kava-node -- curl -s http://localhost:26657/abci_info
```

| Delta over ~60s | Meaning |
|---|---|
| > 60 | Catching up (chain makes ~10/min) |
| ~10 | At the tip |
| 0 | Stalled |

### Step 5 — How far behind?

```powershell
curl.exe -s https://rpc.data.kava.io/status | ConvertFrom-Json | % { $_.result.sync_info.latest_block_height }
```

Compare with step 4. Gap ÷ 10 ≈ minutes of chain time behind.

### Step 6 — Is the APP layer serving, not just consensus?

The step people skip, and the one that catches a half-configured node. Consensus can be
healthy while the REST/gRPC listeners were never enabled.

```powershell
kubectl exec -n kava node-kava-0 -c kava-node -- curl -s http://localhost:1317/cosmos/base/tendermint/v1beta1/syncing
```
→ `{"syncing":false}` — REST on 1317 is up **and** the node reports itself synced.

```powershell
kubectl exec -n kava node-kava-0 -c kava-node -- sh -c "curl -s 'http://localhost:1317/cosmos/bank/v1beta1/supply/by_denom?denom=ukava' | jq"
```
→ `{"amount":{"denom":"ukava","amount":"1082846942247877"}}` — the ABCI application is
genuinely queryable. State is readable, not just blocks arriving.

### Step 7 — Balance query (real account data)

```powershell
kubectl exec -n kava node-kava-0 -c kava-node -- sh -c "curl -s http://localhost:1317/cosmos/bank/v1beta1/balances/kava1wyl4l4zmyas5ymxzyevyfj2j054rn56p637ccn | jq"
```
→ `{"balances":[{"denom":"ukava","amount":"32"}], ...}`

Any `kava1…` address works. **Historical balance** at a specific height uses a header:

```powershell
kubectl exec -n kava node-kava-0 -c kava-node -- curl -s -H "x-cosmos-block-height: 22292500" http://localhost:1317/cosmos/bank/v1beta1/balances/kava1wyl4l4zmyas5ymxzyevyfj2j054rn56p637ccn
```

> **Pruned node caveat:** this only works for heights this node still holds. Check
> `earliest_block_height` from step 4 — below that, and outside the 100-state pruning
> window, the query correctly returns an error. That is not a fault.

### Step 8 — Block queries

```powershell
# latest block
kubectl exec -n kava node-kava-0 -c kava-node -- sh -c "curl -s http://localhost:1317/cosmos/base/tendermint/v1beta1/blocks/latest | jq -r .block.header.height"

# a specific block (must be >= earliest_block_height)
kubectl exec -n kava node-kava-0 -c kava-node -- sh -c "curl -s http://localhost:26657/block?height=22292500 | jq -r .result.block.header.time"

# block results (tx outcomes / events for a height)
kubectl exec -n kava node-kava-0 -c kava-node -- sh -c "curl -s 'http://localhost:26657/block_results?height=22292500' | jq -r .result.height"

# a transaction by hash
kubectl exec -n kava node-kava-0 -c kava-node -- sh -c "curl -s 'http://localhost:26657/tx?hash=0x<TX_HASH>' | jq"
```

### Step 9 — Staking / validators / governance

```powershell
kubectl exec -n kava node-kava-0 -c kava-node -- sh -c "curl -s http://localhost:1317/cosmos/staking/v1beta1/pool | jq"
kubectl exec -n kava node-kava-0 -c kava-node -- sh -c "curl -s 'http://localhost:1317/cosmos/staking/v1beta1/validators?status=BOND_STATUS_BONDED&pagination.limit=5' | jq -r '.validators[].description.moniker'"
kubectl exec -n kava node-kava-0 -c kava-node -- sh -c "curl -s http://localhost:1317/cosmos/distribution/v1beta1/community_pool | jq"
kubectl exec -n kava node-kava-0 -c kava-node -- sh -c "curl -s http://localhost:26657/validators | jq -r .result.total"
```

### Step 10 — Kava-specific modules (verified present)

```powershell
kubectl exec -n kava node-kava-0 -c kava-node -- sh -c "curl -s http://localhost:1317/kava/cdp/v1beta1/params | jq"
kubectl exec -n kava node-kava-0 -c kava-node -- sh -c "curl -s http://localhost:1317/kava/earn/v1beta1/vaults | jq"
kubectl exec -n kava node-kava-0 -c kava-node -- sh -c "curl -s http://localhost:1317/kava/incentive/v1beta1/params | jq"
kubectl exec -n kava node-kava-0 -c kava-node -- sh -c "curl -s http://localhost:1317/kava/evmutil/v1beta1/params | jq"
```

### Step 11 — Mempool and listeners

```powershell
kubectl exec -n kava node-kava-0 -c kava-node -- curl -s http://localhost:26657/num_unconfirmed_txs
kubectl exec -n kava node-kava-0 -c kava-node -- sh -c "ss -ltn 2>/dev/null || netstat -ltn"
```

All of 26656 / 26657 / 1317 / 9090 must listen on `0.0.0.0`. Anything on `127.0.0.1` is
unreachable from a Service — a config fault, not a network one.

### Step 12 — From outside the cluster

```powershell
kubectl get svc node-kava-rpc-lb -n kava -o jsonpath='{.status.loadBalancer.ingress[0].ip}'
curl.exe -s http://<ILB-IP>:26657/status
curl.exe -s http://<ILB-IP>:1317/cosmos/base/tendermint/v1beta1/syncing
```

### Pass criteria

| # | Check | Pass |
|---|---|---|
| 1 | `/health` | `{}` |
| 2 | `/abci_info` | `kava` `0.28.0` |
| 3 | `n_peers` | `> 0`, ideally 5+ |
| 4 | `catching_up` | `false` |
| 5 | `/cosmos/.../syncing` | `{"syncing":false}` |
| 6 | `supply/by_denom?denom=ukava` | returns an amount |
| 7 | balance query | returns balances |
| 8 | listeners | 4 ports on `0.0.0.0` |
| 9 | logs | no `AppHash`, `CONSENSUS FAILURE`, `panic` |

```powershell
kubectl logs node-kava-0 -n kava -c kava-node --tail=4000 | Select-String "AppHash|CONSENSUS FAILURE|panic|OOMKilled"
```

Empty output is the pass.

---

## 6. Operational switches

Both live on the `prepare-statesync` init container. **No throwaway pods** — mounting the PVC
by hand routes around the guard that protects real chain data.

| Env var | Effect |
|---|---|
| `RESET_FOR_STATESYNC=<new value>` | Wipe `data/` and state-sync again. A **token, not a boolean** — recorded on the volume once honoured, so a crashloop cannot re-wipe. Change the value to force again. |
| `ADOPT_EXISTING=true` | Use chain data already on the volume as-is, no wipe. **This is the switch for the prod-data migration.** Never deletes. |

```powershell
kubectl set env statefulset/node-kava -n kava -c prepare-statesync RESET_FOR_STATESYNC=redo-1
kubectl delete pod node-kava-0 -n kava
```

> `kubectl set env` mutates the live StatefulSet, not the YAML. A later `kubectl apply`
> resets both to the file's values. The file is the source of truth.

---

## 6b. Auto-Disk Config (pvc-watcher)

The `pvc-watcher` CronJob (namespace `pvc-watcher`, schedule `*/15 * * * *`) monitors disk
usage and expands the PVC when the threshold is hit.

```
# Format: "NAMESPACE|POD|MOUNT_PATH|PVC_NAME|THRESHOLD|INCREMENT_GI|MAX_SIZE_GI"
"kava|node-kava-0|/kava/.kava|node-kava-data|80|100|500"
```

| Field | Value |
|---|---|
| Namespace | `kava` |
| Pod | `node-kava-0` |
| Mount path | `/kava/.kava` |
| PVC | `node-kava-data` |
| Threshold | 80% usage triggers expansion |
| Increment | +100Gi per trigger |
| Maximum | 500Gi — will not expand beyond this |

**Add it to the live cluster:**

```powershell
# Interactive
kubectl edit configmap pvc-watcher-script -n pvc-watcher

# Or inspect first, then patch
kubectl get configmap pvc-watcher-script -n pvc-watcher -o yaml
kubectl patch configmap pvc-watcher-script -n pvc-watcher --type merge -p '{\"data\":{\"NODE_CONFIG_KAVA\":\"kava|node-kava-0|/kava/.kava|node-kava-data|80|100|500\"}}'
```

> A state-synced node starts small (a few GB, not the 44 GB a snapshot restore produces), so
> expansion is unlikely to trigger soon on this test node. Remove the NODE_CONFIG line at
> teardown.

---

## 7. Stop / start / teardown

```powershell
kubectl scale statefulset node-kava -n kava --replicas=0     # stop, keep data
kubectl scale statefulset node-kava -n kava --replicas=1     # resume, no re-sync

kubectl delete -f 04-service.yaml -f 03-statefulset.yaml -f 02-pvc.yaml
```

`reclaimPolicy: Retain` means the Azure Disk **survives** and the PV goes `Released`:

```powershell
kubectl get pv -o custom-columns=NAME:.metadata.name,STATUS:.status.phase,DISK:.spec.csi.volumeHandle
kubectl delete pv <released-pv>
az disk delete --ids "<volumeHandle>" --yes
```

---

## 8. Node-Specific Notes — lessons that cost real time

### Memory: 10Gi. Do NOT raise it.

A **24Gi limit made state sync fail repeatably** — 230 of 232 chunks applied, the final chunk
never delivered, three runs across two DB backends and three different snapshots. Dropping to
**10Gi fixed it on the first attempt**.

Why lowering helps: at 24Gi the pod grows its page cache until the *node* sits at ~27 of
29 GiB (93%). Node-level pressure triggers kernel reclaim that is not confined to this cgroup
— enough to disrupt the p2p read path and lose the last chunk. A 10Gi cap makes the cgroup
reclaim its own cache early and never pressures the node.

The instinct on a slow start is to raise memory. Here that is exactly wrong.

### persistent_peers must be REACHABILITY-TESTED, not just harvested

`seeds` alone is not enough — a CometBFT seed answers one PEX request and disconnects by
design (`p2p/pex/pex_reactor.go`), so a seeds-only node holds 1–2 peers and stalls.

But a harvested list is not enough either. The first list came straight from a public
`/net_info` and **most of it was dead** — the node reached `numOutPeers=1`. Only ~56% of
harvested peers were dialable from this cluster. Test before trusting:

```powershell
kubectl exec -n kava node-kava-0 -c kava-node -- bash -c 'for i in 91.134.37.131 94.155.47.54 77.74.195.236; do timeout 3 bash -c "echo >/dev/tcp/$i/26656" 2>/dev/null && echo "$i OK"; done'
```

The rocksdb image is Ubuntu — **no `nc`**. Use bash's `/dev/tcp` as above.

### Init containers must not depend on file ownership

`prepare-statesync` runs as **UID 1000, not root**. Root with `capabilities: drop: ["ALL"]`
has no `CAP_DAC_OVERRIDE` or `CAP_FOWNER`, making it *weaker* than a normal user for file
operations — it cannot delete or chmod files owned by 1000, which is what state sync writes.
That caused two `Init:CrashLoopBackOff` rounds.

The wipe now tries `rm -rf` and falls back to `mv` if blocked: **renaming `data/` only needs
write on `/kava/.kava`**, which is group-writable via `fsGroup: 1000` and therefore always
works. Old `data.discard.*` directories are reclaimed best-effort on later boots.

### Probes

All three hit CometBFT `/health` on 26657, **not** a TCP check on 26656 — a TCP probe on the
P2P port reports healthy through a total freeze because the listener never closes. Liveness
additionally tests **height progress** and restarts only after a 30-minute stall.

`startupProbe` allows 60 min. With state sync the RPC listens *before* syncing, so this
clears in minutes; the old snapshot path needed hours and a too-small threshold caused an
endless restart loop.

### Migration to production

Prod's chain data is **RocksDB** — this node matches, so a disk snapshot can be adopted with
`ADOPT_EXISTING=true`. Prod also runs `--pruning nothing` (archive): Kava's docs put that at
~6 TB disk and 128 GB RAM, so the production cluster needs sizing for that, not for this.
