# Cronos POS (Crypto.org Chain) — mainnet full node on AKS

Pruned full node, bootstrapped from a public snapshot. Generated from `docker-compose.yaml`
by the process in `../agent_v2.md`.

| | |
|---|---|
| chain-id | `crypto-org-chain-mainnet-1` |
| daemon | `chain-maind` **v8.0.0** |
| CometBFT | 0.38.22 |
| block time | ~5.08 s (~17,000 blocks/day) |
| pruning | `custom`, keep-recent 100, interval 10 |
| disk | 128Gi **StandardSSD_LRS** (E10, ~500 IOPS) |
| resources | 500m CPU (no limit) / 2Gi req, 4Gi limit |
| files | 5 YAML + this README |

---

## 1. Deviations from the compose file

The compose file is the specification. Everything below came from it unchanged: the chain
home path `/chain-home`, the `command`, and the container-side ports. Three things changed,
each with evidence.

### The image is not publicly pullable

`cryptocom/chain-main:v8.0.0` returns **HTTP 401 on every tag** — Docker Hub across four
namespaces, and GHCR. The identical probe returns 200 for `library/busybox` and
`library/alpine`, so the method is sound and the repository genuinely is not public. The
Docker CLI agrees: `denied: requested access to the resource is denied`.

It is a **local build tag**. Upstream's own `Dockerfile` opens with:

```
# > docker build -t cryptocom/chain-main .
```

The production VM built that image on the box. Nothing was published, so an AKS pod cannot
pull it.

**Resolution:** upstream's v8.0.0 release binary is **fully statically linked** — verified on
the artifact itself (no `PT_INTERP`, no `PT_DYNAMIC`). It runs on any Linux base image. So
`install-chain-maind` downloads it, verifies SHA256
`61c39edd0216455a2c7a59b429c07d917d92c117cc96ee554484e4f3f8beb0fc` against upstream's
`checksums.txt`, and the node runs it on public `alpine:3.19`.

**No ACR, no build pipeline, no private registry.**

> **If you ever do build the image instead** — upstream's Dockerfile declares
> `ARG NETWORK=testnet`, which overrides the Makefile's own `NETWORK ?= mainnet`. Building
> with defaults produces a **testnet binary that will never join mainnet**. Build with
> `--build-arg NETWORK=mainnet`.

### Ports come from the container side

Compose maps `"1332:26657"`, `"2332:1317"`, `"3332:9090"`. Those left-hand numbers are one
machine's local host mappings. The manifests use the container side — 26657 / 1317 / 9090.

### Grace period

`restart: unless-stopped` gives Docker's default 10-second stop. `chain-maind` needs to flush
LevelDB; `terminationGracePeriodSeconds: 600`.

---

## 2. Where every value came from

| Fact | Value | Source |
|---|---|---|
| chain-id | `crypto-org-chain-mainnet-1` | live RPC `/status` |
| live consensus version | `chain-maind` v8.0.0 | live RPC `/abci_info` |
| block time | 5.08 s | measured over 2,000 blocks |
| image publishable? | no — 401 everywhere | registry probe, control-validated |
| binary linkage | fully static | ELF program headers |
| binary SHA256 | `61c39edd…` | upstream `checksums.txt`, re-hashed locally |
| seeds | 5 entries | cosmos chain-registry, DNS-verified |
| genesis | 181,062 bytes | crypto-org-chain/mainnet |
| snapshot | 8,714,727,213 bytes, `.tar.lz4` | Polkachu `content-length` |
| storage SKU | `StandardSSD_LRS` | this cluster's default; matches `arweave-storage` |

---

## 3. Sizing

**Memory — 2Gi request / 4Gi limit.** A pruned Cronos POS node's steady state measures
~1.5–2.5 GB RSS. 2Gi is the smallest request that leaves headroom for IAVL commit spikes.
**A 1Gi limit will OOMKill.**

**CPU — 500m request, no limit.** The absent limit is deliberate: CFS throttling on a
CometBFT node delays block execution past consensus timeouts and is the classic cause of a
node that "runs but falls behind". 500m is a scheduling reservation; the node bursts into
whatever is idle during catch-up.

**Disk — 128Gi StandardSSD_LRS.**

