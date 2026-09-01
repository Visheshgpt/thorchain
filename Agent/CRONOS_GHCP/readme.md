# node_cronos — AKS Manifests


| Field | Value |
|---|---|
| **Blockchain** | Cronos Mainnet (`cronosmainnet_25-1`) |
| **Node type** | Archive node (`pruning = "nothing"`, `app-db-backend = "rocksdb"`) |
| **Daemon** | `cronosd` **v1.7.8** — latest upstream release (Jun 2026) |
| **Container image** | `ubuntu:22.04` + binary on PVC — no ACR, no registry |
| **Namespace** | `cronos` |
| **Node folder** | `blockchains/cronos/aks/` |
| **Volume mount** | `/data` → CRONOS_HOME `/data/.cronos` |
| **Storage** | `5000Gi` Premium_LRS · auto-expands to `8000Gi` via pvc-watcher |


> **Why not `cronos-archive:v1.7.8`?**
> Local VM build tag — not pullable in AKS. The `install-cronosd` init container replicates
> the VM Dockerfile exactly (`ubuntu:22.04` + `apt-get` + `wget` binary from GitHub Releases).
> No ACR required.


> ⚠️  **Before first deploy**: replace `SHA=TODO` in `03-node_cronos_statefulset.yaml`
> with the actual checksum from the v1.7.8 release `checksums.txt`.


## Apply Order


```bash
kubectl apply -f 00-node_cronos_namespace.yaml
kubectl apply -f 01-node_cronos_storageclass.yaml
kubectl apply -f 02-node_cronos_pvc.yaml
kubectl apply -f 03-node_cronos_statefulset.yaml
kubectl apply -f 04-node_cronos_service.yaml
```


First boot: 3 init containers → `restore-snapshot` (adopts migrated data) →
`install-cronosd` (mirrors Dockerfile) → `configure` (mirrors entrypoint.sh).


## Services


| Service | Type | Listener → Container |
|---|---|---|
| `node-cronos` | Headless ClusterIP | 26656, 26657, 1317, 9090, 8545, 8546, 26660 |
| `node-cronos-lb` | Internal LB | 5332 → 26656 (P2P) |
| `node-cronos-rpc-lb` | Internal LB | 2332→26657, 4332→1317, 8555→8545, 9090→9090, 8546→8546 |


See [full-doc.md](./full-doc.md) for data migration, verification commands, and pvc-watcher config.



