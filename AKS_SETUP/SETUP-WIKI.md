# THORChain Full Node — AKS Setup Guide


> Step-by-step guide for deploying a THORChain mainnet full node on Azure Kubernetes Service.
> Two sync methods are provided — pick one. Everything you need is in `manifests/`.


---


## 1. What we are building


A single **THORChain mainnet full node** (`chain-id: thorchain-1`) running in AKS.


```
Namespace: thorchain
├── ConfigMap: thornode-scripts       (init container shell scripts)
├── PVC:       thornode-data          (Azure Premium disk)
├── StatefulSet: thornode (1 replica)
│     ├── init-passwd     (writes /etc/passwd for UID 1025)
│     ├── init-thornode   (thornode init — genesis variant only)
│     ├── init-genesis    (downloads genesis via Liquify HTTPS)   ← genesis variant
│     ├── init-snapshot   (downloads pruned snapshot)             ← snapshot variant
│     └── thornode        (main container, runs image default fullnode.sh)
└── Services:
      ├── thornode-headless (in-cluster DNS)
      ├── thornode-p2p      (Internal Azure LB, port 27146)
      └── thornode-rpc      (Internal Azure LB, ports 27147/1317/9090/26660)
```


### Verified facts (from THORChain source + node-launcher)


| Item | Value |
|------|-------|
| Image | `registry.gitlab.com/thorchain/thornode:mainnet` |
| Image default CMD | `/scripts/fullnode.sh` (we use it as-is) |
| Image UID | root (0) — Gatekeeper forbids, we override to 1025 |
| Chain ID | `thorchain-1` (active since block 17,562,001) |
| P2P port | **27146** (mainnet — from `config_mainnet.go`) |
| RPC port | **27147** (mainnet — from `config_mainnet.go`) |
| REST / gRPC / Metrics | 1317 / 9090 / 26660 |
| Home dir | `/data/.thornode` (we set `HOME=/data`) |


---


## 2. Two sync methods — choose one


Both variants use the SAME namespace/PVC/services/ConfigMap. They differ only in which `03-statefulset*.yaml` you apply.


### A. Genesis sync — `03-statefulset-genesis.yaml`
- Replays every block from **17,562,001** to tip (~27M+).
- Sync time: **~5–10 days** on the recommended node pool.
- No external URL to keep updated — the genesis file is small (~42 MB) and downloaded once by the `init-genesis` container from Liquify's HTTPS gateway on port 443.
- Use this if you want a fully-verified state from genesis, or if the snapshot URL is broken.


### B. Snapshot sync — `03-statefulset.yaml`  ← recommended for prod
- Downloads a **pruned tarball (~40 GB)** from Liquify, extracts it, and catches up the last few thousand blocks.
- Sync time: **minutes to hours** after extraction.
- Requires a **fresh snapshot URL** — see step 4 below.


> ⚠️ **4-million-block sync is NOT possible.** That chain (`thorchain-mainnet-v1`) is dead — no live peers, no snapshots. Only `thorchain-1` (17.5M+) is syncable today.


---


## 3. Why AKS needs these specific hacks (context — read once)


Four AKS-specific problems required the current design. If you understand these, everything else makes sense.


1. **AKS blocks outbound TCP on random ports.** `thornode render-config` normally pulls genesis from live THORChain peers on **port 27147**, which fails from AKS pods with `context deadline exceeded`. **Fix:** we pre-populate `genesis.json` via **Liquify's HTTPS gateway on port 443** BEFORE the main container starts.
2. **Gatekeeper forbids root, image has no non-root user.** The image runs as UID 0. We run as UID 1025, but THORChain's Go code calls `os/user.Current()` and panics without a matching `/etc/passwd` entry. **Fix:** `init-passwd` writes a passwd file into a shared emptyDir which is mounted over `/etc/passwd` (subPath).
3. **The old `YggdrasilVault` type is rejected by the current binary.** **Fix:** the `init-genesis` script does `sed 's/YggdrasilVault/AsgardVault/g'`.
4. **Multi-line shell embedded in YAML `|` blocks kept breaking** (indent + heredoc issues that took down previous attempts). **Fix:** all init scripts live in `03-configmap-scripts.yaml` and are mounted into the containers — YAML never sees the shell.


---


## 4. Prerequisites


- Jumpbox with `kubectl` + `kubelogin` + Azure CLI.
- VPN + Azure AD access to the AKS cluster.
- An AKS node pool with a node large enough for the request/limit values in the StatefulSet.
  - Minimum node size: **`Standard_D4s_v3`** (4 vCPU / 16 GiB) — barely fits.
  - Recommended: **`Standard_D8s_v3`** (8 vCPU / 32 GiB) dedicated pool.
- If you plan to use the **snapshot variant**, grab the current URL from:
  - https://snapshots.liquify.com/
  - https://snapshots.ninerealms.com/


---


## 5. Deployment — step by step


All commands from the jumpbox in PowerShell.


### 5.1 Connect to the cluster


```powershell
az login
az aks get-credentials --resource-group azrg-cus-coinia-aks-dev --name cuscoiniadevaks
kubelogin convert-kubeconfig -l azurecli
kubectl get nodes
```


### 5.2 Go to the manifests folder


```powershell
cd "d:\nodesetup\Thorchain\AKS setup\manifests"
```