```
  extracted pruned state      ~18-22 GB   (8.7 GB lz4, ~2-2.5x on Cosmos state)
  chain-maind binary          ~192 MB
  growth                      ~2-5 GB/mo  (17,000 blocks/day, blockstore dominates)
  12-month working set        ~70 GB
```

On StandardSSD_LRS, IOPS are **flat at ~500 regardless of size**, so unlike Premium there is
no performance reason to size up — 128Gi is a pure capacity decision. If the node proves
IOPS-bound during catch-up, `Premium_LRS` scales IOPS with size (P10 500 → P30 5000) at
roughly 2× the cost. Note that StorageClass `parameters` are **immutable**: switching SKU
needs a new class, a new PVC, and a full re-restore.

**Node pool `Standard_D4ads_v5`** (4 vCPU / 16 GiB) leaves roughly **3.5 CPU / 11 GiB**
usable after AKS reservations and kube-system daemonsets. This node fits comfortably. The VM
caps aggregate uncached disk throughput at **~145 MBps** across all attached data disks, so
if snapshot extraction feels slow, that ceiling is the reason rather than the disk.

---

## 4. Egress required

The node fails to start if these are blocked, with errors that read nothing like "firewall":

| Destination | Port | Why |
|---|---|---|
| `github.com`, `objects.githubusercontent.com` | 443 | binary + genesis |
| `polkachu.com`, `snapshots.polkachu.com` | 443 | snapshot discovery + download |
| `dl-cdn.alpinelinux.org` | 443 | `apk add curl lz4` in the restore container |
| `seed-{0,1,2}.crypto.org` | 26656 | p2p seeds |
| `seed.publicnode.com`, `seeds.polkachu.com` | 26656 / 20256 | p2p seeds |
| any peer | 26656 | block gossip (**mandatory**) |

Only the last is needed steady-state. The rest are first-boot only.

---

## 5. Load balancer / port allocation

The rule, stated correctly — and it is the **inverse** of "same ports can share an IP":

```
  AKS runs ONE Azure Standard Load Balancer per cluster.
  Each type: LoadBalancer Service gets its own FRONTEND IP by default.

  Several Services may share ONE frontend IP  <=>  their port sets are DISJOINT.
```

Every Cosmos chain defaults to 26657/1317/9090, so two chains on one IP **collide** — the
Service sticks in `SyncLoadBalancerFailed` and the chain is silently unreachable.

- **dedicated-ip** (active) — one IP per chain, standard ports. One VNet address each.
- **shared-ip** — one IP total, ports offset per chain. See the commented block at the bottom
  of `04-service.yaml`. Only the LB frontend `port` is offset; `targetPort` stays the real
  container port, so the node is never reconfigured.

Keep a port registry with one row per `(ip, port) -> chain`. A collision is a stop.

p2p is deliberately **not** exposed through any LoadBalancer — see the header of
`04-service.yaml`.

---

## 6. Deploy

```bash
kubectl apply -f 00-namespace.yaml
kubectl apply -f 01-storageclass.yaml
kubectl apply -f 02-pvc.yaml            # Pending is CORRECT (WaitForFirstConsumer)
kubectl apply -f 03-statefulset.yaml
kubectl apply -f 04-service.yaml
```

Then follow `RUNBOOK.md` — it has the watch commands and the five acceptance checks that
decide whether this actually worked.

---

## 7. Structure notes

- **Five YAML files.** Init scripts are inlined into `initContainers[].args`; there is no
  ConfigMap. Nothing here needs a mounted config file.
- **One root container.** `restore-snapshot` overrides the pod's non-root default only
  because busybox has no `lz4` and `apk add` needs root. Capabilities are fully dropped, so
  it can neither read, write, nor chmod anything it does not own. Everything else runs as
  1025.
- **Init order is load-bearing.** Restore runs first precisely because it is the
  capability-dropped root container: it cannot chmod files owned by 1025, so it must run
  before anything writes as 1025. That comment is in the manifest — do not "tidy" the order.

## 8. Deliberately not here

- **No archive mode.** This is a pruned node — different disk tier, different pruning
  setting, different restore path.
- **No `.zip` packaging.** The files are the deliverable.
- **No p2p LoadBalancer**, and **no `external_address`.** Setting it without a real public IP
  makes peering worse.
- **No state-sync snapshot serving** (`snapshot-interval = 0`). Producing snapshots for other
  nodes costs disk and CPU; this node is a consumer.
