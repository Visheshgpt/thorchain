# Fetch Chain Node — AKS Setup Wiki


**Namespace:** `fetch` | **Chain:** `fetchhub-4` | **Sync:** Snapshot sync (Polkachu pruned snapshot) via `fetchd`  
**Internal LB — P2P:** `10.202.17.220:26656` | **Internal LB — RPC/API:** `10.202.17.221`  
**Status:** ✅ Running — syncing from snapshot (2026-08-03)


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


---


## 1. Architecture Overview


### Public Image — No ACR Required


`fetchai/fetchd:0.15.0` is available on public Docker Hub — no custom image build or ACR needed.


The only custom setup needed is around **first-boot initialization**, because the node must:
- Initialize `/data/.fetchd`
- Download the mainnet genesis file
- Apply Kubernetes-specific P2P/RPC config fixes
- Restore a **Polkachu pruned snapshot** before starting normal sync


### Solution: Init Container Pattern (No ACR Required)


Instead of building a custom Docker image, **three sequential init containers** handle all setup at pod startup using only public images:


```
Pod: fetchd-0
│
├── Init 1 — init-fetchd (fetchai/fetchd:0.15.0)
│   └── Runs fetchd init --chain-id fetchhub-4 --home /data/.fetchd → saves config skeleton to PVC
│
├── Init 2 — init-config (golang:1.23.4-alpine)
│   ├── Downloads genesis_migrated_5300200.json
│   └── Configures app.toml + config.toml (seeds, peers, P2P fixes, RPC bind, external address)
│
├── Init 3 — init-snapshot (golang:1.23.4-alpine)
│   ├── Builds inline Go lz4 decompressor using pierrec/lz4/v4
│   └── Downloads Polkachu snapshot fetch_28560367.tar.lz4 → extracts to /data/.fetchd/
│
└── Main — fetchd (fetchai/fetchd:0.15.0)
    └── exec fetchd start --home=/data/.fetchd
```


**All images are public. No ACR required.**


### Key Design Decisions


| Decision | Reason |
|---|---|
| `fetchai/fetchd:0.15.0` for init + main binary | Official public image already includes `fetchd` |
| `golang:1.23.4-alpine` for config/snapshot init containers | Includes Go + basic tooling without requiring `apk add` |
| Inline Go lz4 decompressor instead of `apk add lz4` | `apk add` requires root, blocked by Gatekeeper/Azure Policy |
| `runAsUser: 1025` on all containers | Gatekeeper/Azure Policy enforces no-root containers |
| Only `fsGroup: 1025` at pod level | Per-container securityContext needed because all containers must remain non-root |
| Snapshot sync from Polkachu | Much faster than syncing from genesis for Fetch |
| PVC starts at `200Gi` | pvc-watcher auto-expands; pruned snapshot starts small but grows over time |


---


## 2. Prerequisites


### Cluster Requirements


- AKS cluster with Azure Disk CSI driver (`disk.csi.azure.com`)
- **Gatekeeper/Azure Policy** enforcing no-root containers (observed in this cluster)
- Node pool: `Standard_D2s_v3` (2 vCPU, 8Gi RAM) — schedulable node needs `≥250m CPU` and `≥2Gi` free


### Verify schedulable capacity before deploying


```powershell
kubectl describe nodes | Select-String -Pattern "Allocated resources" -Context 0,6
```


Fetch requires `250m CPU` + `2Gi memory` free on at least one node.


---


## 3. Manifest Files


| File | Purpose |
|---|---|
| `00-namespace.yaml` | Namespace `fetch` |
| `01-storageclass.yaml` | `fetch-storage` — StandardSSD_LRS, `reclaimPolicy: Retain` |
| `02-pvc.yaml` | `fetchd-data` — 200Gi initial, auto-expands to 1000Gi |
| `03-statefulset.yaml` | StatefulSet with 3 init containers + main `fetchd` container |
| `04-service.yaml` | Headless + P2P Internal LB + RPC/API Internal LB |