### 5.3 Apply the base objects (both variants need these)


```powershell
kubectl apply -f 00-namespace.yaml
kubectl apply -f 01-storageclass.yaml
kubectl apply -f 02-pvc.yaml
kubectl apply -f 03-configmap-scripts.yaml
kubectl apply -f 04-service.yaml
```


### 5.4 Choose ONE StatefulSet


**Genesis sync:**


```powershell
kubectl apply -f 03-statefulset-genesis.yaml
```


**Snapshot sync:** edit `03-statefulset.yaml` first — replace the placeholder:


```yaml
- name: SNAPSHOT_URL
  value: "REPLACE_WITH_SNAPSHOT_URL"       # ← paste real Liquify URL here
```


Then apply:


```powershell
kubectl apply -f 03-statefulset.yaml
```


> ⚠️ **Do not apply both** — they share the name `thornode`. The second one will overwrite the first.


---


## 6. Watch it come up


```powershell
kubectl get pods -n thorchain -w
```


Expected sequence for **genesis** variant:


```
thornode-0   0/3    Init:0/3     ← init-passwd
thornode-0   0/3    Init:1/3     ← init-thornode (thornode init)
thornode-0   0/3    Init:2/3     ← init-genesis (Liquify HTTPS download, ~1 min)
thornode-0   1/1    Running      ← fullnode.sh → thornode start
```


Expected sequence for **snapshot** variant:


```
thornode-0   0/2    Init:0/2     ← init-passwd
thornode-0   0/2    Init:1/2     ← init-snapshot (~15–40 min, depends on network)
thornode-0   1/1    Running      ← catches up remaining blocks in minutes
```


Follow logs of the interesting container:


```powershell
kubectl logs -n thorchain thornode-0 -c init-genesis -f
kubectl logs -n thorchain thornode-0 -c init-snapshot -f
kubectl logs -n thorchain thornode-0 -c thornode -f
```


Check sync height (once `Running`):


```powershell
kubectl exec -n thorchain thornode-0 -c thornode -- \
  wget -qO- http://localhost:27147/status | \
  Select-String -Pattern 'latest_block_height'
```


---


## 7. After the ILB is up — set `EXTERNAL_IP`


The p2p ILB gets a private VNet IP. Feed it back to the pod so peers can dial in:


```powershell
$IP = kubectl get svc thornode-p2p -n thorchain -o jsonpath='{.status.loadBalancer.ingress[0].ip}'
kubectl set env sts/thornode -n thorchain EXTERNAL_IP=$IP
kubectl rollout status sts/thornode -n thorchain
```


---


## 8. Files in this folder


| File | Purpose |
|------|---------|
| `manifests/00-namespace.yaml` | Namespace `thorchain` |
| `manifests/01-storageclass.yaml` | Azure `Premium_LRS` StorageClass (Retain policy) |
| `manifests/02-pvc.yaml` | `thornode-data` PVC — 200 Gi (test) / 2 Ti (prod) |
| `manifests/03-configmap-scripts.yaml` | Init scripts (`init-passwd.sh`, `init-genesis.sh`, `init-snapshot.sh`) |
| `manifests/03-statefulset-genesis.yaml` | **Genesis sync** StatefulSet |
| `manifests/03-statefulset.yaml` | **Snapshot sync** StatefulSet |
| `manifests/04-service.yaml` | headless + P2P LB + RPC LB |


---


## 9. Troubleshooting


| Symptom | Meaning | Fix |
|---------|---------|-----|
| `Init:CrashLoopBackOff` on `init-genesis` | Liquify gateway unreachable or slow | `kubectl logs ... -c init-genesis` — retry, then check egress firewall rules for `gateway.liquify.com:443` |
| `panic: user: unknown userid 1025` | `init-passwd` skipped or `/etc/passwd` mount lost | Confirm `passwd-vol` emptyDir + subPath mount in main container |
| `panic: invalid vault type value: YggdrasilVault` | genesis wasn't patched | Delete the PVC, redeploy — `init-genesis` script runs the sed fix |
| `EOF failed to parse chain-id from genesis` | genesis.json is empty/partial | Delete PVC, redeploy — the download failed mid-way |
| `context deadline exceeded` on port 27147 | Something is still trying to call `render-config` genesis fetch on 27147 | Should not happen — genesis is pre-populated. Check that `03-configmap-scripts.yaml` was applied. |
| Snapshot variant fails at `init-snapshot` | Stale `SNAPSHOT_URL` (Liquify rotates URLs) | Get a fresh URL from https://snapshots.liquify.com/ and re-apply |


### Nuclear reset (wipes state, keeps disk)


```powershell
kubectl delete sts thornode -n thorchain
kubectl delete pvc thornode-data -n thorchain     # PV is Retain — Azure disk survives
kubectl apply -f 02-pvc.yaml
kubectl apply -f 03-statefulset-genesis.yaml       # or 03-statefulset.yaml
```



Reference links
https://docs.thorchain.org/thornodes/overview/node-operations
https://docs.thorchain.org/thornodes/overview/thornode-stack
https://docs.thorchain.org/thornodes/deploying#deploy-thornode
https://docs.thorchain.org/thornodes/fullnode/thornode-kubernetes
https://docs.thorchain.org/thornodes/fullnode/thornode-docker