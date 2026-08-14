# THORChain Node — AKS Setup Wiki


**Namespace:** `thorchain` | **Chain:** `thorchain-1` | **Sync:** Snapshot sync (Liquify `mainnet-pruned`) via `thornode`
**Status:** ✅ Running — restored from Liquify pruned snapshot


### ⚠️ Internal LB is `<pending>` — use the NodePort endpoint for testing


The Azure internal LoadBalancer cannot be provisioned: the AKS subnet
`10.202.17.192/27` is full (`SubnetIsFull`, see Issue 9). **This does not block access.**
A LoadBalancer Service still allocates NodePorts, and kube-proxy programs them on every
node — so the node is reachable today at a node IP, consuming **zero** subnet addresses.


**Use these for testing:**


| Service | Endpoint |
|---|---|
| CometBFT RPC | `http://10.202.17.198:30399` |
| REST / LCD API | `http://10.202.17.198:32210` |
| gRPC | `10.202.17.198:30609` |


```powershell
curl.exe -s --max-time 20 http://10.202.17.198:30399/status
curl.exe -s --max-time 20 http://10.202.17.198:32210/thorchain/lastblock
```


> **`10.202.17.198` is a NODE IP, not a service IP.** It survives reboots but **not**
> node replacement — AKS node-image upgrades, autoscaler scale-down, VMSS reimage and
> auto-repair all replace the VM and it takes a new address. Re-read it any time with
> `kubectl get nodes -o wide`, and note that **any** node IP in the cluster works, not
> just this one (`externalTrafficPolicy` defaults to `Cluster`, so kube-proxy forwards
> to the pod wherever it runs). Hand consumers the full node list, not one address.
>
> The NodePort numbers **are** stable — they are pinned in `05-service.yaml`
> (`30399` / `32210` / `30609`), so a Service recreate cannot reassign them.
>
> In-cluster consumers should ignore all of this and use
> `http://thornode.thorchain.svc.cluster.local:1317`, which is permanently stable.


---


## Table of Contents


