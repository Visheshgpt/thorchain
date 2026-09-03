# node_terra2 — AKS Manifests


| Field | Value |
|---|---|
| **Blockchain** | Terra 2.0 mainnet — `phoenix-1` |
| **Node type** | Pruned-snapshot restore, **archive-forward** (`pruning = "nothing"`, `min-retain-blocks = 0`) — see the pruning-floor warning below |
| **Daemon** | `terrad` **v2.20.0** (matches live `phoenix-1` — confirmed via `/abci_info`) |
| **Container image** | `alpine:3.19` + checksum-verified `terrad` binary installed onto the PVC — `phoenix-directive/core` publishes no image |
| **Namespace** | `terra2` |
| **Node folder** | `blockchains/AKS-Migration/node_terra2/aks/` |
| **Volume mount** | `/app` → `terrad --home` (matches compose bind mount) |
| **Storage** | `200Gi` StandardSSD_LRS · auto-expands to `3000Gi` via pvc-watcher |
| **Bootstrap** | Six init containers — `restore-snapshot` (Polkachu tarball, ~30 GB, ~20–60 min), `install-terrad`, `terrad-init`, `install-genesis`, `configure`, `rollback-recovery` |


## Apply Order


```bash
kubectl apply -f 00-namespace.yaml
kubectl apply -f 01-storageclass.yaml
kubectl apply -f 02-pvc.yaml
kubectl apply -f 03-statefulset.yaml
kubectl apply -f 04-service.yaml
```


> ⚠️ **`kubectl apply` alone will not roll this pod.** `podManagementPolicy:
> OrderedReady` refuses to replace a pod that has never become Ready — and a
> node still restoring a snapshot is not Ready. An `apply` against an unready
> pod is a **silent no-op**. After applying, delete the pod explicitly:
> ```bash
> kubectl delete pod node-terra2-0 -n terra2
> ```
> Then confirm the new spec actually took effect:
> ```bash
> kubectl get pod node-terra2-0 -n terra2 -o jsonpath="{.spec.initContainers[*].name}"
> ```
> This cost hours during bring-up: logs were read from a pod running a spec
> several revisions old.


## Is it running?


```bash
kubectl get pods -n terra2
kubectl logs node-terra2-0 -n terra2 -c terra --tail=40
```

Then the check that actually means something:

```bash
kubectl exec node-terra2-0 -n terra2 -c terra -- wget -qO- localhost:26657/status
```

`catching_up: true` with `latest_block_height` climbing is a healthy catch-up.

> ⚠️ **`/health` is not a sync check.** CometBFT's `/health` returns an empty
> `200` unconditionally and never inspects sync state. A node that is
> stalled, or 5 million blocks behind, returns exactly the same `{}` as one at
> the tip. Use `sync_info` from `/status`. Ready ≠ synced.


## Historical depth — read this before querying


> ⚠️ **The Polkachu snapshot is pruned** (`custom/100/10`). This node is an
> archive **going forward from the snapshot height** (~22,654,000 as of
> Sep 2026), but it holds **no state below that height**. `/block?height=N`
> for N below the floor returns "not available" — and that is correct
> behaviour, not a fault.
>
> Get the real floor from `earliest_block_height` in `/status`. Do **not** use
> `/block?height=1` as an acceptance test; it will fail on a healthy node.
>
> For true archive back to block 1, migrate the source VM's
> `/1Disk116/node-terra/data` per fulldoc §2.3 and set `ADOPT_EXISTING=true`
> on `restore-snapshot` so it does not overwrite the migrated data. That needs
> a far larger PVC — it is a different deployment, not a config change.


## Chain upgrades are manual


Cosmovisor is not used. The binary lives on the volume, not in the image, so
`kubectl set image` does **not** apply. To roll a new release, edit the
`install-terrad` init container in `03-statefulset.yaml` — bump `VER` and
`SHA` (from the release's `checksum.txt`) — then restart the pod. The version
sentinel at `/app/bin/.terrad-version` forces a reinstall whenever `VER`
changes.

Track [phoenix-directive/core releases](https://github.com/phoenix-directive/core/releases).

> ⚠️ **`terra-money/core` is abandoned** — `main` frozen 2024-03-07, tags stop
> at 2.12.4, upgrade handlers only reach v2.10. Do not track it. The Cosmos
> chain-registry lists `phoenix-directive/core` as `phoenix-1`'s `git_repo`.
> Watch mainnet governance and roll the version before each upgrade height.


> ℹ️ **Why not state-sync?** Verified empirically (2026-09): the Terra 2.0 P2P
> network has no active state-sync provider — every discovered snapshot was at
> height 14 M, outside CometBFT's 168 h trust window. Polkachu themselves
> point Terra operators to their tarball snapshot, so we mirror that.


> ℹ️ **The 8 KB genesis is deliberate.** `phoenix-1`'s original genesis is
> 747 MB, but the network does not run it — both Polkachu and publicnode serve
> a 7,988-byte genesis and return `total: 1` from `/genesis_chunked`. CometBFT
> stores the genesis hash in `state.db` and compares it on every start, so a
> Polkachu-restored snapshot requires Polkachu's genesis. Installing the
> "official" 747 MB file fails with `genesis doc hash in db does not match`.


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


See [fulldoc.md](./fulldoc.md) for the complete deployment guide, the
root-cause diagnosis from bring-up, snapshot migration steps, the RPC
verification ladder, sync proof, and pvc-watcher config.
