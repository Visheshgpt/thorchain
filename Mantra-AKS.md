# Mantra Chain Node — AKS Setup Wiki

  

**Namespace:** `mantra` | **Chain:** `mantra-1` | **Sync:** Genesis (block 1 → tip) via Cosmovisor  

**Internal LB — P2P:** `10.202.17.214:26656` | **Internal LB — RPC/API:** `10.202.17.215`  

**Status:**  Running — genesis sync in progress (2026-07-22)

  

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

9. [Cosmovisor Upgrade Path](#9-cosmovisor-upgrade-path)

10. [Monitoring & Ops Commands](#10-monitoring--ops-commands)

  

---

  

## 1. Architecture Overview

  

### Why Not Docker Hub / Public Image?

  

Unlike other nodes (Archway, VeChain, etc.) which use official public images, Mantra Chain requires a **custom setup** because:

- The chain needs **Cosmovisor** to handle 11 automatic binary upgrades from genesis (v1.0.0) to v8.2.0

- There is no official public Docker image that bundles all upgrade binaries

  

### Solution: Init Container Pattern (No ACR Required)

  

Instead of building a custom Docker image (which would require ACR), **three sequential init containers** handle all setup at pod startup using only public images:

  

```

Pod: mantrachaind-genesis-0

│

├── Init 1 — install-cosmovisor (golang:1.23.4-alpine)

│   └── Downloads pre-built cosmovisor v1.7.1 binary from GitHub → saves to PVC

│

├── Init 2 — download-binaries (golang:1.23.4-alpine)

│   └── Downloads all 11 mantrachaind binaries from GitHub releases with SHA256 verification

│

├── Init 3 — init-node (golang:1.23.4-alpine)

│   ├── mantrachaind init mantra-archive --chain-id mantra-1

│   ├── Downloads mainnet genesis.json from MANTRA-Chain/net GitHub

│   └── Configures app.toml + config.toml (pruning=nothing, seeds, peers, ports)

│

└── Main — mantrachaind-genesis (alpine:3.19)

    └── exec cosmovisor run start → auto-switches binaries at upgrade heights

```

  

**All images are public.**

  

### Key Design Decisions

  
| Decision | Reason |
| --- | --- |
| `golang:1.23.4-alpine` for all init containers | Already includes `wget` + `ca-certificates` — no `apk add` needed (requires root, blocked by Gatekeeper) |
| Download pre-built cosmovisor instead of `go install` | `go install` compiles 100+ packages → OOMKills on Standard_D2s_v3 (5.79Gi allocatable) |
| `runAsUser: 1025` on all containers | Gatekeeper/Azure Policy enforces no-root containers |
| Only `fsGroup: 1025` at pod level | Per-container securityContext needed because init containers have different UID requirements |
| `pruning = "nothing"` | Archive node — full historical state required |
| PVC starts at `200Gi` | pvc-watcher auto-expands; genesis sync grows slowly at first |

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

  

Mantra requires `250m CPU` + `2Gi memory` free on at least one node.

  

---

  

## 3. Manifest Files

  | File | Purpose |
| --- | --- |
| `00-namespace.yaml` | Namespace `mantra` |
| `01-storageclass.yaml` | `mantra-storage` — StandardSSD_LRS, `reclaimPolicy: Retain` |
| `02-pvc.yaml` | `mantrachaind-genesis-data-pvc` — 200Gi initial, auto-expands to 3000Gi |
| `03-statefulset.yaml` | StatefulSet with 3 init containers + main cosmovisor container |
| `04-service.yaml` | Headless + P2P Internal LB + RPC/API/EVM Internal LB |
  

### Storage Class Note

`reclaimPolicy: Retain` is **mandatory** — blockchain data is irreplaceable. Never use `Delete`.

  

---

  

## 4. Step-by-Step Deployment

  

### Step 1 — Copy manifests to jumpbox

  

```powershell

# On jumpbox

mkdir C:\Users\<user>\mantranode

# Copy all 5 yaml files to this directory

```

  

### Step 2 — Apply in order

  

```powershell

cd C:\Users\<user>\mantranode

  

kubectl apply -f 00-namespace.yaml

kubectl apply -f 01-storageclass.yaml

kubectl apply -f 02-pvc.yaml

kubectl apply -f 03-statefulset.yaml

kubectl apply -f 04-service.yaml

```

  

### Step 3 — Watch pod startup

  

```powershell

# Pod phases: Pending → Init:0/3 → Init:1/3 → Init:2/3 → Init:3/3 → Running

kubectl get pods -n mantra -w

```


### Step 4 — Monitor init container logs

  

```powershell

# Init 1 — cosmovisor download

kubectl logs -n mantra mantrachaind-genesis-0 -c install-cosmovisor

  

# Init 2 — binary downloads (watch for 11 ✓ verified lines)

kubectl logs -n mantra mantrachaind-genesis-0 -c download-binaries -f

  

# Init 3 — node init + config

kubectl logs -n mantra mantrachaind-genesis-0 -c init-node

  

# Main container — cosmovisor sync

kubectl logs -n mantra mantrachaind-genesis-0 -c mantrachaind-genesis -f

```

  

### Step 5 — Verify services have Internal LB IPs

  

```powershell

kubectl get svc -n mantra

```

  

Expected output:

```

mantrachaind-genesis          ClusterIP  None           <none>          (all ports)

mantrachaind-genesis-p2p      LoadBalancer  ...         10.202.x.x     26656

mantrachaind-genesis-rpc-lb   LoadBalancer  ...         10.202.x.x     26657,1317,9090,9091,8545,8546,26660

```

  

### Step 6 — Add to pvc-watcher

  

```powershell

kubectl edit configmap pvc-watcher-script -n pvc-watcher

```

  

Add to `NODE_CONFIG`:

```

"mantra|mantrachaind-genesis-0|/home/nonroot/.mantrachain|mantrachaind-genesis-data-pvc|80|200|3000"

```

  

---

  

## 5. Issues Encountered & Resolutions

  

### Issue 1 — `Pending`: Insufficient CPU + Memory

  

**Error:**

```

0/3 nodes are available: 3 Insufficient cpu, 3 Insufficient memory.

cluster-autoscaler: pod didn't trigger scale-up: 1 max node group size reached

```

  

**Root Cause:** Initial resource requests of `2 CPU + 16Gi` exceeded available capacity across all 3 `Standard_D2s_v3` nodes (each has only 1.9 vCPU / 5.79Gi allocatable).

  

**Node capacity analysis at time of deployment:**

| Node | Free CPU | Free Memory | Schedulable? |
| --- | --- | --- | --- |
| vmss000000 | ~353m | ~3.1Gi |  Only option |
| vmss000005 | ~350m | ~0.9Gi |  Memory full |
| vmss000008 | ~13m | ~2.7Gi |  CPU full |

  

**Resolution:** Reduced resource requests to fit vmss000000:

```yaml

resources:

  requests:

    cpu: "250m"    # was: 2

    memory: "2Gi"  # was: 16Gi

  limits:

    cpu: "2"

    memory: "5Gi"  # was: 32Gi (unrealistic on 5.79Gi node)

```

  

---

  

### Issue 2 — `CreateContainerConfigError`: runAsNonRoot conflict

  

**Error:** Pod moved to `Init:0/3` but `install-cosmovisor` showed `CreateContainerConfigError`

  

**Root Cause:** Pod-level `securityContext.runAsNonRoot: true` conflicted with `install-cosmovisor` needing `runAsUser: 0` (root for `go install`).

  

**Resolution:** Moved `runAsNonRoot`, `runAsUser`, `runAsGroup` to **container level**, keeping only `fsGroup: 1025` at pod level. Added `runAsNonRoot: false` to the init container explicitly.

  

---

  

### Issue 3 — `CreateContainerConfigError` persisted: Gatekeeper blocks root

  

**Error:** Even with `runAsNonRoot: false` at container level, `CreateContainerConfigError` persisted.

  

**Root Cause:** The cluster runs **Gatekeeper (OPA) + Azure Policy** with a cluster-wide no-root enforcement. Any container with `runAsUser: 0` is rejected by the admission webhook regardless of pod-level settings.

  

**Evidence:**

```

gatekeeper-system/gatekeeper-controller-57b6bd4c99   Running

kube-system/azure-policy-76cf6778c8                  Running

kube-system/azure-policy-webhook-f9895f4fd           Running

```

  

**Resolution:** Changed `install-cosmovisor` to run as `runAsUser: 1025` (nonroot). Added `GOPATH=/tmp/go`, `GOCACHE=/tmp/gocache`, `HOME=/tmp` so `go install` could write without root access.

  

---

  

### Issue 4 — `Init:CrashLoopBackOff` (10 restarts): go install OOMKills

  

**Error:** Init container 1 in `CrashLoopBackOff` after 3h27m with 10 restarts.

  

**Root Cause:** `go install cosmossdk.io/tools/cosmovisor/cmd/cosmovisor@v1.7.1` compiles 100+ Go packages including cosmos-sdk and cometbft. On a 250m CPU allocation, this takes **60-90+ minutes** and the Go linker eventually exceeds the 5Gi memory limit → OOMKill (Exit Code 137).

  

The compilation was downloading ~150 packages successfully but failing during the memory-intensive link phase.

  

**Resolution:** **Replace compilation with a pre-built binary download:**

```sh

# Before (takes 60-90 min, OOMKills):

go install cosmossdk.io/tools/cosmovisor/cmd/cosmovisor@v1.7.1

  

# After (takes ~5 seconds):

wget -qO /tmp/cosmovisor.tar.gz \

  "https://github.com/cosmos/cosmos-sdk/releases/download/tools%2Fcosmovisor%2Fv1.7.1/cosmovisor_1.7.1_linux_amd64.tar.gz"

tar -xzf /tmp/cosmovisor.tar.gz -C /tmp/

cp /tmp/cosmovisor "$BIN"

```

  

---

  

### Issue 5 — `apk add` blocked for download-binaries and init-node

  

**Discovered during:** Redesign after Issue 4 — reviewing all init containers.

  

**Root Cause:** `download-binaries` and `init-node` both used `alpine:3.19` and ran `apk add wget ca-certificates` / `apk add curl jq`. Since `apk add` requires root (writes to `/usr/bin/`, `/lib/apk/db/`) and Gatekeeper blocks root — these containers would also have crashed once reached.

  

**Resolution:** Switch all three init containers to `golang:1.23.4-alpine` which **already includes**:

- ✅ `wget` (from busybox)

- ✅ `ca-certificates` (added by golang image Dockerfile)

- ✅ `sha256sum` (from busybox)

- ✅ `sed`, `bash`, `tar`, `grep` (from busybox/alpine base)

  

Replaced `curl -fsSL` with `wget -qO` in init-node. Replaced `jq` chain_id display with `grep -o`.

  

**Result:** Zero `apk add` calls anywhere. All init containers run as UID 1025, fully Gatekeeper-compliant.

  

---

  

### Summary of All Fixes

 | # | Issue | Root Cause | Fix |
| --- | --- | --- | --- |
| 1 | `Pending` | CPU/mem requests too high for cluster | Reduce to `250m` / `2Gi` |
| 2 | `CreateContainerConfigError` | Pod-level `runAsNonRoot:true` vs init container `runAsUser:0` | Per-container securityContext |
| 3 | `CreateContainerConfigError` (persists) | Gatekeeper/Azure Policy blocks root globally | All containers `runAsUser:1025` |
| 4 | `Init:CrashLoopBackOff` | `go install` OOMKills on 5.79Gi nodes (60-90 min compile) | Download pre-built cosmovisor binary |
| 5 | Latent `apk add` crash | `apk add` requires root, Gatekeeper blocks | Switch to `golang:1.23.4-alpine` (has wget+ca-certs) |

---

  

## 6. Verification — curl Commands

  

> Replace `10.202.17.215` with the actual Internal LB IP from `kubectl get svc -n mantra`.  

> Use `kubectl port-forward -n mantra svc/mantrachaind-genesis 26657:26657` for local testing.

  

### CometBFT RPC (port 26657)

  

```bash

# Node status + sync info

curl -s http://10.202.17.215:26657/status | python3 -m json.tool

  

# Quick sync check — height and catching_up

curl -s http://10.202.17.215:26657/status | python3 -c \

  "import sys,json; d=json.load(sys.stdin)['result']['sync_info']; \

   print(f\"Height: {d['latest_block_height']}\nCatching up: {d['catching_up']}\")"

  

# Health check (returns 200 OK if healthy)

curl -s http://10.202.17.215:26657/health

  

# Latest block

curl -s http://10.202.17.215:26657/block | python3 -m json.tool

  

# Block by height

curl -s http://10.202.17.215:26657/block?height=890 | python3 -m json.tool

  

# Block results (transactions, events) at height

curl -s http://10.202.17.215:26657/block_results?height=890 | python3 -m json.tool

  

# Peer count and info

curl -s http://10.202.17.215:26657/net_info | python3 -c \

  "import sys,json; d=json.load(sys.stdin)['result']; \

   print(f\"Peers: {d['n_peers']}\")"

  

# Full peer details

curl -s http://10.202.17.215:26657/net_info | python3 -m json.tool

  

# Current validator set

curl -s http://10.202.17.215:26657/validators | python3 -m json.tool

  

# Transaction by hash

curl -s "http://10.202.17.215:26657/tx?hash=0x<TX_HASH>" | python3 -m json.tool

  

# Search transactions by event

curl -s "http://10.202.17.215:26657/tx_search?query=tx.height=890" | python3 -m json.tool

  

# ABCI application info

curl -s http://10.202.17.215:26657/abci_info | python3 -m json.tool

  

# Consensus state

curl -s http://10.202.17.215:26657/consensus_state | python3 -m json.tool

  

# Dump consensus state (detailed)

curl -s http://10.202.17.215:26657/dump_consensus_state | python3 -m json.tool

  

# Current upgrade info

curl -s http://10.202.17.215:26657/abci_query?path="\"store/upgrade/key\"" | python3 -m json.tool

```

  

### Cosmos REST API — LCD (port 1317)

  

```bash

# Latest block

curl -s http://10.202.17.215:1317/cosmos/base/tendermint/v1beta1/blocks/latest | python3 -m json.tool

  

# Block by height

curl -s http://10.202.17.215:1317/cosmos/base/tendermint/v1beta1/blocks/890 | python3 -m json.tool

  

# Node info

curl -s http://10.202.17.215:1317/cosmos/base/tendermint/v1beta1/node_info | python3 -m json.tool

  

# Syncing status

curl -s http://10.202.17.215:1317/cosmos/base/tendermint/v1beta1/syncing | python3 -m json.tool

  

# Validators at latest height

curl -s http://10.202.17.215:1317/cosmos/base/tendermint/v1beta1/validatorsets/latest | python3 -m json.tool

  

# Account balance

curl -s "http://10.202.17.215:1317/cosmos/bank/v1beta1/balances/<WALLET_ADDRESS>" | python3 -m json.tool

  

# Transaction by hash (Cosmos REST)

curl -s "http://10.202.17.215:1317/cosmos/tx/v1beta1/txs/<TX_HASH>" | python3 -m json.tool

  

# Search transactions by height

curl -s "http://10.202.17.215:1317/cosmos/tx/v1beta1/txs?events=tx.height=890" | python3 -m json.tool

  

# Chain parameters

curl -s http://10.202.17.215:1317/cosmos/staking/v1beta1/params | python3 -m json.tool

  

# Bank total supply

curl -s http://10.202.17.215:1317/cosmos/bank/v1beta1/supply | python3 -m json.tool

  

# Governance proposals

curl -s "http://10.202.17.215:1317/cosmos/gov/v1beta1/proposals?proposal_status=2" | python3 -m json.tool

```

  

### EVM JSON-RPC (port 8545) — Active from block 14,488,888 (v8.0.0)

  

> ⚠️ These endpoints are NOT available until the node syncs past block 14,488,888

  

```bash

# EVM chain ID (should return 5888 for mantra-1 mainnet)

curl -s -X POST http://10.202.17.215:8545 \

  -H "Content-Type: application/json" \

  -d '{"jsonrpc":"2.0","method":"eth_chainId","params":[],"id":1}' | python3 -m json.tool

  

# Latest EVM block number

curl -s -X POST http://10.202.17.215:8545 \

  -H "Content-Type: application/json" \

  -d '{"jsonrpc":"2.0","method":"eth_blockNumber","params":[],"id":1}' | python3 -m json.tool

  

# Get EVM block by number

curl -s -X POST http://10.202.17.215:8545 \

  -H "Content-Type: application/json" \

  -d '{"jsonrpc":"2.0","method":"eth_getBlockByNumber","params":["latest",true],"id":1}' | python3 -m json.tool

  

# EVM transaction by hash

curl -s -X POST http://10.202.17.215:8545 \

  -H "Content-Type: application/json" \

  -d '{"jsonrpc":"2.0","method":"eth_getTransactionByHash","params":["0x<EVM_TX_HASH>"],"id":1}' | python3 -m json.tool

  

# EVM transaction receipt

curl -s -X POST http://10.202.17.215:8545 \

  -H "Content-Type: application/json" \

  -d '{"jsonrpc":"2.0","method":"eth_getTransactionReceipt","params":["0x<EVM_TX_HASH>"],"id":1}' | python3 -m json.tool

  

# EVM account balance (in wei)

curl -s -X POST http://10.202.17.215:8545 \

  -H "Content-Type: application/json" \

  -d '{"jsonrpc":"2.0","method":"eth_getBalance","params":["0x<EVM_ADDRESS>","latest"],"id":1}' | python3 -m json.tool

```

  

### PowerShell equivalents (for jumpbox)

  

```powershell

# Sync status

Invoke-RestMethod http://localhost:26657/status | Select-Object -ExpandProperty result | Select-Object -ExpandProperty sync_info

  

# Block height only

(Invoke-RestMethod http://localhost:26657/status).result.sync_info.latest_block_height

  

# Peer count

(Invoke-RestMethod http://localhost:26657/net_info).result.n_peers

  

# Latest block

Invoke-RestMethod http://localhost:26657/block | Select-Object -ExpandProperty result

```

  

### Sync progress monitoring script (PowerShell)

  

```powershell

$TARGET = 15700000

while ($true) {

    $height = (Invoke-RestMethod http://localhost:26657/status).result.sync_info.latest_block_height

    $pct = [math]::Round(($height / $TARGET) * 100, 2)

    $peers = (Invoke-RestMethod http://localhost:26657/net_info).result.n_peers

    Write-Host "$(Get-Date -Format 'HH:mm:ss') | Height: $height / $TARGET ($pct%) | Peers: $peers"

    Start-Sleep -Seconds 30

}

```

  

---

  

## 7. Postman Collection

  

Import the file `mantra-postman-collection.json` (in this folder) directly into Postman.

  

**Collection Variables to set:**

| Variable | Value |
| --- | --- |
| `rpc_url` | `http://10.202.17.215:26657` |
| `lcd_url` | `http://10.202.17.215:1317` |
| `evm_url` | `http://10.202.17.215:8545` |
| `block_height` | `890` |
| `tx_hash` | _(paste a tx hash)_ |
| `wallet_address` | _(paste a wallet address)_ |
  

> For local testing via port-forward, set `rpc_url` to `http://localhost:26657` and `lcd_url` to `http://localhost:1317`

  

---

  

## 8. pvc-watcher Configuration

  

The pvc-watcher CronJob automatically expands the disk when usage exceeds 80%.

  

**Active entry in `pvc-watcher-script` ConfigMap:**

```

"mantra|mantrachaind-genesis-0|/home/nonroot/.mantrachain|mantrachaind-genesis-data-pvc|80|200|3000"

```

  
| Field | Value | Meaning |
| --- | --- | --- |
| NAMESPACE | `mantra` | Kubernetes namespace |
| POD | `mantrachaind-genesis-0` | Pod to exec df into |
| MOUNT_PATH | `/home/nonroot/.mantrachain` | Path to check |
| PVC_NAME | `mantrachaind-genesis-data-pvc` | PVC to expand |
| THRESHOLD | `80` | Expand when 80% full |
| INCREMENT_GI | `200` | Add 200Gi each time |
| MAX_SIZE_GI | `3000` | Cap at 3000Gi (3TB) |

**Check current disk usage:**

```powershell

kubectl exec -n mantra mantrachaind-genesis-0 -- df -h /home/nonroot/.mantrachain

```

  

**Manual expansion (if needed):**

```powershell

kubectl patch pvc mantrachaind-genesis-data-pvc -n mantra \

  -p '{"spec":{"resources":{"requests":{"storage":"400Gi"}}}}'

```

  

---

  

## 9. Cosmovisor Upgrade Path

  

Cosmovisor automatically switches the active binary at each governance upgrade height. **No manual intervention required.**

  
| Block Height | Upgrade Name | Binary Version | Notes |
| --- | --- | --- | --- |
| 1 | genesis | v1.0.0 | Starting point |
| 3,103,100 | v2 | v2.0.0 |  |
| 3,833,000 | v3 | v3.0.0 |  |
| 4,428,500 | v4 | v4.0.0 |  |
| 8,618,888 | v5.0 | v5.0.0 |  |
| 9,664,888 | v6.0.0 | v6.0.0 |  |
| 9,898,888 | v6.1.0 | v6.1.0 |  |
| 13,000,000 | v7.0.0 | v7.0.0 |  |
| **14,488,888** | **v8.0.0** | **v8.0.0** | EVM JSON-RPC activates |
| 14,570,000 | v8.1.1 | v8.1.1 |  |
| 15,559,888 | v8.2.0 | v8.2.0 | Current tip binary |
  

**Monitor upgrade events:**

```powershell

kubectl logs -n mantra mantrachaind-genesis-0 -c mantrachaind-genesis -f 2>&1 | Select-String -Pattern "upgrade|switching|UPGRADE|panic"

```

  

**Check current active binary:**

```powershell

kubectl exec -n mantra mantrachaind-genesis-0 -- sh -c \

  'cat /home/nonroot/.mantrachain/data/upgrade-info.json 2>/dev/null || echo "genesis"'

  

kubectl exec -n mantra mantrachaind-genesis-0 -- sh -c \

  'ls -la /home/nonroot/.mantrachain/cosmovisor/current'

```

  

---

  

## 10. Monitoring & Ops Commands

  

### Daily health check

  

```powershell

Write-Host "=== Mantra Chain Health ==="

$status = Invoke-RestMethod http://localhost:26657/status

Write-Host "Height:      " $status.result.sync_info.latest_block_height

Write-Host "Catching Up: " $status.result.sync_info.catching_up

Write-Host "Chain ID:    " $status.result.node_info.network

Write-Host "Peers:       " (Invoke-RestMethod http://localhost:26657/net_info).result.n_peers

```

  

### Check pod + PVC + services

  

```powershell

kubectl get pods,pvc,svc -n mantra

```

  

### Restart node (e.g. after config change)

  

```powershell

kubectl delete pod mantrachaind-genesis-0 -n mantra

# StatefulSet recreates it automatically

# Init containers skip via sentinel files (idempotent)

```

  

### View full logs

  

```powershell

# Live tail

kubectl logs -n mantra mantrachaind-genesis-0 -c mantrachaind-genesis -f

  

# Last 100 lines

kubectl logs -n mantra mantrachaind-genesis-0 -c mantrachaind-genesis --tail=100

```

  

### Force pvc-watcher to run immediately

  

```powershell

kubectl create job pvc-watcher-manual --from=cronjob/pvc-watcher -n pvc-watcher

kubectl logs -n pvc-watcher -l job-name=pvc-watcher-manual -f

```

  

### Teardown (caution — data loss if PVC deleted)

  

```powershell

kubectl delete -f 04-service.yaml

kubectl delete -f 03-statefulset.yaml

# ⚠ STOP HERE if you want to keep the data (PVC stays bound in Retain policy)

# Only delete PVC if you also plan to delete the Azure disk manually in portal

kubectl delete -f 02-pvc.yaml

kubectl delete -f 01-storageclass.yaml

kubectl delete -f 00-namespace.yaml

```

  

---

  

## Node Endpoint Summary

  
| Endpoint | Internal LB | Port | Status |
| --- | --- | --- | --- |
| CometBFT P2P | `10.202.17.214` | `26656` |  Active |
| CometBFT RPC | `10.202.17.215` | `26657` |  Active |
| Cosmos REST API | `10.202.17.215` | `1317` |  Active |
| Cosmos gRPC | `10.202.17.215` | `9090` |  Active |
| gRPC-Web | `10.202.17.215` | `9091` |  Active |
| EVM JSON-RPC HTTP | `10.202.17.215` | `8545` |  Active at block 14,488,888 |
| EVM WebSocket | `10.202.17.215` | `8546` |  Active at block 14,488,888 |
| Prometheus Metrics | `10.202.17.215` | `26660` |  Active |

---

 For Postman Collection refer below Link

https://dev.azure.com/symphonyvsts/GATC/_git/BlockchainAgent?path=/blockchains/mantra/aks/mantra-postman-collection.json&version=GBdevelop


