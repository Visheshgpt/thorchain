# node_cronos — Full Deployment Guide


## Node Overview


| Field | Value |
|---|---|
| **Blockchain** | Cronos Mainnet (`cronosmainnet_25-1`) |
| **Node type** | Archive node (`pruning = "nothing"`) |
| **Daemon** | `cronosd` **v1.7.8** (latest upstream release, Jun 2026) |
| **Container image** | `ubuntu:22.04` + binary on PVC — no ACR, no registry |
| **Namespace** | `cronos` |
| **Pod** | `node-cronos-0` |
| **Volume mount** | `/data` (matches Dockerfile `VOLUME ["/data"]`) |
| **CRONOS_HOME** | `/data/.cronos` (matches `entrypoint.sh` `CRONOS_HOME="/data/.cronos"`) |
| **Original VM data** | `/CronosDisk/node-cronos-archive/data/.cronos/` |
| **DB backend** | RocksDB (`app-db-backend = "rocksdb"`) |
| **Storage** | `5000Gi` Premium_LRS · auto-expands to `8000Gi` via pvc-watcher |


> **Why `ubuntu:22.04` not `cronos-archive:v1.7.8`?**
> The internal tag `cronos-archive:v1.7.8` is a local VM build — not pullable inside AKS.
> The `install-cronosd` init container replicates the Dockerfile RUN block exactly:
> `ubuntu:22.04` + `apt-get install curl wget jq tar` + `wget` binary from GitHub Releases.
> No ACR required.


---


## Prerequisites


### 1. Get the SHA256 checksum for the binary


Replace the `TODO` placeholder in `03-node_cronos_statefulset.yaml` before deploying:


```bash
curl -fsSL https://github.com/crypto-org-chain/cronos/releases/download/v1.7.8/checksums.txt \
  | grep cronos_1.7.8_Linux_x86_64.tar.gz
# Copy the hash into SHA= in the install-cronosd init container
```


### 2. Migrate existing archive data from the old VM (recommended path)


The original docker-compose mounted `/CronosDisk/node-cronos-archive/data:/data`.
Inside the container, `CRONOS_HOME="/data/.cronos"` — so the full chain data on the VM is at:
```
/CronosDisk/node-cronos-archive/data/.cronos/
```


Migrate to the PVC before first boot:


```bash
# Step A: Create the namespace, StorageClass, and PVC
kubectl apply -f 00-node_cronos_namespace.yaml
kubectl apply -f 01-node_cronos_storageclass.yaml
kubectl apply -f 02-node_cronos_pvc.yaml


# Step B: Wait for PVC to bind
kubectl get pvc -n cronos -w


# Step C: Spin up a migration pod to mount the PVC
kubectl run cronos-migrate -n cronos --image=ubuntu:22.04 --restart=Never \
  --overrides='{
    "spec":{
      "volumes":[{"name":"data","persistentVolumeClaim":{"claimName":"node-cronos-data"}}],
      "containers":[{
        "name":"migrate","image":"ubuntu:22.04",
        "command":["sleep","7200"],
        "volumeMounts":[{"name":"data","mountPath":"/data"}]
      }]
    }}'


# Step D: Rsync from the old VM (run this on the jumpbox or old VM)
# Replace <node-ip> with the migration pod IP:
MIGRATE_POD_IP=$(kubectl get pod cronos-migrate -n cronos -o jsonpath='{.status.podIP}')
rsync -avz --progress \
  /CronosDisk/node-cronos-archive/data/.cronos/ \
  root@${MIGRATE_POD_IP}:/data/.cronos/


# Step E: Delete the migration pod
kubectl delete pod cronos-migrate -n cronos
```


`ADOPT_EXISTING=true` is the default in the StatefulSet — the `restore-snapshot`
init container will detect the data and skip snapshot pulling automatically.


### 3. If starting from a pruned Polkachu snapshot (fresh archive going forward)


Set `ADOPT_EXISTING=false` in the `restore-snapshot` env block.
Polkachu Cronos snapshots: https://polkachu.com/tendermint_snapshots/cronos


> ⚠️ Pruned snapshot = `debug_traceTransaction` will fail on blocks before snapshot height.


---


## Apply Order


```bash
kubectl apply -f 00-node_cronos_namespace.yaml
kubectl apply -f 01-node_cronos_storageclass.yaml
kubectl apply -f 02-node_cronos_pvc.yaml


# Verify PVC is Bound before applying StatefulSet
kubectl get pvc -n cronos


kubectl apply -f 03-node_cronos_statefulset.yaml
kubectl apply -f 04-node_cronos_service.yaml
```


First boot runs three init containers:
1. `restore-snapshot` — adopts migrated data (or pulls Polkachu snapshot)
2. `install-cronosd` — `apt-get install` + `wget` binary from GitHub (mirrors Dockerfile)
3. `configure` — `cronosd init` + `genesis.json` + all `app.toml`/`config.toml` patches


---


## Verification Commands


```bash
# Pod status
kubectl get pods -n cronos


# All services and ILB IPs
kubectl get svc -n cronos


# PVC status
kubectl get pvc -n cronos


# Init container logs (follow bootstrap progress)
kubectl logs -n cronos node-cronos-0 -c restore-snapshot -f
kubectl logs -n cronos node-cronos-0 -c install-cronosd -f
kubectl logs -n cronos node-cronos-0 -c configure -f


# Main container logs (same as `docker logs` on the old VM)
kubectl logs -n cronos node-cronos-0 -c cronosd -f


# Describe pod (events, probe failures, scheduling)
kubectl describe pod node-cronos-0 -n cronos


# CometBFT health
kubectl exec -n cronos node-cronos-0 -- \
  /data/bin/cronosd status --home /data/.cronos


# Block height via RPC (after svc has ILB IP)
curl http://<rpc-lb-ip>:2332/status | python3 -m json.tool | grep latest_block_height


# EVM JSON-RPC test
curl -X POST http://<rpc-lb-ip>:8555 \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","method":"eth_blockNumber","params":[],"id":1}'


# Cosmos REST
curl http://<rpc-lb-ip>:4332/cosmos/base/tendermint/v1beta1/syncing
```


