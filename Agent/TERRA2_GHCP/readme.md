# node_terra2 — AKS Manifests


| Field | Value |
|---|---|
| **Blockchain** | Terra 2.0 mainnet — `phoenix-1` |
| **Node type** | Archive (`PROFILE=archive`, `PRUNING_STRATEGY=nothing`) |
| **Daemon** | `terrad` **v2.20.0** (matches live phoenix-1 — confirmed via `/abci_info`) |
| **Container image** | `alpine:3.19` + verified `terrad` binary on the PVC — `phoenix-directive/core` publishes no image |
| **Namespace** | `terra2` |
| **Node folder** | `blockchains/AKS-Migration/node_terra2/aks/` |
| **Volume mount** | `/app` → terrad `--home` and image `WORKDIR` (matches compose bind mount) |
| **Storage** | `200Gi` StandardSSD_LRS · auto-expands to `3000Gi` via pvc-watcher |
| **Bootstrap** | Four init containers — `restore-snapshot` (Polkachu tarball, ~28 GB, ~20–60 min), `terrad-init`, `install-genesis`, `configure` |


> ⚠️ **The Polkachu snapshot is pruned** (`pruning-keep-recent=100`).
> This node will be an archive going forward from the snapshot height (~22.6 M
> as of Sep 2026), but `/block?height=N` for N below that height will return
> "block not available". To get true archive back to block 1, migrate the
> source VM's `/1Disk116/node-terra/data` per full-doc §2.3 (set
> `ADOPT_EXISTING=true` on the `restore-snapshot` init so it does not
> overwrite the migrated data).


> ℹ️ **Why not state-sync?** Verified empirically (2026-09): the Terra 2.0 P2P
> network has no active state-sync provider — every discovered snapshot was
> at height 14 M, outside CometBFT's 168 h trust window. Polkachu themselves
> point Terra operators to their tarball snapshot, so we mirror that.


> **Why not `terraformlabs/cosmovisor:terra-mainnet-edge`?** That reference
> produces `ImagePullBackOff` — the whole `terraformlabs` Docker Hub namespace
> returns 404. It was a *local build tag* built on the source VM (same class
> of bug as CronosPOS's `cryptocom/chain-main:v8.0.0`). The working image is
> a checksum-verified `terrad` v2.20.0 binary installed onto the PVC,
> runs as UID 1000, WORKDIR `/app`, default CMD `terrad --home /app start`.


> ⚠️ **Cosmovisor is not in the upstream image**, so chain upgrades are now
> **manual**: at each mainnet upgrade height, roll the image tag:
> ```


> ⚠️ **Archive node warning:** syncing Terra 2.0 from block 0 takes weeks.
> Pre-load a snapshot (see full-doc §2.3) before scaling `replicas` to 1, or
> the pod's readiness will stay false for weeks.


> ⚠️ **Archive node warning:** syncing Terra 2.0 from block 0 takes weeks.
> Pre-load a snapshot (see full-doc §2.3) before setting `replicas: 1`, or
> the pod's readiness will stay false for weeks.


## Apply Order


```bash
kubectl apply -f 00-node_terra2_namespace.yaml
kubectl apply -f 01-node_terra2_storageclass.yaml
kubectl apply -f 02-node_terra2_pvc.yaml
kubectl apply -f 03-node_terra2_statefulset.yaml
kubectl apply -f 04-node_terra2_service.yaml
```


## Services


| Service | Type | Ports |
|---|---|---|
| `node-terra2` | Headless ClusterIP | 26656, 26657, 1317, 9090 |
| `node-terra2-lb` | Internal LB | 26656 (P2P) — external requires firewall port-enable |
| `node-terra2-rpc-lb` | Internal LB | 26657 (RPC), 1317 (REST), 9090 (gRPC — auto-added) |


## Auto-added port


Port `9090` (Cosmos gRPC) is not in the source `docker-compose.yaml`. The
generator auto-adds it for every Cosmos SDK chain so the NextGen team can
query balances, transactions, and staking data over gRPC directly.


## External P2P (`PUBLIC_ADDRESS=13.86.34.113:26656`)


The compose file advertises a public IP for inbound peers. The AKS P2P LB is
**internal** by default and cannot receive that traffic. To restore inbound
peering, raise a DevOps firewall port-enable request for TCP `26656` — do
NOT flip the service type without written approval. Outbound peering already
works from the pod egress path, and the node will sync fine without inbound
peers.


See [full-doc.md](./full-doc.md) for the complete deployment guide, snapshot
migration steps, RPC verification ladder, and pvc-watcher config.