### Storage Class Note
`reclaimPolicy: Retain` is **mandatory** — blockchain data is irreplaceable. Never use `Delete`.


---


## 4. Step-by-Step Deployment


### Step 1 — Copy manifests to jumpbox


```powershell
# On jumpbox
mkdir C:\Users\<user>\fetchnode
# Copy all 5 yaml files from D:\nodesetup\Fetch\AKS setup\manifests\ to this directory
```


### Step 2 — Apply in order


Before applying, update `SNAPSHOT_URL` in `03-statefulset.yaml` `init-snapshot` to the latest Polkachu snapshot:
`https://snapshots.polkachu.com/snapshots/fetch/fetch_XXXXXXX.tar.lz4`


```powershell
cd C:\Users\<user>\fetchnode


kubectl apply -f 00-namespace.yaml
kubectl apply -f 01-storageclass.yaml
kubectl apply -f 02-pvc.yaml
kubectl apply -f 03-statefulset.yaml
kubectl apply -f 04-service.yaml
```


### Step 3 — Watch pod startup


```powershell
# Pod phases: Pending → Init:0/3 → Init:1/3 → Init:2/3 → Init:3/3 → Running
kubectl get pods -n fetch -w
```


Expected timelines:
| Phase | Duration | What's happening |
|---|---|---|
| `Init:0/3` (init-fetchd) | ~10-20 seconds | `fetchd init --chain-id fetchhub-4 --home /data/.fetchd` |
| `Init:1/3` (init-config) | ~30-60 seconds | genesis download + `app.toml` / `config.toml` updates |
| `Init:2/3` (init-snapshot) | ~10-25 minutes | build Go lz4 decompressor + download/extract snapshot |
| `Running 0/1` → `1/1` | ~1-3 minutes | `fetchd start` begins; RPC port 26657 opens |


### Step 4 — Monitor init container logs


```powershell
kubectl logs -n fetch fetchd-0 -c init-fetchd
kubectl logs -n fetch fetchd-0 -c init-config
kubectl logs -n fetch fetchd-0 -c init-snapshot -f
kubectl logs -n fetch fetchd-0 -c fetchd -f
```


### Step 5 — Verify services have Internal LB IPs


```powershell
kubectl get svc -n fetch
```


### Step 6 — Add to pvc-watcher


```powershell
kubectl edit configmap pvc-watcher-script -n pvc-watcher
```


Add to `NODE_CONFIG`:
```
"fetch|fetchd-0|/data/.fetchd|fetchd-data|80|200|1000"
```


---


## 5. Issues Encountered & Resolutions


### Issue 1 — `Init:2/3 CrashLoopBackOff`: Snapshot URL placeholder not replaced
The initial manifest still had a placeholder snapshot filename (`fetch_UPDATE_ME` / `fetch_XXXXXXX`) in `SNAPSHOT_URL`.


**Fix:** Update `SNAPSHOT_URL` in `03-statefulset.yaml` before `kubectl apply`, using the latest Polkachu snapshot filename.


### Issue 2 — `lz4` decompressor install blocked by Gatekeeper
Polkachu snapshots are `.tar.lz4`, but `apk add lz4` requires root and was blocked by cluster policy.


**Fix:** Use `golang:1.23.4-alpine` and build a tiny inline Go lz4 decompressor with `github.com/pierrec/lz4/v4` inside `init-snapshot`.


### Issue 3 — No peers connecting (`numPeers=0`)
Default `config.toml` values were not Kubernetes-friendly:
- `addr_book_strict=true` blocked private/VNet-style addresses
- RPC `laddr` was bound to `127.0.0.1:26657`
- `external_address` was not set, so the node advertised its pod IP


