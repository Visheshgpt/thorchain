# THORChain full node on AKS — diagnosis and corrected manifests

All commands are PowerShell (Azure jumpbox). No Helm.

---

## 1. Why your node is not syncing

Your node reported:

```
latest_block_height : 27301368
latest_block_time   : 2026-08-05T09:57:15Z
catching_up         : true
```

The live network at the time of this analysis was at **27,372,302** (2026-08-10T14:45Z).
Your node is **~71,000 blocks / 5 days behind and not moving.**

### The primary cause: wrong binary version

```
Network consensus version   : 3.19.3   (since block 26,897,608)
Your image                  : mainnet-3.18.1
```

Verify it yourself:

```powershell
curl.exe -s "https://gateway.liquify.com/chain/thorchain_api/thorchain/version"
# {"current":"3.19.3","next":"3.19.3","next_since_height":26897608,"querier":"3.19.0"}
```

The network switched to 3.19.3 at block 26,897,608 — **400,000 blocks before the
snapshot you restored**. A 3.18.1 binary does not contain 3.19.x consensus logic at
all. It restores the snapshot fine and can even commit a block or two, but as soon as
a 3.19-gated code path executes it produces a **different app hash** than the rest of
the network. CometBFT then rejects every following block, and the node parks on one
height forever with `catching_up: true` — which is exactly what you are seeing.

`registry.gitlab.com/thorchain/thornode:mainnet-3.19.3` exists in the registry and is
the tag that matches live consensus.

### Second cause: you restored a snapshot and went nowhere

Liquify's snapshot list shows the tarball you used:

| type | block_height | created |
|---|---|---|
| mainnet-pruned | **27301367** | 2026-08-05T16:24Z |

Your node sits at **27301368 = snapshot height + 1**. It has made essentially **zero**
forward progress since restore. This is not "slow sync" — it never started. Two
things independently prevent progress, and both are in your manifests (below).

### Not the cause

- **Not a port problem inside the cluster.** The node is listening on `0.0.0.0:27146`
  and `0.0.0.0:27147` correctly; `thornode status` answers.
- **Not a genesis problem.** Your block hashes match the canonical chain exactly
  (block 27301368 hash `4C10D980…`, app hash `7577F5B8…`). You are on the right chain.
- **Not disk.** 300 Gi is too tight long-term, but it is not what froze you.

---

## 2. What is wrong in `manifests/` (v1), item by item

| # | v1 does | Why it breaks | v2 |
|---|---|---|---|
| 1 | `image: mainnet-3.18.1` | Behind live consensus 3.19.3 → app-hash divergence → permanent halt | `mainnet-3.19.3` |
| 2 | Overrides the entrypoint with `thornode start` to skip `render-config` | `render-config` is the **only** thing that writes `seeds` into config.toml. Skipping it leaves the node with no seed discovery. | Run the image's own `CMD ["/scripts/fullnode.sh"]` |
| 3 | Comment claims `render-config` overwrites `genesis.json` | **False.** `config.go: thornodeFetchGenesis()` returns early if `genesis.json` exists. The premise the whole v1 design rests on is wrong. | Uses `render-config` as intended |
| 4 | `init-config` sets `seeds = ""` and deletes `addrbook.json` **on every pod start** | Blanks seed discovery and throws away every peer the node ever learned, every restart | Deleted — `render-config` handles it |
| 5 | `init-config` builds `persistent_peers` from Liquify `net_info` `.url` | In CometBFT `net_info`, `url` for an **inbound** peer is `mconn://id@ip:<ephemeral-source-port>`, not `:27146`. Most of those 15 "peers" are undialable garbage. | Deleted; optional verified static peer list in the StatefulSet |
| 6 | Init order is `init-config` **before** `init-snapshot` | The tarball ships its own `config/config.toml`, which overwrites every patch on first boot — so your effective config depends on how many times the pod restarted | Snapshot restores first; producer's `config.toml`/`app.toml` are deleted so `render-config` writes fresh ones |
| 7 | `limits: cpu 1500m, memory 3Gi` | Upstream's own chart requests **4 CPU / 4Gi with no limits**. A CPU limit CFS-throttles block execution; 3Gi OOMKills a mainnet thornode. | requests 4 CPU / 8Gi, memory limit only, **no CPU limit** |
| 8 | Probes are `tcpSocket` on the p2p port | The p2p listener stays open on a totally frozen node — the probes reported healthy for 5 days | HTTP `/health` on the RPC port; no liveness probe |
| 9 | Runbook says set `EXTERNAL_IP` to the internal LB IP | Advertises an RFC1918 address to the public network. Peers that fail to dial back deprioritise and drop you. | `EXTERNAL_IP` intentionally unset |
| 10 | `cachingMode: ReadWrite` on the StorageClass | Host caching in front of a write-heavy LevelDB workload adds latency spikes | `cachingMode: None` |
| 11 | PVC 300 Gi | Restore peak alone is ~240 GB (40 GB tarball + ~200 GB extracted) | 600 Gi |
| 12 | Snapshot script mints one signed URL | Liquify signed URLs expire in **600 s**; a 40 GB download does not finish in 10 minutes | Re-mints the URL per retry, aria2c resumes the partial file |

