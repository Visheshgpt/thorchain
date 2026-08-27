# node_kava — AKS Manifests

| Field | Value |
|---|---|
| **Blockchain** | Kava — `kava_2222-10` |
| **Daemon** | `kava` v0.28.0 |
| **Container image** | `kava/kava:v0.28.0-rocksdb` (official, public) |
| **DB backend** | **`rocksdb`** — matches the production VM |
| **Namespace** | `kava` |
| **Bootstrap** | **state sync** (no snapshot download) |
| **Storage** | `100Gi` StandardSSD_LRS · auto-expands to `500Gi` via pvc-watcher |
| **Resources** | 2 CPU / 4Gi request · **10Gi memory limit**, no CPU limit |

> **The memory limit is 10Gi on purpose. Do not raise it.** A 24Gi limit made state sync
> fail repeatably (230 of 232 chunks, three runs). See `fulldoc.md` §8.

## Apply Order

```bash
kubectl apply -f 00-namespace.yaml
kubectl apply -f 01-storageclass.yaml
kubectl apply -f 02-pvc.yaml
kubectl apply -f 03-statefulset.yaml
kubectl apply -f 04-service.yaml
```

Two init containers run first — `prepare-statesync` (seconds) and `configure` (computes a live
trust point). State sync itself takes **10–15 minutes**. No 35 GiB download.

## Services

| Service | Type | Ports |
|---|---|---|
| `node-kava` | Headless ClusterIP | 26656, 26657, 1317, 9090 |
| `node-kava-lb` | Internal LB | 26656 (P2P) |
| `node-kava-rpc-lb` | Internal LB | 26657, 1317, 9090 |

## Stop / start without re-syncing

```bash
kubectl scale statefulset node-kava -n kava --replicas=0    # stop
kubectl scale statefulset node-kava -n kava --replicas=1    # resume
```

`reclaimPolicy: Retain` — the PVC and chain data survive.

See [fulldoc.md](./fulldoc.md) for the full guide and the RPC verification ladder.
