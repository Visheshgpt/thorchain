# node_cronos — AKS Manifests

| Field | Value |
|---|---|
| **Blockchain** | Cronos EVM — `cronosmainnet_25-1` |
| **Daemon** | `cronosd` **v1.7.8** (release binary on `ubuntu:22.04`) |
| **DB backend** | **`rocksdb`** — matches the production VM |
| **Bootstrap** | Cronos native `rocksdb-pruned` snapshot (~22 GB) |
| **Namespace** | `cronos` · pod `node-cronos-0` · container `cronosd` |
| **Data mount** | `/data` → `CRONOS_HOME=/data/.cronos` |
| **Storage** | `256Gi` StandardSSD_LRS · `reclaimPolicy: Delete` |

> **`reclaimPolicy: Delete` is TEST-ONLY.** Deleting the PVC destroys the disk with it.
> Switch back to `Retain` before this ever holds migrated prod data.

## Apply Order

```bash
kubectl apply -f 00-namespace.yaml
kubectl apply -f 01-storageclass.yaml
kubectl apply -f 02-pvc.yaml
kubectl apply -f 03-statefulset.yaml
kubectl apply -f 04-service.yaml
```

Three init containers run first: `restore-snapshot` (~22 GB, 20–25 min) →
`install-cronosd` → `configure`.

> **A StatefulSet will not roll a pod that is not Ready.** During startup `kubectl apply`
> silently does nothing — always follow it with `kubectl delete pod node-cronos-0 -n cronos`.

## Switching DB backend

`DB_BACKEND` drives both the snapshot source and the config, so they cannot diverge:

```bash
kubectl set env statefulset/node-cronos -n cronos -c restore-snapshot DB_BACKEND=goleveldb RESET_DATA=switch-1
kubectl set env statefulset/node-cronos -n cronos -c configure DB_BACKEND=goleveldb
kubectl delete pod node-cronos-0 -n cronos
```

| value | snapshot | size |
|---|---|---|
| `rocksdb` | Cronos native `rocksdb-pruned` | ~22 GB |
| `goleveldb` | PublicNode `cronos-pruned` | ~20 GB |

See [fulldoc.md](./fulldoc.md) for verification, RPC commands, and the sync proof.