**Fix:** Set:
- `addr_book_strict=false`
- `allow_duplicate_ip=true`
- P2P `laddr = "tcp://0.0.0.0:26656"`
- RPC `laddr = "tcp://0.0.0.0:26657"`
- `external_address = "10.202.17.220:26656"`
- `seeds = "ade4d8bc8cbe014af6ebdf3cb7b1e9ad36f412c0@seeds.polkachu.com:15256,17693da418c15c95d629994a320e2c4f51a8069b@connect-fetchhub.fetch.ai:36456,a575c681c2861fe945f77cb3aba0357da294f1f2@connect-fetchhub.fetch.ai:36457,d7cda986c9f59ab9e05058a803c3d0300d15d8da@connect-fetchhub.fetch.ai:36458"`
- `persistent_peers = "e3d21a822e3903a96c14bfd8f8e06132f198d7b4@sentries-fetchhub.fetch.ai:36400,4be243f5d22403f6069d2ed3c4a79161216f22a0@sentries-fetchhub.fetch.ai:36401,d6faadb9e785642b355136ed278d5b5d6b2f87dd@sentries-fetchhub.fetch.ai:36402"`


### Issue 4 — `wget` not present in `fetchai/fetchd:0.15.0`
The official Fetch image does not include `wget`, `netstat`, or `ss`, so normal in-container network debugging is limited.


**Fix:** Use `golang:1.23.4-alpine` for download-heavy init work, and use `curl.exe` from Windows with `kubectl port-forward` for RPC verification.


### Issue 5 — RPC unreachable directly from dev laptops
The RPC/Internal LB IP (`10.202.17.221`) is VNet-only and not reachable from developer laptops outside the AKS network.


**Fix:** Use:
```powershell
kubectl port-forward -n fetch fetchd-0 26657:26657
curl.exe http://localhost:26657/status
```


### Issue 6 — Azure LB health probe rejections on port 26656
Logs showed repeated messages like:
`Inbound Peer rejected: auth failure: secret conn failed: EOF`


These came from Azure Internal Load Balancer health probes hitting the P2P port with plain TCP, not from real blockchain peers.


**Fix:** No action required. This is normal and harmless log noise.


### Summary of All Fixes


| # | Issue | Root Cause | Fix |
|---|---|---|---|
| 1 | `Init:2/3 CrashLoopBackOff` | Snapshot placeholder URL not replaced | Update `SNAPSHOT_URL` before apply |
| 2 | `lz4` install blocked | `apk add` requires root | Build inline Go lz4 decompressor |
| 3 | `numPeers=0` | Default P2P/RPC config not AKS-friendly | Set P2P/RPC bind + external address + peer flags |
| 4 | Missing tools in fetch image | `fetchd:0.15.0` lacks `wget`/`ss` | Use `golang` init image + `curl.exe` / port-forward |
| 5 | RPC unreachable from laptop | Internal LB is VNet-only | Use `kubectl port-forward` |
| 6 | P2P auth failure noise | Azure LB TCP health probes | Ignore as expected behavior |


---


## 6. Verification — curl Commands


> Use `kubectl port-forward -n fetch fetchd-0 26657:26657` for local testing.


### RPC — CometBFT (`26657`)


```bash
curl http://10.202.17.221:26657/status
curl http://10.202.17.221:26657/net_info
curl http://10.202.17.221:26657/health
curl "http://10.202.17.221:26657/block?height=1"
```


### REST — Cosmos API (`1317`)


```bash
curl http://10.202.17.221:1317/cosmos/base/tendermint/v1beta1/node_info
curl http://10.202.17.221:1317/cosmos/base/tendermint/v1beta1/syncing
curl http://10.202.17.221:1317/cosmos/base/tendermint/v1beta1/blocks/latest
```


### gRPC Port Check (`9090`)


```bash
curl http://10.202.17.221:9090
```


### Metrics (`26660`)


```bash
curl http://10.202.17.221:26660/metrics
```


### PowerShell equivalents


```powershell
curl.exe http://10.202.17.221:26657/status
curl.exe http://10.202.17.221:26657/net_info
curl.exe http://10.202.17.221:26657/health
curl.exe "http://10.202.17.221:26657/block?height=1"


curl.exe http://10.202.17.221:1317/cosmos/base/tendermint/v1beta1/node_info
curl.exe http://10.202.17.221:1317/cosmos/base/tendermint/v1beta1/syncing
curl.exe http://10.202.17.221:1317/cosmos/base/tendermint/v1beta1/blocks/latest


curl.exe http://10.202.17.221:26660/metrics
```