---

## 3. Facts this was verified against

Read from THORChain source and live APIs, not from memory:

- `build/docker/Dockerfile` → `CMD ["/scripts/fullnode.sh"]`, `ENV NET=mainnet`, debian-bookworm-slim
- `build/scripts/fullnode.sh` → `init_chain` (only if no genesis) → `thornode render-config` → `exec thornode start`
- `build/scripts/core.sh` → mainnet `PORT_P2P=27146`, `PORT_RPC=27147`, `ulimit -n 65535`
- `config/default.yaml` → `thor.seed_nodes_endpoint = https://gateway.liquify.com/chain/thorchain_api/thorchain/nodes`, `thor.cosmos.pruning = nothing`, `prometheus_listen_addr = ":26660"`, `indexer = "kv"`
- `config/config.go` → `viper.SetEnvKeyReplacer(".","_") + AutomaticEnv()`, so **any** key in `default.yaml` is settable as `THOR_<UPPER_SNAKE>`
- `config/config.go: thornodeFetchGenesis()` → returns early when `genesis.json` exists
- `config/config.go: InitThornode()` → overwrites `.P2P.Seeds` unconditionally, but **not** `.P2P.PersistentPeers`
- `node-launcher/thornode/values.yaml` → `resources.requests: cpu 4, memory 4Gi`, **no limits**; mainnet PVC `1.5Ti`
- Registry tags: `mainnet-3.19.3` is the newest `mainnet-3.*`

---

## 4. Egress your AKS pods actually need

This is the one thing I cannot check from outside your VNet, and it decides whether
you need the fallback peer list.

| Destination | Port | Needed for |
|---|---|---|
| `gateway.liquify.com` | 443/TCP | seed list (`/thorchain/nodes`) and, if enabled, state-sync |
| `snapshot-api.liquify.com`, `snapshots-*.liquify.com` | 443/TCP | snapshot download |
| any active validator IP | **27147/TCP** | `render-config` probes `http://<ip>:27147/status` to learn each seed's node ID |
| any active validator IP | **27146/TCP** | the actual block gossip |

`render-config` resolves seeds like this: fetch active validator IPs over HTTPS, then
probe each on **27147** for its node ID, then write `nodeid@ip:27146` into `seeds`.
**If outbound 27147 is blocked, the seed list comes back empty and `seeds = ""`.**

I resolved the current node IDs for you from outside (94 of 95 active validators
responded), so if 27147 is blocked you can skip that probe entirely by uncommenting
`THOR_TENDERMINT_P2P_PERSISTENT_PEERS` in `04-statefulset.yaml`. `persistent_peers`
already carries node IDs, so it only needs 27146.

Note: `THOR_TENDERMINT_P2P_SEEDS` would **not** work as a fallback — `InitThornode()`
overwrites `.P2P.Seeds` after config load. Use `PERSISTENT_PEERS`.

