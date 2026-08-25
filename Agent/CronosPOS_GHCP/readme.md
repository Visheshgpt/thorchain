# node_cronospos — AKS Manifests

| Field | Value |
|---|---|
| **Blockchain** | CronosPOS — `crypto-org-chain-mainnet-1` |
| **Daemon** | `chain-maind` **v8.0.0** (matches the network's live consensus version) |
| **Container image** | `alpine:3.19` + the SHA256-verified upstream release binary — see note below |
| **Namespace** | `cronospos` |
| **Node folder** | `blockchains/node_cronospos/aks/` |
| **Storage** | `100Gi` StandardSSD_LRS · auto-expands to `500Gi` via pvc-watcher |
| **Bootstrap** | Polkachu pruned snapshot, restored automatically by an init container |

> **Why not `cryptocom/chain-main:v8.0.0`?** That image is **not publicly pullable** — every tag
> returns HTTP 401 on Docker Hub and GHCR, while `library/alpine` returns 200 on the identical
> probe. It is a *local build tag*: upstream's own Dockerfile opens with
> `# > docker build -t cryptocom/chain-main .`. Referencing it is what produced the original
> `ImagePullBackOff`. The upstream v8.0.0 release binary is fully static, so an init container
> downloads and SHA256-verifies it and the node runs it on public alpine. No ACR required.

## Apply Order

```bash
kubectl apply -f 00-namespace.yaml
kubectl apply -f 01-storageclass.yaml
kubectl apply -f 02-pvc.yaml
kubectl apply -f 03-statefulset.yaml
kubectl apply -f 04-service.yaml
```

First boot runs three init containers — `restore-snapshot` (streams ~9 GB, about 5 min),
`install-chain-maind`, then `configure`. No manual data seeding is needed.

## Services

| Service | Type | Ports |
|---|---|---|
| `node-cronospos` | Headless ClusterIP | 26656, 26657, 1317, 9090 |
| `node-cronospos-lb` | Internal LB | 26656 (P2P) |
| `node-cronospos-rpc-lb` | Internal LB | 1332→26657, 2332→1317, 3332→9090 |

See [fulldoc.md](./fulldoc.md) for the full deployment guide and the RPC verification ladder.