---


## Config Values Applied (from entrypoint.sh)


| Key | Value | Source |
|---|---|---|
| `pruning` | `"nothing"` | entrypoint.sh — archive |
| `minimum-gas-prices` | `"0.025basecro"` | entrypoint.sh |
| `app-db-backend` | `"rocksdb"` | entrypoint.sh |
| `query-gas-limit` | `"100000000"` | entrypoint.sh (v1.5.1+ upgrade) |
| `CHAIN_ID` | `cronosmainnet_25-1` | entrypoint.sh |
| Genesis URL | `cronosmainnet_25-1/genesis.json` | entrypoint.sh |
| `laddr` (RPC) | `tcp://0.0.0.0:26657` | K8s addition (Docker handled via port-mapping) |
| EVM JSON-RPC | `0.0.0.0:8545/8546` | K8s addition (enabled explicitly) |


---


## Auto-Disk Config (pvc-watcher)


```
# pvc-watcher NODE_CONFIG — copy this line into the ConfigMap
# Format: "NAMESPACE|POD|MOUNT_PATH|PVC_NAME|THRESHOLD|INCREMENT_GI|MAX_SIZE_GI"
"cronos|node-cronos-0|/data|node-cronos-data|80|500|8000"
```


Apply to the live cluster:


```bash
kubectl edit configmap pvc-watcher-script -n pvc-watcher
# Add the line above inside the NODE_CONFIGS list


# Or via kubectl patch:
kubectl patch configmap pvc-watcher-script -n pvc-watcher --type=merge \
  -p '{"data":{"NODE_CONFIGS":"<existing>\ncronos|node-cronos-0|/data|node-cronos-data|80|500|8000"}}'
```


> Mount path is `/data` (PVC root) — not `/data/.cronos`. The pvc-watcher measures
> disk usage at the volume mount point.


---


## Port Reference


| Container Port | LB Listener | Service | Purpose |
|---|---|---|---|
| `26656` | `5332` | `node-cronos-lb` | CometBFT P2P |
| `26657` | `2332` | `node-cronos-rpc-lb` | CometBFT RPC |
| `1317` | `4332` | `node-cronos-rpc-lb` | Cosmos REST API |
| `9090` | `9090` | `node-cronos-rpc-lb` | Cosmos gRPC |
| `8545` | `8555` | `node-cronos-rpc-lb` | EVM JSON-RPC HTTP (`eth_`, `debug_trace*`) |
| `8546` | `8546` | `node-cronos-rpc-lb` | EVM JSON-RPC WebSocket |
| `26660` | headless only | — | Prometheus metrics |


---


## Force a New Snapshot


To wipe data and pull a fresh Polkachu pruned snapshot:


1. Edit StatefulSet: set `FORCE_RESTORE` to any new string (e.g. `"2026-09-01"`)
2. Set `ADOPT_EXISTING` to `"false"`
3. Save — the pod restarts and runs the snapshot restore


---


## Upgrade cronosd Version


1. Update `VERSION="1.7.8"` → new version in `install-cronosd` init container
2. Update `SHA=` with the new checksum from the release's `checksums.txt`
3. `kubectl rollout restart statefulset/node-cronos -n cronos`
4. The init container will detect the binary is outdated (or delete `/data/bin/cronosd` manually first)


---


## Teardown


```bash
kubectl delete -f 04-node_cronos_service.yaml
kubectl delete -f 03-node_cronos_statefulset.yaml
kubectl delete -f 02-node_cronos_pvc.yaml
kubectl delete -f 01-node_cronos_storageclass.yaml
kubectl delete -f 00-node_cronos_namespace.yaml
```


> ⚠️  PVC deletion removes only the Kubernetes object.
> The underlying Azure Managed Disk is **RETAINED** (reclaimPolicy: Retain).
> Delete it manually in the Azure portal or via:
> ```bash
> az disk delete --name <disk-name> --resource-group <rg> --yes
> ```


---


## Notes


- **SHA256** — the `install-cronosd` init container has a `TODO` for the binary checksum.
  Get it from: `https://github.com/crypto-org-chain/cronos/releases/download/v1.7.8/checksums.txt`


- **RocksDB + RAM** — Cronos docs recommend 32 GB (LevelDB) or 64 GB (RocksDB) for archive.
  Current `limits.memory: 32Gi`. Raise to `64Gi` if heavy `debug_trace*` workloads are expected.


- **MONIKER** — `pegasus-node`, from the original docker-compose `MONIKER=pegasus-node`.


- **apt-get in init containers** — `install-cronosd` and `configure` run `apt-get update`
  each pod start. This is ~20–30s and requires outbound internet. It mirrors the Dockerfile
  exactly and is the correct approach without an ACR.


- **EVM JSON-RPC not in entrypoint.sh** — The original entrypoint.sh does not explicitly
  enable EVM JSON-RPC. It was likely pre-configured in the VM's `app.toml`. The `configure`
  init container now sets it explicitly to ensure a fresh K8s deploy gets the correct bindings.