### Local port-forward test


```powershell
kubectl port-forward -n fetch fetchd-0 26657:26657
curl.exe http://localhost:26657/status
curl.exe http://localhost:26657/net_info
```


### Sync monitoring loop


```powershell
while ($true) {
  $status = curl.exe -s http://localhost:26657/status | ConvertFrom-Json
  $latest = [int64]$status.result.sync_info.latest_block_height
  $catchingUp = $status.result.sync_info.catching_up
  Write-Host "$(Get-Date -Format s) | height=$latest | catching_up=$catchingUp"
  Start-Sleep -Seconds 15
}
```


---


## 7. Postman Collection


No Postman collection yet — **to be added**.


---


## 8. pvc-watcher Configuration


Add the Fetch node entry to the pvc-watcher ConfigMap:


```powershell
kubectl edit configmap pvc-watcher-script -n pvc-watcher
```


Add:
```
"fetch|fetchd-0|/data/.fetchd|fetchd-data|80|200|1000"
```


Meaning:
- Namespace: `fetch`
- Pod: `fetchd-0`
- Data path: `/data/.fetchd`
- PVC: `fetchd-data`
- Expand when usage exceeds `80%`
- Start size: `200Gi`
- Max size: `1000Gi`


---


## 9. Upgrade Procedure


Fetch does **not** use Cosmovisor in this deployment.


Upgrade path is **manual** via image tag change:
1. Update the `fetchai/fetchd:<tag>` image in `03-statefulset.yaml`
2. Apply the updated StatefulSet
3. Let Kubernetes restart `fetchd-0` with the new binary
4. Monitor logs and confirm RPC/peer recovery


Use the existing PVC so chain data is preserved across pod restarts.


---


## 10. Monitoring & Ops Commands


### Pod / StatefulSet / PVC


```powershell
kubectl get pods -n fetch
kubectl get sts -n fetch
kubectl get pvc -n fetch
kubectl describe pod fetchd-0 -n fetch
kubectl describe pvc fetchd-data -n fetch
```


### Logs


```powershell
kubectl logs -n fetch fetchd-0 -c init-fetchd
kubectl logs -n fetch fetchd-0 -c init-config
kubectl logs -n fetch fetchd-0 -c init-snapshot
kubectl logs -n fetch fetchd-0 -c fetchd -f
```


### Services / Endpoints


```powershell
kubectl get svc -n fetch
kubectl get endpoints -n fetch
```


### Exec / Config Inspection


```powershell
kubectl exec -n fetch fetchd-0 -c fetchd -- sh -c "ls -lah /data/.fetchd"
kubectl exec -n fetch fetchd-0 -c fetchd -- sh -c "cat /data/.fetchd/config/config.toml | grep -E 'seeds|persistent_peers|addr_book_strict|allow_duplicate_ip|external_address'"
kubectl exec -n fetch fetchd-0 -c fetchd -- sh -c "cat /data/.fetchd/config/app.toml | grep 'minimum-gas-prices'"
```


### Port-forward for RPC testing


```powershell
kubectl port-forward -n fetch fetchd-0 26657:26657
```


### Rollout / Restart


```powershell
kubectl rollout restart statefulset fetchd -n fetch
kubectl rollout status statefulset fetchd -n fetch
```


## Node Endpoint Summary


| Endpoint | Type | Address |
|---|---|---|
| P2P | Internal LB | `10.202.17.220:26656` |
| RPC | Internal LB | `http://10.202.17.221:26657` |
| REST API | Internal LB | `http://10.202.17.221:1317` |
| gRPC | Internal LB | `10.202.17.221:9090` |
| Metrics | Internal LB | `http://10.202.17.221:26660/metrics` |
| Headless DNS | Cluster-internal | `fetchd-0.fetchd-headless.fetch.svc.cluster.local` |


---


Wiki created: 2026-08-03 | Cluster: symphonyvsts/GATC AKS | Maintained by: DevOps team



