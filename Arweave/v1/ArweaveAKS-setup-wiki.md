# Arweave Archival Validator — AKS Setup Wiki


**Namespace:** `arweave` | **Chain:** Arweave mainnet | **Version:** `2.9.5.1`  
**Image:** Custom-built `arweave-custom:2.9.5.1` → pushed to Azure Container Registry  
**External LB — API + P2P:** Port **1984** (combined Arweave port)  
**Status:** Manifests ready — ACR image build required before first deploy


---


## Table of Contents


1. [Architecture Overview](#1-architecture-overview)
2. [Prerequisites](#2-prerequisites)
3. [Manifest Files](#3-manifest-files)
4. [Pre-Deployment — Build & Push Docker Image](#4-pre-deployment--build--push-docker-image)
5. [Step-by-Step Deployment (Jumpbox / Windows)](#5-step-by-step-deployment-jumpbox--windows)
6. [Verification — curl Commands](#6-verification--curl-commands)
7. [Sync Monitoring](#7-sync-monitoring)
8. [Configuration Customisation](#8-configuration-customisation)
9. [Known Issues & Resolutions](#9-known-issues--resolutions)
10. [pvc-watcher Configuration](#10-pvc-watcher-configuration)
11. [Upgrade Procedure](#11-upgrade-procedure)
12. [Monitoring & Ops Commands](#12-monitoring--ops-commands)
13. [Node Endpoint Summary](#13-node-endpoint-summary)


---


## 1. Architecture Overview


### Why a Custom Image Is Required


Unlike Fetch or Zilliqa which publish official Docker Hub images, **the Arweave team does not publish a Docker image**. They release only binary tarballs on GitHub:


```
https://github.com/ArweaveTeam/arweave/releases/download/N.2.9.5.1/
  arweave-2.9.5.1.ubuntu22.x86_64.tar.gz
```


A custom Docker image must be built from these binaries, pushed to your Azure Container Registry, and referenced in the StatefulSet.


> **Jumpbox constraint:** The jumpbox is a **Windows machine** with `kubectl` installed but **Docker is not available** (`docker: command not found`). The image build must happen elsewhere — see [Section 4](#4-pre-deployment--build--push-docker-image) for all options.


### Init Container Pattern


One init container runs before the main Arweave container on first boot:


```
Pod: arweave-0
│
├── Init 1 — init-config  (busybox:1.36)
│   └── Writes /config/config.json with peers, data_dir, port, mine: false
│       Sentinel file (.arweave-initialized) prevents re-running on restarts
│
└── Main — arweave  (<ACR>.azurecr.io/arweave-custom:2.9.5.1)
    └── exec /opt/arweave/bin/arweave foreground config_file /config/config.json
```


`busybox:1.36` is a public image — only the main container requires the ACR image.


### Storage Layout


| PVC | Mount | Size | Purpose |
|---|---|---|---|
| `arweave-data` | `/data` | 500 Gi | Chain indices, wallet, block headers |
| `arweave-storage` | `/storage` | 4096 Gi (4 TiB) | Weave data / storage module partitions |
| `arweave-config` | `/config` | 32 Gi | `config.json` written by init container |


> **Full weave is ~373 TB.** The 4 TiB `arweave-storage` PVC covers approximately one storage module partition (~3.6 TB). Expand the PVC or add additional PVCs as you sync more weave data.


### Port Strategy


| Port | Direction | Service | Purpose |
|---|---|---|---|
| 1984 | Inbound (internet) | `arweave-api` (Public LB) | HTTP REST API + P2P peer gossip (combined) |
| 1984 | In-cluster | `arweave-headless` | StatefulSet stable DNS identity |


Arweave uses a **single port for everything** — unlike Cosmos chains, there is no separate P2P vs RPC port.


### Key Design Decisions


| Decision | Reason |
|---|---|
| Custom ACR image | ArweaveTeam ships binaries, not Docker images |
| `busybox:1.36` for init | Public image, writes config.json without `apk add` |
| `runAsNonRoot: false` at pod level | Arweave Erlang binary requires root (no USER in Dockerfile) |
| Public External LoadBalancer | Arweave P2P requires inbound internet reachability on 1984 |
| `reclaimPolicy: Retain` | Arweave weave data is irreplaceable — disk must survive PVC deletion |
| `terminationGracePeriodSeconds: 60` | Allows Erlang VM to flush in-flight writes and close RocksDB cleanly |


---


## 2. Prerequisites


### Jumpbox (Windows — where you run kubectl)


| Requirement | Check |
|---|---|
| `kubectl` installed | `kubectl version --client` |
| AKS credentials configured | `kubectl get nodes` |
| `curl.exe` available | Built into Windows 10/11 — always present |
| **Docker NOT required on jumpbox** | Image is built elsewhere and already in ACR |


### AKS Cluster


| Requirement | Notes |
|---|---|
| AKS version | 1.27+ recommended |
| Azure Disk CSI driver | Pre-installed on AKS 1.21+ — no action needed |
| Node pool VM SKU | `Standard_D8s_v3` or larger (8 vCPU / 32 GiB) — Arweave requests 2 CPU / 8 Gi, bursts to 8 CPU / 24 Gi |
| Public IP quota | `arweave-api` provisions one Public LB — confirm quota in AKS node RG |
| ACR attached to AKS | `az aks update --attach-acr <ACR_NAME>` — needed to pull the custom image |


### NSG Rule Required


Arweave peers need inbound access on port 1984:


```
Inbound rule: TCP 1984   Source: Any (0.0.0.0/0)   Destination: AKS node subnet
```


Add this to the NSG on the AKS node subnet **before** deploying. Without it the node will have 0 peers.


### Verify cluster capacity


```powershell
kubectl describe nodes | Select-String -Pattern "Allocated resources" -Context 0,6
```


Arweave requires **≥ 2 CPU** and **≥ 8 Gi memory** free on at least one node.


---


## 3. Manifest Files


| File | Purpose |
|---|---|
| `00-namespace.yaml` | Namespace `arweave` |
| `01-storageclass.yaml` | `arweave-storage` — Premium_LRS, `reclaimPolicy: Retain` |
| `02-pvc.yaml` | Three PVCs: `arweave-data` 500Gi + `arweave-storage` 4096Gi + `arweave-config` 32Gi |
| `03-statefulset.yaml` | 1 init container (`init-config`) + main `arweave` container |
| `04-service.yaml` | Headless + External Public LoadBalancer (port 1984) |
| `README.md` | Full deployment reference |


> **Before applying `03-statefulset.yaml`:** Replace `<ACR_NAME>.azurecr.io/arweave-custom:2.9.5.1` with your real ACR image reference.


---


## 4. Pre-Deployment — Build & Push Docker Image


> **Docker is not available on the Windows jumpbox.** Use one of the options below to build and push the image to ACR.


### Option A — Azure Cloud Shell (Recommended — zero setup)


Azure Cloud Shell is a free Linux shell in the Azure Portal with Docker and az CLI pre-installed. No local setup needed.


1. Open [https://shell.azure.com](https://shell.azure.com) → select **Bash**
2. Run:


```bash
# Set your values
ACR_NAME="<your-acr-name>"
ARWEAVE_VERSION="2.9.5.1"


# Login to ACR
az acr login --name $ACR_NAME


# Download Arweave binaries
wget https://github.com/ArweaveTeam/arweave/releases/download/N.${ARWEAVE_VERSION}/arweave-${ARWEAVE_VERSION}.ubuntu22.x86_64.tar.gz


# Extract
tar -xzf arweave-${ARWEAVE_VERSION}.ubuntu22.x86_64.tar.gz


# Organise binaries
if [ -d "arweave-${ARWEAVE_VERSION}" ]; then
  mv arweave-${ARWEAVE_VERSION} arweave-binaries
else
  mkdir -p arweave-binaries
  mv bin lib releases erts-* genesis_data arweave-binaries/ 2>/dev/null || true
  mv .erlang arweave-binaries/ 2>/dev/null || true
fi


# Write the Dockerfile
cat > Dockerfile << 'EOF'
FROM ubuntu:22.04
RUN apt-get update && apt-get install -y \
    wget curl screen libsodium-dev libssl-dev \
    && rm -rf /var/lib/apt/lists/*
WORKDIR /opt/arweave
COPY --chown=root:root ./arweave-binaries/ /opt/arweave/
RUN chmod +x /opt/arweave/bin/*
RUN mkdir -p /data /config /storage
ENV ARNODE=arweave@127.0.0.1
ENV ARCOOKIE=arweave
EXPOSE 1984
ENTRYPOINT ["/opt/arweave/bin/arweave"]
CMD ["foreground", "config_file", "/config/config.json"]
EOF


# Build and push to ACR
docker build -t arweave-custom:${ARWEAVE_VERSION} .
docker tag arweave-custom:${ARWEAVE_VERSION} ${ACR_NAME}.azurecr.io/arweave-custom:${ARWEAVE_VERSION}
docker push ${ACR_NAME}.azurecr.io/arweave-custom:${ARWEAVE_VERSION}


echo "Image pushed: ${ACR_NAME}.azurecr.io/arweave-custom:${ARWEAVE_VERSION}"
```


> **Note:** Azure Cloud Shell has a 5 GB persistent storage quota. The Arweave tar.gz is ~100 MB; the built image is ~500 MB. You may need to clean up after pushing.


---


### Option B — WSL2 on the Jumpbox (if WSL2 is installed)


If Windows Subsystem for Linux is available on the jumpbox:


```bash
# Open WSL terminal on the jumpbox, then:
sudo apt update && sudo apt install -y docker.io
sudo systemctl start docker
sudo usermod -aG docker $USER
newgrp docker


# Then follow the same commands as Option A above
```


Check if WSL2 is available:


```powershell
wsl --list --verbose
```


---


### Option C — ACR Build Tasks (Build runs in Azure, no local Docker needed)


ACR Build Tasks run the Docker build entirely in Azure — no Docker install required anywhere.


```bash
# From Azure Cloud Shell or any machine with az CLI:
ACR_NAME="<your-acr-name>"
ARWEAVE_VERSION="2.9.5.1"


# Clone or create a build context directory and Dockerfile (same Dockerfile as Option A)
# Then trigger an ACR build:
az acr build \
  --registry $ACR_NAME \
  --image arweave-custom:${ARWEAVE_VERSION} \
  --file Dockerfile \
  .
```


This streams build logs to your terminal and pushes the image directly to ACR when done.


---


### Step: Attach ACR to AKS (if not already done)


Run once from any machine with `az` CLI (Cloud Shell works):


```bash
az aks update \
  --name <AKS_CLUSTER_NAME> \
  --resource-group <RESOURCE_GROUP> \
  --attach-acr <ACR_NAME>
```


Verify the pull works:


```powershell
kubectl run test-pull --image=<ACR_NAME>.azurecr.io/arweave-custom:2.9.5.1 --restart=Never -n default -- echo ok
kubectl get pod test-pull
kubectl delete pod test-pull
```


---


### Step: Update the StatefulSet Image Reference


Once the image is in ACR, update `03-statefulset.yaml` on the jumpbox:


Open the file in Notepad or VS Code and replace:
```
"<ACR_NAME>.azurecr.io/arweave-custom:2.9.5.1"
```
with your real ACR name, e.g.:
```
myregistry.azurecr.io/arweave-custom:2.9.5.1
```


---


## 5. Step-by-Step Deployment (Jumpbox / Windows)


All commands below run in **PowerShell on the Windows jumpbox** using `kubectl`.


### Step 1 — Copy manifests to the jumpbox


```powershell
# Manifests are already in:
# D:\nodesetup\Arweave\AKS\manifests\
cd D:\nodesetup\Arweave\AKS\manifests
```


### Step 2 — Confirm ACR image is ready


```powershell
# Verify the image exists in ACR before deploying
az acr repository show-tags --name <ACR_NAME> --repository arweave-custom --output table
```


Expected output:
```
Result
--------
2.9.5.1
```


### Step 3 — Apply manifests in order


```powershell
kubectl apply -f 00-namespace.yaml
kubectl apply -f 01-storageclass.yaml
kubectl apply -f 02-pvc.yaml
kubectl apply -f 03-statefulset.yaml
kubectl apply -f 04-service.yaml
```


### Step 4 — Watch pod startup


```powershell
# Pod phases: Pending → Init:0/1 → PodInitializing → Running 0/1 → Running 1/1
kubectl get pods -n arweave -w
```


Expected startup timeline:


| Phase | Duration | What's happening |
|---|---|---|
| `Pending` | 1-5 min | AKS schedules pod, provisions 3 Azure Disks (500Gi + 4TiB + 32Gi) |
| `Init:0/1` | 5-15 s | `init-config` writes `/config/config.json` |
| `PodInitializing` | 2-5 min | AKS pulls the ACR image onto the node |
| `Running 0/1` | 5-10 min | Arweave initialises account tree, joins network |
| `Running 1/1` | ✅ Ready | Readiness probe passes — port 1984 accepting connections |


> **Note:** The 3 PVCs (especially `arweave-storage` at 4 TiB) may take 2-5 minutes to provision from Azure. This is normal.


### Step 5 — Monitor init container logs


```powershell
# Init container — config.json write
kubectl logs -n arweave arweave-0 -c init-config


# Main container — Arweave startup
kubectl logs -n arweave arweave-0 -c arweave -f
```


Look for this in the main container logs to confirm the node joined the network:
```
Joined the Arweave network successfully at block <hash>, height <N>.
```


### Step 6 — Get the External LoadBalancer IP


```powershell
kubectl get svc -n arweave
```


Expected output when ready (EXTERNAL-IP usually takes 1-3 minutes):
```
NAME               TYPE           CLUSTER-IP    EXTERNAL-IP      PORT(S)          AGE
arweave-api        LoadBalancer   10.0.x.x      <PUBLIC-IP>      1984:xxxxx/TCP   3m
arweave-headless   ClusterIP      None          <none>           1984/TCP         3m
```


### Step 7 — Verify the node is running


```powershell
$IP = kubectl get svc arweave-api -n arweave -o jsonpath='{.status.loadBalancer.ingress[0].ip}'
curl.exe -s "http://${IP}:1984/info"
```


Expected response:
```json
{
  "network": "arweave.N.1",
  "version": 5,
  "release": 89,
  "height": 1975020,
  "blocks": 208,
  "peers": 5
}
```


---


## 6. Verification — curl Commands


> **No Docker on jumpbox** — use `curl.exe` (built into Windows) + `kubectl port-forward` for all verification.


### Get the LB IP


```powershell
$IP = kubectl get svc arweave-api -n arweave -o jsonpath='{.status.loadBalancer.ingress[0].ip}'
Write-Host "Arweave node: http://${IP}:1984"
```


### Node status


```powershell
# Full node info (height, blocks, peers, network name)
curl.exe -s "http://${IP}:1984/info"


# Current block
curl.exe -s "http://${IP}:1984/block/current"


# Connected peers (first 10)
curl.exe -s "http://${IP}:1984/peers"


# Storage modules status
curl.exe -s "http://${IP}:1984/storage_modules"
```


### Port-forward (use when LB IP is not yet assigned or for local testing)


```powershell
# Run in one PowerShell window — keep it open
kubectl port-forward pod/arweave-0 -n arweave 1984:1984


# Run in a second PowerShell window
curl.exe -s http://localhost:1984/info
curl.exe -s http://localhost:1984/block/current
curl.exe -s http://localhost:1984/peers
```


### Pretty-print JSON (PowerShell built-in)


```powershell
curl.exe -s "http://${IP}:1984/info" | ConvertFrom-Json | Format-List
```


---


## 7. Sync Monitoring


Arweave sync progress is tracked by comparing `blocks` (blocks synced so far) against `height` (total chain height). Sync is complete when `blocks == height`.


```powershell
# One-off check
$info = curl.exe -s "http://${IP}:1984/info" | ConvertFrom-Json
Write-Host "Height: $($info.height) | Blocks synced: $($info.blocks) | Peers: $($info.peers)"
```


```powershell
# Continuous monitor loop (checks every 60 seconds)
while ($true) {
  try {
    $info = curl.exe -s "http://${IP}:1984/info" | ConvertFrom-Json
    $pct = [math]::Round(($info.blocks / $info.height) * 100, 2)
    Write-Host "$(Get-Date -Format s) | Height: $($info.height) | Synced: $($info.blocks) ($pct%) | Peers: $($info.peers)"
  } catch {
    Write-Host "$(Get-Date -Format s) | Node unreachable"
  }
  Start-Sleep -Seconds 60
}
```


**Expected sync timeline:**


| Phase | Duration | Description |
|---|---|---|
| Account tree init | 5-10 min | Arweave initialises ETS/Merkle tables on first boot |
| Block header sync | Hours | Downloads all block headers (~1.97 million) from peers |
| Data sync | Days to weeks | Downloads transaction data into `/storage` partitions |
| Full archive | Months | Full ~373 TB weave — only if `arweave-storage` PVC is large enough |


---


## 8. Configuration Customisation


The node config (`/config/config.json`) is written by the `init-config` init container on first boot.


### View current config


```powershell
kubectl exec -n arweave arweave-0 -c arweave -- cat /config/config.json
```


### Edit config.json


```powershell
# Copy config to local machine
kubectl cp arweave/arweave-0:/config/config.json .\config.json


# Edit locally (Notepad, VS Code, etc.)
# Then copy back
kubectl cp .\config.json arweave/arweave-0:/config/config.json


# Restart pod to apply changes (StatefulSet recreates it automatically)
kubectl delete pod arweave-0 -n arweave
```


### Key configuration options


| Key | Default in manifests | Description |
|---|---|---|
| `sync_jobs` | `20` | Parallel sync workers — try `40` on high-CPU nodes |
| `mine` | `false` | Set `true` to enable mining (requires wallet + hugepages) |
| `mining_addr` | `REPLACE_WITH_WALLET_ADDRESS` | Your Arweave wallet address for mining rewards |
| `port` | `1984` | HTTP + P2P port (do not change without updating services) |
| `start_from_latest_state` | *(not set)* | Add `true` to resume from last persisted state after a crash |


### Enable mining (optional)


If you want to mine AR tokens:


1. **Generate a wallet** on a Linux machine using the Arweave binary (see `docker-setup.md`)
2. **Edit config.json**: set `mining_addr` to your wallet address and `mine: true`
3. **Hugepages**: Arweave RandomX requires `vm.nr_hugepages=3500` on the host. AKS nodes don't support this by default. See [Known Issues](#9-known-issues--resolutions) section.


---


## 9. Known Issues & Resolutions


### Issue 1 — `ImagePullBackOff` on arweave-0 pod


**Symptom:**
```
Events:
  Warning  Failed  ...  Failed to pull image "<ACR_NAME>.azurecr.io/arweave-custom:2.9.5.1": ...
            unauthorized: authentication required
```


**Cause:** ACR is not attached to the AKS cluster, or the image tag doesn't match.


**Fix:**
```bash
# Attach ACR (run from Cloud Shell or az CLI machine)
az aks update --name <AKS_CLUSTER_NAME> --resource-group <RG> --attach-acr <ACR_NAME>


# Verify image tag in ACR
az acr repository show-tags --name <ACR_NAME> --repository arweave-custom --output table
```


---


### Issue 2 — Pod stuck in `Pending` state — PVC not bound


**Symptom:**
```powershell
kubectl get pvc -n arweave
# STATUS shows Pending for more than 5 minutes
```


**Cause:** `WaitForFirstConsumer` means PVCs stay `Pending` until the pod is scheduled. If the pod itself is `Pending` (not enough CPU/memory), the PVCs stay stuck too.


**Fix:**
```powershell
# Check pod events
kubectl describe pod arweave-0 -n arweave


# Check node capacity
kubectl describe nodes | Select-String -Pattern "Allocated resources" -Context 0,6
```


If the node doesn't have enough capacity, scale up the AKS node pool or choose a larger VM SKU.


---


### Issue 3 — `0 peers` after startup


**Symptom:** Node starts but `/info` always shows `"peers": 0`.


**Cause:** NSG on the AKS subnet is blocking inbound TCP 1984.


**Fix:**
1. Go to Azure Portal → AKS node resource group → Network Security Group
2. Add inbound rule: Protocol TCP, Port 1984, Source Any, Action Allow
3. Restart the pod: `kubectl delete pod arweave-0 -n arweave`


---


### Issue 4 — `randomx_alloc_cache failed` error in logs


**Symptom:**
```
error: "randomx_alloc_cache failed"
```


**Cause:** AKS nodes do not support configuring `vm.nr_hugepages` (required by Arweave's RandomX proof-of-work engine).


**Impact on archival/sync:** The node will operate in **light mode** — block headers and transaction data still sync correctly. Mining will not work.


**Fix for archival-only nodes:** Add to `config.json`:
```json
"randomx_bulk_hashing_iterations": 0
```


**Fix for mining:** Deploy a privileged DaemonSet that sets `vm.nr_hugepages=3500` on each node pool VM, or use a custom AKS node image. This requires elevated cluster permissions — contact the platform team.


---


### Issue 5 — Version 2.9.5.1 Prometheus crash


**Symptom:** Pod keeps restarting with `[os_mon] cpu supervisor port (cpu_sup): Erlang has closed` in the logs.


**Cause:** Known bug in Arweave 2.9.5.1 — the Prometheus metrics collector can crash the Erlang VM on some environments.


**Fix:** Downgrade to stable version 2.9.4.1:
1. Rebuild the Docker image with the 2.9.4.1 binary (change the version in the `wget` URL during build)
2. Push new image to ACR: `arweave-custom:2.9.4.1`
3. Update `03-statefulset.yaml` — change image tag to `2.9.4.1`
4. Re-apply: `kubectl apply -f 03-statefulset.yaml`


---


### Issue 6 — `config.json` parsing error


**Symptom:**
```
Failed to parse config: unknown: {<<"mining">>,false}
```


**Cause:** Wrong key name. Arweave config uses `mine` (boolean), not `mining`.


**Fix:** Edit `/config/config.json` inside the pod and correct the key:
```json
"mine": false
```


---


### Issue 7 — Azure LB provisioning slow or fails


**Symptom:** `arweave-api` service stays in `<pending>` for `EXTERNAL-IP` for more than 10 minutes.


**Check:**
```powershell
kubectl describe svc arweave-api -n arweave
```


Common causes:
- Public IP quota exhausted in the AKS node resource group
- AKS cluster doesn't have the `LoadBalancer` service principal permission


**Fix:** Request a quota increase for Public IP addresses in the subscription, or contact the platform team to check AKS managed identity permissions.


---


### Summary Table


| # | Issue | Root Cause | Fix |
|---|---|---|---|
| 1 | `ImagePullBackOff` | ACR not attached to AKS | `az aks update --attach-acr` |
| 2 | PVC stuck `Pending` | Insufficient node capacity | Scale node pool / larger SKU |
| 3 | `peers: 0` | NSG blocking TCP 1984 inbound | Add NSG allow rule for TCP 1984 |
| 4 | `randomx_alloc_cache failed` | AKS nodes lack hugepages | Add `randomx_bulk_hashing_iterations: 0` for sync-only |
| 5 | Pod crash-looping | 2.9.5.1 Prometheus bug | Downgrade to 2.9.4.1 |
| 6 | Config parse error | Wrong key `mining` vs `mine` | Correct to `"mine": false` |
| 7 | LB stays `<pending>` | Public IP quota / permission | Request quota increase |


---


## 10. pvc-watcher Configuration


If you use the pvc-watcher auto-expansion service in this cluster, add the Arweave PVCs to its config.


> The `arweave-storage` PVC is 4 TiB by default. Enable watcher for `arweave-data` which grows as block headers accumulate. The `arweave-storage` PVC should be sized manually based on how many weave modules you intend to store.


```powershell
kubectl edit configmap pvc-watcher-script -n pvc-watcher
```


Add to `NODE_CONFIG` (expand `arweave-data` at 80% up to 2000Gi):
```
"arweave|arweave-0|/data|arweave-data|80|500|2000"
```


---


## 11. Upgrade Procedure


To upgrade the Arweave node to a new version (e.g. 2.9.6.x when released):


### Step 1 — Build new image


Repeat Section 4 (Option A — Azure Cloud Shell) with the new version number.


```bash
ARWEAVE_VERSION="2.9.6.0"   # update this
# ... same build steps as before
docker push ${ACR_NAME}.azurecr.io/arweave-custom:${ARWEAVE_VERSION}
```


### Step 2 — Update StatefulSet image tag


```powershell
# Edit 03-statefulset.yaml on the jumpbox
# Change:  myregistry.azurecr.io/arweave-custom:2.9.5.1
# To:      myregistry.azurecr.io/arweave-custom:2.9.6.0


kubectl apply -f 03-statefulset.yaml
```


### Step 3 — Trigger rolling update


```powershell
kubectl rollout restart statefulset arweave -n arweave
kubectl rollout status statefulset arweave -n arweave
```


### Step 4 — Verify


```powershell
kubectl logs -n arweave arweave-0 -c arweave | Select-String -Pattern "arweave 2.9"
curl.exe -s "http://${IP}:1984/info"
```


---


## 12. Monitoring & Ops Commands


### Pod health


```powershell
kubectl get pod arweave-0 -n arweave
kubectl describe pod arweave-0 -n arweave
kubectl top pod arweave-0 -n arweave
```


### Container logs


```powershell
# Live log stream
kubectl logs -n arweave arweave-0 -c arweave -f


# Last 100 lines
kubectl logs -n arweave arweave-0 -c arweave --tail=100


# Search logs for sync progress
kubectl logs -n arweave arweave-0 -c arweave | Select-String -Pattern "Added block|Synced up to|Joined the Arweave"
```


### Storage usage (inside the pod)


```powershell
kubectl exec -n arweave arweave-0 -c arweave -- df -h /data /storage /config
```


### Restart the pod (applies config changes, triggers re-sync from last state)


```powershell
kubectl delete pod arweave-0 -n arweave
# StatefulSet controller recreates it automatically
kubectl get pod -n arweave -w
```


### PVC usage


```powershell
kubectl get pvc -n arweave
```


### Full teardown (PVCs are retained — Azure Disks NOT deleted)


```powershell
kubectl delete statefulset arweave -n arweave
kubectl delete svc arweave-api arweave-headless -n arweave
kubectl delete pvc arweave-data arweave-storage arweave-config -n arweave
kubectl delete storageclass arweave-storage
kubectl delete namespace arweave
```


> After teardown, manually delete the three retained Azure Disks from the AKS node resource group in the Azure Portal if you want to stop incurring disk charges.


---


## 13. Node Endpoint Summary


> Fill in the External IP once provisioned.


| Service | Type | IP | Port | Purpose |
|---|---|---|---|---|
| `arweave-api` | Public LoadBalancer | `<FILL_AFTER_DEPLOY>` | 1984 | HTTP REST API + P2P |
| `arweave-headless` | ClusterIP (None) | n/a | 1984 | StatefulSet DNS identity |


### Quick-reference curl commands (PowerShell)


```powershell
$IP = "<FILL_AFTER_DEPLOY>"


curl.exe -s "http://${IP}:1984/info"             # Node status
curl.exe -s "http://${IP}:1984/block/current"    # Latest block
curl.exe -s "http://${IP}:1984/peers"            # Peer list
curl.exe -s "http://${IP}:1984/storage_modules"  # Weave storage status
```