---

## 5. Deploy

### 5.1 Check the egress question first (5 minutes, before anything else)

```powershell
kubectl run netcheck -n thorchain --rm -it --restart=Never --image=alpine:3.19 -- sh -c "
  apk add --no-cache curl >/dev/null 2>&1
  echo '--- 443 to liquify';        curl -s -o /dev/null -w '%{http_code}\n' --max-time 15 https://gateway.liquify.com/chain/thorchain_api/thorchain/version
  echo '--- 27147 to validator';    curl -s -o /dev/null -w '%{http_code}\n' --max-time 15 http://70.34.247.166:27147/status
  echo '--- 27146 to validator';    (nc -z -w 10 70.34.247.166 27146 && echo open) || echo BLOCKED
"
```

- `200`, `200`, `open` → everything works, deploy as-is.
- `200`, timeout/`000`, `open` → **uncomment `THOR_TENDERMINT_P2P_PERSISTENT_PEERS`** in `04-statefulset.yaml`.
- 27146 blocked → nothing will sync. Get the firewall changed first.

### 5.2 Tear down the poisoned state

The existing PVC holds state written by a 3.18.1 binary. Do not try to resume it —
re-restore from a fresh snapshot with the correct binary.

```powershell
kubectl delete sts thornode -n thorchain --cascade=foreground
kubectl delete pvc thornode-data -n thorchain     # PV is Retain; the Azure disk survives
```

### 5.3 Apply v2

```powershell
cd <path>\AKS_SETUP\manifests-v2
kubectl apply -f 00-namespace.yaml
kubectl apply -f 01-storageclass.yaml
kubectl apply -f 02-pvc.yaml
kubectl apply -f 03-configmap-scripts.yaml
kubectl apply -f 04-statefulset.yaml
kubectl apply -f 05-service.yaml
```

> **Node pool.** The pod now requests **4 CPU / 8 Gi**. Your v1 comments say the
> cluster is at capacity — if the pod stays `Pending`, that is a scheduling problem,
> not a manifest bug. `Standard_D8s_v3` (8 vCPU / 32 GiB) dedicated is the practical
> minimum. Do not shrink the requests back down to make it fit; that is what caused
> problem #7.

### 5.4 Watch it come up

```powershell
kubectl get pods -n thorchain -w
```

```
Init:0/3   init-passwd     ~5 s
Init:1/3   init-keys       ~10 s
Init:2/3   init-snapshot   30-60 min  (40 GB download + extract + chmod)
Running    thornode        render-config, then catch up
```

```powershell
kubectl logs -n thorchain thornode-0 -c init-snapshot -f
kubectl logs -n thorchain thornode-0 -c thornode -f
```

In the `thornode` logs you must see `render-config` report seeds, e.g.
`found N thorchain seeds` then `found N p2p seeds`. **If it says `found 0 p2p seeds`,
your 27147 egress is blocked — go back to section 4.**

---

## 6. Commands to check state

### Is it actually progressing?

```powershell
# Run twice, ~60 s apart. The number MUST increase by ~10 per minute (6 s blocks).
kubectl exec -n thorchain thornode-0 -c thornode -- `
  curl -s http://localhost:27147/status | `
  ConvertFrom-Json | Select-Object -ExpandProperty result | `
  Select-Object -ExpandProperty sync_info | `
  Select-Object latest_block_height, latest_block_time, catching_up