1. [Architecture Overview](#1-architecture-overview)
2. [Prerequisites](#2-prerequisites)
3. [Manifest Files](#3-manifest-files)
4. [Step-by-Step Deployment](#4-step-by-step-deployment)
5. [Issues Encountered & Resolutions](#5-issues-encountered--resolutions)
6. [Verification — curl Commands](#6-verification--curl-commands)
7. [Postman Collection](#7-postman-collection)
8. [pvc-watcher Configuration](#8-pvc-watcher-configuration)
9. [Upgrade Procedure](#9-upgrade-procedure)
10. [Monitoring & Ops Commands](#10-monitoring--ops-commands)
11. [Archive Node — Full History](#11-archive-node--full-history) ← **different build, read before deploying**


---


## 1. Architecture Overview


### Public Image — No ACR Required


`registry.gitlab.com/thorchain/thornode:mainnet-3.19.3` is public on GitLab Container Registry — no custom build or ACR needed.


**The THORChain image already does the whole setup itself** — there is no need to
hand-build config or download genesis. Its default `CMD` is `/scripts/fullnode.sh`,
which on every boot:


```bash
if [ ! -f ~/.thornode/config/genesis.json ]; then
  init_chain                     # generates node_key.json + priv_validator_key.json
  rm -rf ~/.thornode/config/genesis.json
fi
thornode render-config           # writes config.toml + app.toml, resolves live seeds,
exec thornode start              # fetches genesis if missing
```


`render-config` is the important part. It fetches the active validator list over
**HTTPS/443** from `https://gateway.liquify.com/chain/thorchain_api/thorchain/nodes`,
probes each on `:27147/status` for its CometBFT node ID, and writes real
`nodeid@ip:27146` entries into `seeds`. **Never override the entrypoint and never hand-edit
the TOMLs** — render-config rewrites them on every boot, and skipping it is what broke the
first attempt (Issue 2).


### Solution: Init Container Pattern (No ACR Required)


Only the snapshot restore and a non-root workaround need init containers:


```
Pod: thornode-0
│
├── Init 1 — init-passwd (busybox:1.36, UID 1025)
│   └── Writes /etc/passwd with a UID 1025 entry into a shared emptyDir
│
├── Init 2 — init-snapshot (alpine:3.19, UID 0)
│   ├── Queries https://snapshot-api.liquify.com for the newest mainnet-pruned snapshot
│   ├── Mints a signed URL (expires in 600s — re-minted on every retry)
│   ├── aria2c 16-connection download of ~40 GB .tar.gz → /data/snapshot.tar.gz
│   ├── Extracts to /data/.thornode (~200 GB), deletes tarball
│   └── chmod -R a+rwX so UID 1025 can use it
│
├── Init 3 — init-keys (thornode:mainnet-3.19.3, UID 1025)
│   └── thornode init local --chain-id thorchain-1  (only if node_key.json missing)
│
└── Main — thornode (thornode:mainnet-3.19.3, UID 1025)
    └── Image default CMD → /scripts/fullnode.sh → render-config → thornode start
```


**All images are public. No ACR required.**


### Key Design Decisions


| Decision | Reason |
|---|---|
| `thornode:mainnet-3.19.3`, pinned exactly | Must match the network's live consensus version. A mismatched binary halts on app-hash divergence — see Issue 1 |
| Run the image's own `CMD`, no `command:`/`args:` override | `fullnode.sh` → `render-config` is the only thing that populates `seeds` |
| `alpine:3.19` + root for `init-snapshot` | Needs `apk add aria2 tar gzip` — Liquify serves `.tar.gz`, aria2c gives 16-way parallel download (~147 MiB/s measured) |
| `runAsUser: 1025` everywhere except `init-snapshot` | Gatekeeper/Azure Policy enforces no-root containers |
| `/etc/passwd` shim via emptyDir + subPath | THORChain Go code calls `os/user.Current()` and panics on `unknown userid 1025` |
| `init-snapshot` **before** `init-keys` | Root has `CAP_DAC_OVERRIDE` dropped and cannot read 1025-owned `node_key.json` — see Issue 7 |
| Snapshot sync from Liquify | Nine Realms infrastructure is decommissioned; Liquify is the community provider. Genesis sync is a multi-week replay of 27M+ blocks |
| `THOR_*` env vars, never `sed` on the TOMLs | Every key in the upstream `config/default.yaml` maps to `THOR_<UPPER_SNAKE>` via viper `AutomaticEnv` |
| `persistent_peers` pinned to 20 verified nodes | Removes any dependence on seed discovery succeeding |
| `EXTERNAL_IP` deliberately **not** set | It would advertise an RFC1918 address to public peers who cannot dial it |
| PVC `400Gi`, `Premium_LRS`, `cachingMode: None` | ~240 GB restore peak; write-heavy LevelDB should not sit behind host caching |


---


## 2. Prerequisites


### Cluster Requirements


- AKS cluster with Azure Disk CSI driver (`disk.csi.azure.com`)
- **Gatekeeper/Azure Policy** enforcing no-root containers (observed in this cluster)
- Node pool: `Standard_D2s_v3` (2 vCPU / 8Gi) — this is **below** upstream's recommendation


### Egress required


| Destination | Port | Needed for |
|---|---|---|
| `gateway.liquify.com` | 443 | seed list, consensus version |
| `snapshot-api.liquify.com`, `snapshots-*.liquify.com` | 443 | snapshot download |
| any active validator IP | 27147 | seed node-ID discovery, genesis fetch |
| any active validator IP | 27146 | block gossip (**mandatory**) |


Verify before deploying:


```powershell
kubectl run netcheck -n thorchain --rm -it --restart=Never --image=alpine:3.19 -- sh -c "apk add --no-cache curl >/dev/null 2>&1; for ip in 167.235.109.114 173.234.136.17 80.91.65.181; do echo -n \"\$ip 27146:\"; nc -z -w 8 \$ip 27146 && echo -n ' OPEN' || echo -n ' BLOCKED'; echo -n ' 27147:'; curl -s -o /dev/null -w '%{http_code}' --max-time 12 http://\$ip:27147/status; echo; done; curl -s -o /dev/null -w '443 liquify=%{http_code}\n' --max-time 15 https://gateway.liquify.com/chain/thorchain_api/thorchain/version"
```


> **Always probe 3+ validators.** 2 of the ~95 active validators return `503` from their
> own RPC reverse-proxy. A single-IP test led to a wrong "egress proxy" conclusion here.


### Verify schedulable capacity before deploying


```powershell
kubectl describe nodes | Select-String -Pattern "Allocated resources" -Context 0,6
```


THORChain needs `250m CPU` + `1Gi memory` free on at least one node **as configured here**.
Upstream's own Helm chart requests **4 CPU / 4Gi with no limits** — see Issue 5 for why this
deployment runs far below that and what it costs.


---


## 3. Manifest Files


| File | Purpose |
|---|---|
| `00-namespace.yaml` | Namespace `thorchain` |
| `01-storageclass.yaml` | `thorchain-storage` — Premium_LRS, `cachingMode: None`, `reclaimPolicy: Retain` |
| `02-pvc.yaml` | `thornode-data` — 400Gi |
| `03-configmap-scripts.yaml` | `init-passwd.sh` + `init-snapshot.sh` |
| `04-statefulset.yaml` | StatefulSet with 3 init containers + main `thornode` container |
| `05-service.yaml` | Headless + ClusterIP + RPC/API Internal LB |


### Storage Class Note
`reclaimPolicy: Retain` is **mandatory** — blockchain data is irreplaceable. Never use
`Delete`. Be aware of the cost: see Issue 12.


---


## 4. Step-by-Step Deployment


### Step 0 — Decide PRUNED or ARCHIVE before you apply anything


This changes the disk tier, the node pool and the image tag. Getting it wrong means
re-downloading terabytes, so decide first.


| | **PRUNED** (sections 1-10) | **ARCHIVE** (section 11) |
|---|---|---|
| Answers "balance at height N"? | only recent heights | **any height ≥ 17,562,001** |
| Snapshot | `mainnet-pruned`, 39.6 GB | `mainnet-archive`, 2,519 GB |
| Disk | 400Gi (P20) | **16Ti (P70)** |
| Image at deploy | `mainnet-3.19.3` | **`mainnet-3.16.4`** |
| CPU / memory | 250m / 1Gi (constrained) | 4 CPU / 8-16Gi |
| Time to usable | ~45 min | ~5 h download + multi-day replay |
| Image swaps during catch-up | none | **3, at fixed heights** |


**Everything in sections 1-10 describes the PRUNED build.** If you need archive, read
section 11 first — the two differ from the very first `kubectl apply`.


### Step 1 — Copy manifests to jumpbox


```powershell
mkdir C:\Users\<user>\thorchain
# Copy all 6 yaml files from AKS_SETUP\manifests-v2\ to this directory
```


### Step 2 — Confirm the image matches the network's consensus version


**Do this every single time.** A stale tag is the single most damaging failure mode.


```powershell
curl.exe -s "https://gateway.liquify.com/chain/thorchain_api/thorchain/version"
# {"current":"3.19.3","next":"3.19.3","next_since_height":26897608,"querier":"3.19.0"}
```


`current` must match the tag in `04-statefulset.yaml` (both `init-keys` and `thornode`).


### Step 3 — Apply in order


No URL editing needed — `init-snapshot.sh` queries the Liquify API for the newest
snapshot and mints a fresh signed URL at pod start.


```powershell
cd C:\Users\<user>\thorchain

kubectl apply -f 00-namespace.yaml
kubectl apply -f 01-storageclass.yaml
kubectl apply -f 02-pvc.yaml
kubectl apply -f 03-configmap-scripts.yaml
kubectl apply -f 04-statefulset.yaml
kubectl apply -f 05-service.yaml
```


### Step 4 — Watch pod startup


```powershell
# Pod phases: Pending → Init:0/3 → Init:1/3 → Init:2/3 → Running
kubectl get pods -n thorchain -w
```


Expected timelines:


| Phase | Duration | What's happening |
|---|---|---|
| `Init:0/3` (init-passwd) | ~5 seconds | writes `/etc/passwd` with UID 1025 |
| `Init:1/3` (init-snapshot) | ~30-60 minutes | 40 GB download + ~200 GB extract + chmod |
| `Init:2/3` (init-keys) | ~10 seconds | `thornode init local --chain-id thorchain-1` |
| `Running 0/1` → `1/1` | ~2-5 minutes | render-config resolves seeds, then `thornode start` |


### Step 5 — Monitor init container logs


```powershell
kubectl logs -n thorchain thornode-0 -c init-passwd
kubectl logs -n thorchain thornode-0 -c init-snapshot -f
kubectl logs -n thorchain thornode-0 -c init-keys
kubectl logs -n thorchain thornode-0 -c thornode -f
```


The extract phase prints nothing for 15-25 minutes. Track it by disk instead:


```powershell
while ($true) { kubectl exec -n thorchain thornode-0 -c init-snapshot -- df -h /data | Select-String "/data"; Start-Sleep 60 }
```


`Used` climbing = working. Flat for 10+ minutes with zero CPU = stuck.


### Step 6 — Verify peers and forward progress


```powershell
kubectl exec -n thorchain thornode-0 -c thornode -- curl -s --max-time 20 http://localhost:27147/net_info | ConvertFrom-Json | ForEach-Object result | Select-Object n_peers
```


`n_peers = 0` means it will never sync. Everything else is secondary.


### Step 7 — Add to pvc-watcher


```powershell
kubectl edit configmap pvc-watcher-script -n pvc-watcher
```


Add to `NODE_CONFIG`:
```
"thorchain|thornode-0|/data/.thornode|thornode-data|80|400|1000"
```


---


## 5. Issues Encountered & Resolutions


### Issue 1 — Node ran for days at a frozen height (`catching_up: true`, no errors)
The image was pinned to `mainnet-3.18.1` while the network's live consensus version was
**3.19.3**, and had been since block **26,897,608** — roughly 400,000 blocks before the
snapshot being restored. A 3.18.1 binary contains no 3.19.x consensus logic, so the first
time a 3.19-gated code path executes it computes a different app hash than the network
and CometBFT rejects every subsequent block.


**Fix:** Pin `registry.gitlab.com/thorchain/thornode:mainnet-3.19.3`. Verify against
`/thorchain/version` before every deploy and after every network upgrade.


> Note: this was **not** what froze the original node — Issue 2 was. But it would have
> halted it within thousands of blocks regardless, so both were fixed together.


### Issue 2 — `n_peers = 0`, height frozen at snapshot+1, 0 restarts, clean logs
The original manifest overrode the image entrypoint with `thornode start` to avoid
`render-config`, on the belief that render-config overwrites `genesis.json`. **That belief
is wrong** — `config.go: thornodeFetchGenesis()` returns early whenever the file exists.


Skipping render-config meant nothing ever populated `seeds`. A separate `init-config`
container then made it worse: it set `seeds = ""` explicitly, deleted `addrbook.json` on
**every** pod start, and built `persistent_peers` from Liquify's `net_info` `.url` field —
where inbound peers carry an **ephemeral source port**, not `:27146`, so most entries were
undialable. CometBFT had nothing valid to dial and logged nothing, because it never tried.


**Fix:** Delete `init-config` entirely. Run the image's own `CMD`. If the TOMLs ever
genuinely need pinning, the supported way is `THORNODE_PRESERVE_CONFIG_TOML` /
`THORNODE_PRESERVE_APP_TOML` — not deleting the entrypoint.


### Issue 3 — Config silently depended on restart count
Init order was `init-config` → `init-snapshot`. The snapshot tarball ships its own
`config/config.toml`, so on first boot it overwrote every patch; on later boots the
snapshot sentinel short-circuited and the patches survived.


**Fix:** Restore the snapshot first, then **delete** the producer's `config.toml` and
`app.toml` so render-config writes correct ones. Never inherit a stranger's peers,
`external_address` or pruning settings.


### Issue 4 — `EXTERNAL_IP` set to the internal LB address
The original runbook instructed setting `EXTERNAL_IP` to the internal Azure LB IP.
`render-config` turns that into `external_address = "<ip>:27146"` and advertises it to the
entire network — an RFC1918 address no public peer can dial. Peers that fail to dial back
deprioritise and eventually drop the node.


**Fix:** Leave `EXTERNAL_IP` unset. A full node needs only **outbound** connectivity.


### Issue 5 — `0/5 nodes are available: Insufficient cpu, Insufficient memory`
Upstream's chart requests 4 CPU / 4Gi with no limits. The shared pool is five
`Standard_D2s_v3` nodes (~1.9 CPU / ~5.7 GiB allocatable each), 71-86% committed, and the
autoscaler was at max node group size. A 4-CPU request cannot schedule on **any** node —
the whole node is under 2 CPU.


Measured free capacity:
```
node1 266m/2166Mi   node2 365m/1221Mi   node3 342m/1046Mi
node4 555m/1932Mi   node5 361m/1564Mi
```


**Fix (constrained):** requests `250m` / `1Gi` so it fits all five nodes, memory limit
`3Gi`, and **no CPU limit** so it bursts into idle CPU (those percentages are
*reservations*, not usage).


**Proper fix:** a dedicated node pool. Not available on this shared cluster.
```powershell
az aks nodepool add -g <rg> --cluster-name <cluster> --name thornode --node-count 1 `
  --node-vm-size Standard_D8s_v3 --node-osdisk-size 128 `
  --labels workload=thornode --node-taints workload=thornode:NoSchedule
```


### Issue 6 — Still `Pending` after lowering the main container's requests
A pod's effective request is `max(init container requests, sum of app containers)`.
`init-snapshot` was requesting `500m`, which set the whole pod's CPU request to 500m
regardless of the main container's 250m.


**Fix:** No init container may request more than the main container. Dropped
`init-snapshot` to `250m`.


### Issue 7 — `cp: can't open '.../node_key.json': Permission denied` (running as root)
`init-snapshot` runs as UID 0 but with `capabilities: drop: ["ALL"]`, which removes
`CAP_DAC_OVERRIDE` — the capability that normally lets root bypass file permissions.
`thornode init` writes `node_key.json` as mode `0600` owned by 1025, so root could not
read it.


**Fix:** Reorder to `init-passwd` → `init-snapshot` → `init-keys`. The snapshot creates
everything as root, `chmod -R a+rwX` hands write access to 1025, and `init-keys` then
writes its own keys. Root never has to read a file it does not own.


### Issue 8 — `[snapshot] WARN: snapshot had no genesis.json` → 2 KB placeholder genesis
Liquify's pruned tarball ships state but no `genesis.json`, so `init-keys`'
`thornode init` wrote a 2,080-byte placeholder (`"thorchain": {}`, `mint_denom: "stake"`).


**This is harmless and needs no fix.** CometBFT v0.38 `node/setup.go` stores the entire
genesis doc inside `data/state.db` and prefers it:
```go
genDoc, err := loadGenesisDoc(stateDB)   // state.db FIRST
if err != nil {
  genDoc, err = genesisDocProvider()      // only then genesis.json
  saveGenesisDoc(stateDB, genDoc)
}
```
The snapshot's `state.db` already carries the real genesis, so `genesis.json` is never
read. Confirm any time:
```powershell
kubectl exec -n thorchain thornode-0 -c thornode -- curl -s --max-time 20 "http://localhost:27147/genesis_chunked?chunk=0"
```
Expect `"chain_id":"thorchain-1"`, `"initial_height":"17562001"`, `total: 3`.


### Issue 9 — Internal LB stuck at `<pending>` — `SubnetIsFull`
```
SyncLoadBalancerFailed  RESPONSE 400: SubnetIsFull
Subnet .../subnets/default-0 with address prefix 10.202.17.192/27
does not have enough capacity for 1 IP addresses.
```
A `/27` is 32 addresses; Azure reserves 5, leaving 27 for every node and every internal LB
in this shared cluster. The subnet belongs to another team.


**Fix:** None available from the manifest. Reach the node via port-forward or in-cluster
DNS (Section 6). If the network team frees addresses or dedicates a subnet, add the
subnet annotation:
```yaml
service.beta.kubernetes.io/azure-load-balancer-internal: "true"
service.beta.kubernetes.io/azure-load-balancer-internal-subnet: "<subnet-name>"
```
> The pods draw from this same exhausted `/27`. A future pod `Pending` on IP allocation
> rather than CPU has the same root cause.


### Issue 10 — Signed snapshot URL expires mid-download
Liquify's download URLs expire in **600 seconds**. A 40 GB transfer does not finish in
10 minutes, so any connection drop retried against an expired signature and failed.


**Fix:** `init-snapshot.sh` re-mints a fresh signed URL on every retry attempt and lets
`aria2c --continue=true` resume the partial file. Never paste a signed URL into a manifest.


### Issue 11 — Probes reported healthy while the chain was frozen for 5 days
The original probes were `tcpSocket` on the p2p port, which stays open on a completely
stuck node.


**Fix:** HTTP `/health` on the RPC port for startup/readiness, and **no liveness probe** —
restarting a node that is behind does not help it catch up, and killing it mid-compaction
risks LevelDB corruption.


### Issue 12 — 15 orphaned PVs / ~5 TiB of Azure disks billing
`reclaimPolicy: Retain` plus 15 redeploy attempts left 15 `Released` PVs, each with a live
Azure managed disk. Billing rounds up by tier: 200/250Gi → P15 (256 GiB), 300Gi → P20
(512 GiB).


**Fix:** Audit after every redeploy. Capture the disk map **before** deleting PVs, or you
lose the ID mapping:
```powershell
kubectl get pv -o custom-columns=NAME:.metadata.name,NS:.spec.claimRef.namespace,PVC:.spec.claimRef.name,STATUS:.status.phase,SIZE:.spec.capacity.storage,DISK:.spec.csi.volumeHandle
kubectl delete pv <released-pv-name>
az disk delete --ids "<volumeHandle>" --yes
```
> **Never** delete Azure disks filtered by `diskState=='Unattached'`. The AKS node resource
> group holds disks for every team in the cluster, and another chain's disk shows as
> Unattached while its pod reschedules. Match by the ID from the PV map only.


### Issue 13 — `kubectl exec ... curl /status` hangs during catch-up
CometBFT's RPC shares a lock with block execution, so `/status` can block for minutes
while replaying.


**Fix:** Always bound it with `--max-time 20`, or read height from the logs instead:
```powershell
kubectl logs -n thorchain thornode-0 -c thornode --tail=300 | Select-String "committed state" | Select-Object -Last 1
```


### Issue 14 — Alarming `ERR ... consensus_failure_logger.go` lines in normal operation
```
fail to handle MsgSwap  error="emit asset ... less than price limit"
msg add liquidity fail validation  error="unable to add liquidity while chain has paused LP actions"
fail to get volume  error="Total volume not found: pool=ETH.QWERTY"
```
These are application-level rejections of user transactions, not node faults — a user hit
their own slippage limit, LP is paused network-wide, dead pools have no volume stats. The
filename is simply where THORChain routes handler errors.


**Fix:** None. Every full node on the network logs these identically.


### Summary of All Fixes


| # | Issue | Root Cause | Fix |
|---|---|---|---|
| 1 | Frozen height, no errors | Binary 3.18.1 vs network consensus 3.19.3 | Pin `mainnet-3.19.3`; check `/thorchain/version` every deploy |
| 2 | `n_peers = 0` | Entrypoint overridden, `seeds` blanked, addrbook deleted, ephemeral-port peers | Run the image's own `CMD`; delete `init-config` |
| 3 | Config depended on restart count | `init-config` ran before `init-snapshot` | Snapshot first, then delete the producer's TOMLs |
| 4 | Peers drop the node | `EXTERNAL_IP` = internal LB (RFC1918) | Leave `EXTERNAL_IP` unset |
| 5 | Pod `Pending` | 4 CPU request vs 2 vCPU nodes at max autoscale | 250m/1Gi requests, no CPU limit |
| 6 | Still `Pending` | Init container requested more than the main container | Cap init requests at the main container's |
| 7 | `Permission denied` as root | `CAP_DAC_OVERRIDE` dropped; 0600 file owned by 1025 | Reorder: snapshot before keys |
| 8 | 2 KB placeholder genesis | Liquify tarball ships no `genesis.json` | None needed — CometBFT prefers the copy in `state.db` |
| 9 | LB `<pending>` | `/27` subnet exhausted, owned by another team | Port-forward + in-cluster DNS |
| 10 | Download fails partway | Signed URL expires in 600s | Re-mint per retry, `aria2c --continue` |
| 11 | Probes green while frozen | `tcpSocket` on p2p | HTTP `/health` on RPC; no liveness probe |
| 12 | ~5 TiB orphaned disks | `Retain` + 15 redeploys | Audit and delete by PV disk map, never by `Unattached` |
| 13 | `exec curl` hangs | RPC shares a lock with block execution | `--max-time`, or read height from logs |
| 14 | Red `ERR` lines | App-level tx rejections | Expected; ignore |


---


## 6. Verification — curl Commands


> The Internal LB is `<pending>` (Issue 9). Use the **NodePort** endpoint below — it is
> the primary access path today. Port-forward and in-cluster DNS also work.


### 6.1 NodePort — the endpoint to give consumers


Reachable from anywhere in the VNet. No LB, no subnet IP required.


| Service | Port | NodePort | Endpoint |
|---|---|---|---|
| CometBFT RPC | 27147 | **30399** | `http://10.202.17.198:30399` |
| REST / LCD | 1317 | **32210** | `http://10.202.17.198:32210` |
| gRPC | 9090 | **30609** | `10.202.17.198:30609` |


**Always pass `--max-time`.** CometBFT's RPC shares a lock with block execution, so
`/status` can block for minutes during catch-up and an unbounded `curl` will appear to
hang — that is not a connectivity failure.


```powershell
# is it alive?
curl.exe -s --max-time 20 http://10.202.17.198:30399/health

# sync state — the number that matters
curl.exe -s --max-time 20 http://10.202.17.198:30399/status | ConvertFrom-Json | ForEach-Object result | ForEach-Object sync_info | Select-Object latest_block_height, latest_block_time, catching_up

# peers — 0 means it will never sync
curl.exe -s --max-time 20 http://10.202.17.198:30399/net_info | ConvertFrom-Json | ForEach-Object result | Select-Object n_peers

# THORChain REST
curl.exe -s --max-time 20 http://10.202.17.198:32210/thorchain/lastblock
curl.exe -s --max-time 20 http://10.202.17.198:32210/thorchain/version
curl.exe -s --max-time 20 http://10.202.17.198:32210/thorchain/pools
curl.exe -s --max-time 20 http://10.202.17.198:32210/thorchain/inbound_addresses
```


**Is the data fresh?** Consumers must check this before trusting any response — a
catching-up node answers happily with days-old state:


```powershell
$local  = (curl.exe -s --max-time 20 http://10.202.17.198:30399/status | ConvertFrom-Json).result.sync_info.latest_block_height
$remote = (curl.exe -s "https://gateway.liquify.com/chain/thorchain_rpc/status" | ConvertFrom-Json).result.sync_info.latest_block_height
"local=$local  network=$remote  behind=$([int]$remote - [int]$local)"
```


Under ~100 blocks behind is current. Thousands means it is still catching up.


**If a NodePort stops answering** — the node was probably replaced and its IP changed.
Get the current list; any node in the cluster serves the same NodePorts:


```powershell
kubectl get nodes -o wide
kubectl get nodes -o jsonpath="{range .items[*]}{.status.addresses[?(@.type=='InternalIP')].address}{'\n'}{end}"
```


Distinguish a blocked port from a slow app — a hang is usually an NSG, not the node:


```powershell
Test-NetConnection -ComputerName 10.202.17.198 -Port 30399
```


`TcpTestSucceeded : False` → an NSG is blocking the NodePort range 30000-32767 inside
the VNet. Prove the Service itself is healthy from inside the cluster:


```powershell
kubectl get svc thornode-rpc -n thorchain
kubectl get endpoints thornode-rpc -n thorchain
```


`ENDPOINTS` must list a pod IP. If it says `<none>`, the pod is not Ready and no endpoint
will work.


### 6.2 Port-forward (jumpbox only)


```powershell
kubectl port-forward -n thorchain svc/thornode 1317:1317 27147:27147
```


### RPC — CometBFT (`27147`)


```powershell
curl.exe -s http://localhost:27147/health
curl.exe -s http://localhost:27147/status
curl.exe -s http://localhost:27147/net_info
curl.exe -s "http://localhost:27147/block?height=27356957"
curl.exe -s "http://localhost:27147/genesis_chunked?chunk=0"
```


### REST — THORChain API (`1317`)


```powershell
curl.exe -s http://localhost:1317/thorchain/lastblock
curl.exe -s http://localhost:1317/thorchain/version
curl.exe -s http://localhost:1317/thorchain/network
curl.exe -s http://localhost:1317/thorchain/pools
curl.exe -s http://localhost:1317/thorchain/pool/BTC.BTC
curl.exe -s http://localhost:1317/thorchain/inbound_addresses
curl.exe -s http://localhost:1317/thorchain/nodes
curl.exe -s http://localhost:1317/thorchain/queue
```


Swagger UI: <http://localhost:1317/swagger/>


### REST — Cosmos base endpoints (`1317`)


```powershell
curl.exe -s http://localhost:1317/cosmos/base/tendermint/v1beta1/node_info
curl.exe -s http://localhost:1317/cosmos/base/tendermint/v1beta1/syncing
curl.exe -s http://localhost:1317/cosmos/base/tendermint/v1beta1/blocks/latest
```


### Metrics (`26660`)


```powershell
kubectl port-forward -n thorchain svc/thornode 26660:26660
curl.exe -s http://localhost:26660/metrics
```


> While `catching_up: true`, every answer is stale and the quote endpoints
> (`/thorchain/quote/swap`) **refuse outright** — they enforce a 3-minute freshness limit.


### In-cluster (from other pods)


```
http://thornode.thorchain.svc.cluster.local:1317
http://thornode.thorchain.svc.cluster.local:27147
```


### Sync monitoring loop


```powershell
kubectl port-forward -n thorchain svc/thornode 27147:27147
# in a second window:
while ($true) {
  $status = curl.exe -s --max-time 20 http://localhost:27147/status | ConvertFrom-Json
  $latest = [int64]$status.result.sync_info.latest_block_height
  $catchingUp = $status.result.sync_info.catching_up
  Write-Host "$(Get-Date -Format s) | height=$latest | catching_up=$catchingUp"
  Start-Sleep -Seconds 15
}
```


### How far behind is it


```powershell
$local  = (kubectl exec -n thorchain thornode-0 -c thornode -- curl -s --max-time 20 http://localhost:27147/status | ConvertFrom-Json).result.sync_info.latest_block_height
$remote = (curl.exe -s "https://gateway.liquify.com/chain/thorchain_rpc/status" | ConvertFrom-Json).result.sync_info.latest_block_height
"local=$local  network=$remote  behind=$([int]$remote - [int]$local)"
```


THORChain blocks are ~6s, so the chain grows 10 blocks/minute. **Catch-up must exceed
~10 blocks/min** or it will never converge.


---


## 7. Postman Collection


No Postman collection yet — **to be added**.


---


## 8. pvc-watcher Configuration


```powershell
kubectl edit configmap pvc-watcher-script -n pvc-watcher
```


Add:
```
"thorchain|thornode-0|/data/.thornode|thornode-data|80|400|1000"
```


Meaning:
- Namespace: `thorchain`
- Pod: `thornode-0`
- Data path: `/data/.thornode`
- PVC: `thornode-data`
- Expand when usage exceeds `80%`
- Start size: `400Gi`
- Max size: `1000Gi`


> Sizing: ~40 GB tarball + ~200 GB extracted = ~240 GB restore peak. Steady growth is
> ~15-25 GB/month on `pruning = "default"`. An **archive** node
> (`THOR_COSMOS_PRUNING=nothing`, THORChain's own default) is multi-terabyte — use 4Ti+.


---


## 9. Upgrade Procedure


THORChain does **not** use Cosmovisor in this deployment. Upstream's Helm chart automates
version bumps with a `versions:` height→image map; without Helm this is **manual and you
must own it**.


**Node halts of the Issue 1 kind recur every time THORChain bumps its consensus version
and the image tag is left behind.** Check before and after every network upgrade:


```powershell
# what the network runs
curl.exe -s "https://gateway.liquify.com/chain/thorchain_api/thorchain/version"
# what the pod runs
kubectl get sts thornode -n thorchain -o jsonpath="{.spec.template.spec.containers[0].image}"
```


If `current` is ahead of the tag, bump **both** containers and roll:


```powershell
kubectl set image sts/thornode -n thorchain `
  thornode=registry.gitlab.com/thorchain/thornode:mainnet-<version> `
  init-keys=registry.gitlab.com/thorchain/thornode:mainnet-<version>
kubectl rollout status sts/thornode -n thorchain
```


Available tags: <https://gitlab.com/thorchain/thornode/container_registry>


The existing PVC is reused, so chain data is preserved across the restart.


### Force a fresh snapshot re-restore (only if the store is corrupt)


```powershell
kubectl exec -n thorchain thornode-0 -c thornode -- rm -f /data/.thornode/.snapshot-restored
kubectl delete pod thornode-0 -n thorchain
kubectl logs -n thorchain thornode-0 -c init-snapshot -f
```


`node_key.json` and `priv_validator_key.json` survive this, so the node keeps its p2p
identity.


---


## 10. Monitoring & Ops Commands


### Pod / StatefulSet / PVC


```powershell
kubectl get pods -n thorchain
kubectl get sts -n thorchain
kubectl get pvc -n thorchain
kubectl describe pod thornode-0 -n thorchain
kubectl describe pvc thornode-data -n thorchain
```


### Logs


```powershell
kubectl logs -n thorchain thornode-0 -c init-passwd
kubectl logs -n thorchain thornode-0 -c init-snapshot
kubectl logs -n thorchain thornode-0 -c init-keys
kubectl logs -n thorchain thornode-0 -c thornode -f
```


Fatal-pattern grep (should return nothing):
```powershell
kubectl logs -n thorchain thornode-0 -c thornode --tail=4000 | Select-String -Pattern "wrong Block.Header.AppHash|Consensus Failure|panic|OOM|found 0 p2p seeds"
```


### Services / Endpoints


```powershell
kubectl get svc -n thorchain
kubectl get endpoints -n thorchain
kubectl describe svc thornode-rpc -n thorchain
```


### Exec / Config Inspection


```powershell
kubectl exec -n thorchain thornode-0 -c thornode -- sh -c "ls -lah /data/.thornode"
kubectl exec -n thorchain thornode-0 -c thornode -- sh -c "grep -E '^(seeds|persistent_peers|external_address|pex|addr_book_strict|laddr) ' /data/.thornode/config/config.toml"
kubectl exec -n thorchain thornode-0 -c thornode -- sh -c "grep -E '^(pruning|min-retain-blocks|halt-height)' /data/.thornode/config/app.toml"
kubectl exec -n thorchain thornode-0 -c thornode -- cat /data/.thornode/.snapshot-restored
kubectl exec -n thorchain thornode-0 -c thornode -- df -h /data
```


### Resource pressure


```powershell
kubectl top pod thornode-0 -n thorchain --containers
kubectl describe node <node-name> | Select-String -Pattern "Allocated resources" -Context 0,12
```


### Rollout / Restart


```powershell
kubectl rollout restart statefulset thornode -n thorchain
kubectl rollout status statefulset thornode -n thorchain
```


---


## 11. Archive Node — Full History


> **This is a different build, not a toggle.** Do not follow sections 1-10 and then
> "switch on archive". The disk tier, node pool and image tag all differ from the first
> apply. Deploy it in its own namespace, on its own node pool, with its own PVC.


### 11.1 When you need it


Only when you must answer **"what was this wallet's balance at block N"** — crypto
auditing, reconciliation, historical reporting. Nothing else justifies the cost.


The query is a standard Cosmos header, not a special API:


```bash
curl -H "x-cosmos-block-height: 27300000" \
  http://<node>:1317/cosmos/bank/v1beta1/balances/thor1dheycdevq39qlkxs2a6wuuzyn4aqxhve4qxtxt
```


On a **pruned** node that returns:


```json
{"code":2, "message":"failed to load state at height 26000000; no commit info found (latest height: 27374319): not found"}
```


That error is the whole reason this section exists. An archive node
(`pruning = "nothing"`) keeps every height's commit info, so the same query succeeds.


> Balances come back in base units (1e8) with THORChain denoms — `rune`, `btc/btc`,
> `avax/usdc-0xb97ef9...`. Native RUNE and synths are **separate denoms**. Agree the
> denom handling with the audit team before building tooling on the output.


### 11.2 What one archive node can and cannot cover


THORChain has restarted twice via state-export hard forks. **Each era is a separate
network with its own chain-id — no single node can serve all history.**


| Era | `chain_id` | Blocks | Status |
|---|---|---|---|
| v0 | not publicly exposed | 1 → 4,786,560 | dead |
| v1 | `thorchain-mainnet-v1` | 4,786,560 → 17,562,001 | dead |
| current | `thorchain-1` | 17,562,001 → live | ✅ live |


An archive node built here covers **`thorchain-1` only — heights ≥ 17,562,001
(from 2024-09-04)**. Audits reaching further back need separate read-only nodes:


| Range | Snapshot | Image | Note |
|---|---|---|---|
| 4,786,560 → 17,562,001 | `mainnet-archive-v1` (4,421 GB) | `mainnet-1.134.1` | dead chain, never advances |
| 1 → 4,786,560 | `mainnet-archive-v0` (700 GB) | 0.x era | dead chain, never advances |


A 3.x binary **cannot** read a `thorchain-mainnet-v1` database.


### 11.3 Do NOT sync from genesis


The archive snapshot **is** the history — replaying is a slow way to rebuild what the
download already contains. Two hard facts:


- Only **1 of 92** responding active validators still serves blocks from 17,562,001
  (`173.234.17.198`). The next-deepest starts at 20,140,001. Genesis sync therefore
  depends on one node staying reachable for 9.8M blocks.
- It needs **14** height-gated image switches starting at `mainnet-3.6.1`
  (`node-launcher/thornode-stack/mainnet.yaml`).


Verify the retention claim yourself any time:


```powershell
curl.exe -s http://173.234.17.198:27147/status | ConvertFrom-Json | ForEach-Object result | ForEach-Object sync_info | Select-Object earliest_block_height
```


### 11.4 Coverage and sizing


| | Value |
|---|---|
| Snapshot height | 25,390,400 |
| History in the snapshot | **7,828,399 blocks** (17,562,001 → 25,390,400) |
| Replay to reach tip | ~1.98M blocks |
| Total coverage once caught up | **9,812,365 blocks** — all of `thorchain-1` |
| Tarball | 2,519 GB |
| Extracted (est. 2.08× — measured on the pruned snapshot) | ~5,240 GB |
| **Peak during extraction** | **~7,759 GB** — tarball and extract coexist |
| Steady state after catch-up | ~6,500 GB |
| Growth | ~290 GB/month |


**PVC must be 16Ti (P70).** An 8 TiB P60 is 8,796 GB — the extraction peak hits 88% of
it, and overflowing wastes a 5-hour download. Azure has no tier between P60 and P70.


### 11.5 YAML changes


Five lines. Copy `manifests-v2/` to a separate folder and edit there — do not modify the
pruned manifests in place.


```yaml
# 02-pvc.yaml
  storage: 400Gi                                   →  storage: 16Ti

# 04-statefulset.yaml — init-snapshot env
  - { name: SNAP_TYPE, value: "mainnet-pruned" }   →  value: "mainnet-archive"

# 04-statefulset.yaml — BOTH init-keys and thornode containers
  image: ...thornode:mainnet-3.19.3                →  ...thornode:mainnet-3.16.4

# 04-statefulset.yaml — thornode env
  - { name: THOR_COSMOS_PRUNING, value: "default" } →  value: "nothing"
```


`mainnet-3.16.4`, **not** `3.19.3` — the snapshot sits at 25,390,400, and 3.16.4 owned
heights 25,215,740 → 25,959,000. Deploying 3.19.3 here produces the app-hash halt from
Issue 1.


Also raise the resources (the constrained profile in section 5, Issue 5 exists only
because the shared cluster had nothing spare — archive needs the real thing):


```yaml
  resources:
    requests: { cpu: "4", memory: "8Gi" }
    limits:   { memory: "16Gi" }        # no CPU limit
  startupProbe:
    failureThreshold: 960               # opening a multi-TB LevelDB store far exceeds 60 min
```


### 11.6 One required script change


`chmod -R a+rwX` took 3-5 minutes on 82 GB. Archive is ~63× that data, so the same pass
runs for **hours**. Set the modes during extraction instead.


In `03-configmap-scripts.yaml`, `init-snapshot.sh`, replace the extract line:


```sh
# before
gzip -dc "$TARBALL" | tar -xf - -C "$HOME_DIR"

# after — files land 0666 / dirs 0777 directly
umask 000
gzip -dc "$TARBALL" | tar --no-same-owner --no-same-permissions -xf - -C "$HOME_DIR"
```


Then delete the `chmod -R a+rwX "$HOME_DIR"` line further down.


### 11.7 Image switches during replay


The node restores at 25,390,400 and must change binary at three exact heights. **Miss one
and it halts on app-hash divergence.**


| At height | Switch to |
|---|---|
| 25,959,000 | `mainnet-3.17.0` |
| 26,143,000 | `mainnet-3.18.2` |
| 26,518,000 | `mainnet-3.19.3` |


Watch the height, then swap **both** containers:


```powershell
kubectl logs -n thorchain-archive thornode-0 -c thornode --tail=300 | Select-String "committed state" | Select-Object -Last 1

kubectl set image sts/thornode -n thorchain-archive `
  thornode=registry.gitlab.com/thorchain/thornode:mainnet-3.17.0 `
  init-keys=registry.gitlab.com/thorchain/thornode:mainnet-3.17.0
kubectl rollout status sts/thornode -n thorchain-archive
```


After 26,518,000 it stays on `3.19.3` and follows the normal upgrade procedure
(section 9).


### 11.8 Verify BEFORE trusting any audit output


The moment the node starts, spot-check an early height. If this fails, the snapshot is
not full-history and the entire plan changes:


```powershell
kubectl exec -n thorchain-archive thornode-0 -c thornode -- `
  curl -s --max-time 30 -H "x-cosmos-block-height: 17600000" `
  http://localhost:1317/cosmos/bank/v1beta1/balances/thor1dheycdevq39qlkxs2a6wuuzyn4aqxhve4qxtxt
```


- Balance list returned → archive history is intact, proceed.
- `no commit info found` → **stop**. It is not a full archive. Do not run audits against it.


Confirm pruning actually took effect:


```powershell
kubectl exec -n thorchain-archive thornode-0 -c thornode -- sh -c "grep -E '^(pruning|min-retain-blocks)' /data/.thornode/config/app.toml"
# expect: pruning = "nothing"
```


### 11.9 Expected timeline


| Phase | Duration |
|---|---|
| Snapshot download (2.5 TB @ ~147 MiB/s measured) | ~5 hours |
| Extract (~5.2 TB) | 2-4 hours |
| Replay 1.98M blocks + 3 image swaps | days — depends entirely on CPU and disk IOPS |


Archive replay writes every IAVL version rather than discarding old ones, so it is
materially heavier than the pruned sync. Do not size it from pruned experience.


### 11.10 Prerequisites checklist


Confirm all five **before** starting — three of them cannot be fixed once you have begun:


- [ ] 16Ti Premium_LRS quota available in the subscription
- [ ] Node pool with ≥4 dedicated vCPU and ≥16 GiB (a shared, fully-committed pool will not converge)
- [ ] Free IPs in the AKS subnet (the `/27` here is exhausted — see Issue 9)
- [ ] Audit height range confirmed: does it reach before 2024-09-04? If yes you need `mainnet-archive-v1` too
- [ ] Owner assigned for the three image switches — they are manual and time-sensitive


---


## Node Endpoint Summary


| Endpoint | Type | Address | Stable? |
|---|---|---|---|
| **RPC** | **NodePort** | **`http://10.202.17.198:30399`** | port yes, node IP no |
| **REST API** | **NodePort** | **`http://10.202.17.198:32210`** | port yes, node IP no |
| **gRPC** | **NodePort** | **`10.202.17.198:30609`** | port yes, node IP no |
| RPC / REST / gRPC | Internal LB | `<pending>` — `SubnetIsFull`, Issue 9 | — |
| P2P | Pod-only (outbound) | `27146` — no Service; a full node needs only outbound | — |
| Metrics | ClusterIP | `http://thornode.thorchain.svc.cluster.local:26660/metrics` | ✅ |
| RPC / REST / gRPC (in-cluster) | ClusterIP | `http://thornode.thorchain.svc.cluster.local:27147` / `:1317` / `:9090` | ✅ |
| Headless DNS | Cluster-internal | `thornode-0.thornode-headless.thorchain.svc.cluster.local` | ✅ — StatefulSet plumbing, do not hand out |


**NodePorts are pinned** in `05-service.yaml` (`30399` / `32210` / `30609`), so a Service
recreate cannot reassign them. **Node IPs are not** — any node in the cluster serves the
same NodePorts, so distribute the whole list from `kubectl get nodes -o wide` rather than
a single address.


## Reference


- <https://docs.thorchain.org/thornodes/fullnode/thornode-docker>
- <https://docs.thorchain.org/thornodes/fullnode/thornode-kubernetes>
- <https://gitlab.com/thorchain/thornode/-/blob/develop/config/default.yaml> — every key is a `THOR_*` env var
- <https://gitlab.com/thorchain/thornode/-/blob/develop/build/scripts/fullnode.sh>
- <https://gitlab.com/thorchain/devops/node-launcher/-/blob/master/thornode/values.yaml> — upstream resource sizing
- <https://snapshot-api.liquify.com/files/by-network/thorchain> — snapshot catalogue


---


Wiki created: 2026-08-10 | Cluster: `cuscoiniadevaks` / `azrg-cus-coinia-aks-dev` | Maintained by: DevOps team