```

### How far behind is it?

```powershell
$local  = (kubectl exec -n thorchain thornode-0 -c thornode -- curl -s http://localhost:27147/status | ConvertFrom-Json).result.sync_info.latest_block_height
$remote = (curl.exe -s "https://gateway.liquify.com/chain/thorchain_rpc/status" | ConvertFrom-Json).result.sync_info.latest_block_height
"local=$local  network=$remote  behind=$([int]$remote - [int]$local)"
```

### Peers — the single most useful check

```powershell
kubectl exec -n thorchain thornode-0 -c thornode -- `
  curl -s http://localhost:27147/net_info | ConvertFrom-Json | `
  Select-Object -ExpandProperty result | Select-Object n_peers
```

`n_peers = 0` → peering problem (section 4). `n_peers > 5` but height frozen →
version/app-hash problem (section 1).

### Restart count — this distinguishes the two failure modes

```powershell
kubectl get pod thornode-0 -n thorchain -o wide
kubectl describe pod thornode-0 -n thorchain | Select-String -Pattern "Restart|Last State|Reason|Exit Code|OOM"
```

- High restart count + `Reason: Error` → app-hash panic (wrong binary) or OOMKill.
- `Reason: OOMKilled` → memory limit too low.
- 0 restarts + frozen height → no peers.

### Did it panic? (grep the fatal patterns)

```powershell
kubectl logs -n thorchain thornode-0 -c thornode --tail=4000 | `
  Select-String -Pattern "wrong Block.Header.AppHash|Consensus Failure|panic|UPGRADE NEEDED|no seeds|found 0 p2p seeds|dial|version"
```

`wrong Block.Header.AppHash. Expected X, got Y` is the definitive signature of the
version mismatch in section 1.

### What the node thinks its config is

```powershell
kubectl exec -n thorchain thornode-0 -c thornode -- `
  sh -c "grep -E '^(seeds|persistent_peers|external_address|laddr|pex|addr_book_strict) ' /data/.thornode/config/config.toml"

kubectl exec -n thorchain thornode-0 -c thornode -- `
  sh -c "grep -E '^(pruning|min-retain-blocks|halt-height)' /data/.thornode/config/app.toml"
```

### Which snapshot height was it seeded from?

```powershell
kubectl exec -n thorchain thornode-0 -c thornode -- cat /data/.thornode/.snapshot-restored
```

### Disk

```powershell
kubectl exec -n thorchain thornode-0 -c thornode -- df -h /data
```

Expand (PowerShell quoting — note the doubled quotes):

```powershell
kubectl patch pvc thornode-data -n thorchain -p '{\"spec\":{\"resources\":{\"requests\":{\"storage\":\"1Ti\"}}}}'
```

### Resource pressure

```powershell
kubectl top pod thornode-0 -n thorchain --containers
kubectl describe node <node-name> | Select-String -Pattern "Allocated resources" -Context 0,12
```

---

## 7. Force a fresh snapshot re-restore later

```powershell
kubectl exec -n thorchain thornode-0 -c thornode -- rm -f /data/.thornode/.snapshot-restored
kubectl delete pod thornode-0 -n thorchain
```

`node_key.json` and `priv_validator_key.json` are preserved across this, so the node
keeps its p2p identity.

---

## 8. Keeping up with versions

Node halts of this kind recur every time THORChain bumps its consensus version and
the image tag is left behind. Check before and after every network upgrade:

```powershell
# what the network runs
curl.exe -s "https://gateway.liquify.com/chain/thorchain_api/thorchain/version"
# what your pod runs
kubectl get sts thornode -n thorchain -o jsonpath="{.spec.template.spec.containers[0].image}"
```

If `current` is ahead of your tag, bump the image in `04-statefulset.yaml`
(**both** the `init-keys` and `thornode` containers) and roll:

```powershell
kubectl set image sts/thornode -n thorchain `
  thornode=registry.gitlab.com/thorchain/thornode:mainnet-<version> `
  init-keys=registry.gitlab.com/thorchain/thornode:mainnet-<version>
kubectl rollout status sts/thornode -n thorchain
```

Upstream's Helm chart automates this with a `versions:` height→image map. Without
Helm this is a manual step you have to own.

---

## 9. Reference

- <https://docs.thorchain.org/thornodes/fullnode/thornode-docker>
- <https://docs.thorchain.org/thornodes/fullnode/thornode-kubernetes>
- <https://gitlab.com/thorchain/thornode/-/blob/develop/config/default.yaml>
- <https://gitlab.com/thorchain/thornode/-/blob/develop/build/scripts/fullnode.sh>
- <https://gitlab.com/thorchain/devops/node-launcher/-/blob/master/thornode/values.yaml>
