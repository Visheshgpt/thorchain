# Arweave on AKS — Command Runbook

**Version pinned:** `N.2.9.5.1` (latest stable, released 2026-04-02) · **Namespace:** `arweave` · **Port:** 1984 (everything)

Commands are **PowerShell** (Windows jumpbox, `kubectl` + `curl.exe`) unless a block is marked **Cloud Shell (bash)**.

This is a first deployment — nothing here deletes or migrates an existing node.

---

## 0. Two profiles — pick one

Same image, same namespace, same binary. They differ **only in configuration**.

| | `manifests-validator/` | `manifests-archive/` |
|---|---|---|
| Matches | **`Arweave/v1/docker-setup.md` exactly** | What you originally asked for |
| `storage_modules` | none | `0,unpacked`, `1,unpacked` |
| Weave data stored | **none** | 2 partitions (~4 TB actual) |
| Disks | 1 × 500Gi | 1 × 1Ti + 2 × 4Ti |
| Requests | **500m / 2Gi** (sized to node4) | 2 CPU / 8Gi (needs a new pool) |
| Memory limit | 4Gi | 24Gi |
| Time to useful | 5–15 min | days–weeks per partition |
| Est. disk cost | ~$70/mo | ~$800/mo, ×55 for a full archive |
| Fits current cluster | yes, **on node4 only** | no |

**Start with `manifests-validator/`.** It's a straight port of the node your team already proved out, so if it misbehaves the problem is the AKS wrapper, not Arweave. Switching to the archive profile later is `kubectl apply -f ../manifests-archive/` over the top — the data PVC only grows (500Gi → 1Ti, expandable in place).

Everything below applies to both; where they differ it's called out.

---

## 1. Three findings that shape everything below

**1. There is no official Arweave Docker image.** You told me there was; I had this verified twice, adversarially. There isn't one — not on Docker Hub (the `arweave` and `arweaveteam` namespaces are both empty, `count: 0`), not on GHCR (ArweaveTeam publishes exactly one package and it's npm), and the release CI has zero references to `docker`/`ghcr`/`registry`. The Dockerfiles inside the repo are *build* tooling — their `CMD` clones the repo, runs `rebar3 as prod tar`, and copies a tarball out. They never run a node. ArweaveTeam's own (archived) `testweave-docker` repo pulls a **third party's** image, `lucaarweave/arweave-node:0.0.4`, last pushed 2021. Issue #523 asking how to run mainnet in Docker has been open and unanswered since Feb 2024.

So we build our own — **the Mantra situation, but Mantra's solution doesn't transfer.** Mantra avoided ACR by downloading binaries in init containers using public images. That works for a Go binary; it does not work here. Arweave ships a 52 MB Erlang release with a bundled ERTS and precompiled NIFs (RandomX, VDF, secp256k1, RocksDB) that link against `libcrypto.so.3` and glibc 2.35 — it needs a real Ubuntu 22.04/24.04 filesystem, not Alpine, and pulling 52 MB + extracting 2000 files on every pod start is worse than a one-time image build. **ACR is the right call here.** Section 3.

**2. "Sync from start" isn't a thing on Arweave, and neither is a snapshot.** There is no genesis replay. Joining = downloading the block index (~170 MB, ~198 paged requests) from a trusted peer and verifying the head block. **5–15 minutes from an empty disk, always.** No historical block is re-executed.

There is also **no snapshot service** — official or community. Your Thorchain fallback plan doesn't exist here. But you don't need it: the thing a snapshot would save you (weeks of block replay) doesn't happen on Arweave. *Chunk snapshots can't exist even in principle* — packed chunks are symmetrically encrypted to a specific mining address, so another operator's data is useless to you without a full unpack+repack costing about the same CPU as syncing fresh. The `ar.io` snapshot you may find while searching is a SQLite index for a **different product** (the ar-io gateway) and will not help an Erlang node.

**3. "Archive node" isn't official Arweave terminology.** Grepping the entire docs repo for `archive node` / `archival` / `full node` returns zero hits. The official taxonomy is Solo Miner, Coordinated Miner, Pool Miner, VDF Server, and **Validator**. What you want is a Validator holding `unpacked` storage modules. There's no archive flag — you express it by (a) not setting `mine`, and (b) configuring `storage_module N,unpacked`. Section 2.

---

## 1b. Reconciliation with the team's existing Docker node

`Arweave/v1/docker-setup.md` documents a node a team member already has running on bare metal. It is a genuinely useful artifact and it **corroborates the findings above** — they built their own image from the release tarball on `ubuntu:22.04` (because there was nothing to pull), the tarball extracted flat, `ENTRYPOINT` is `bin/arweave` + `foreground`, ERTS is 14.2.5.11, and their "Issue 3" is a live demonstration that an unknown config key is fatal (`Failed to parse config: unknown: {<<"mining">>,false}`).

**But that node is not an archive node, despite the title.** Its `config.json` has **no `storage_modules` key** — in the whole document `storage_modules` appears only as an API endpoint to *query*. With no storage modules configured, Arweave downloads no weave data at all. Their own sample output shows it:

```json
{ "height": 1972319, "blocks": 208 }
```

208 block headers out of 1.97 million, and zero chunks. That is a healthy, correctly-running **Validator** — it joined, it validates, it serves the API. It just archives nothing. The doc's closing claim that it "will synchronize the full blockchain from genesis" and that sync completes "once `blocks` reaches `height`" is not what the software does; at the default `header_sync_jobs: 1` that counter would take a very long time and still wouldn't imply any weave data.

So: **their setup works and is worth trusting on operational detail — it just answers a different question than the one you asked me.** Adding `storage_modules` is the entire difference between it and an archive.

**What I took from their doc and folded in:**

| Their finding | Change made |
|---|---|
| `--shm-size=4g` | K8s defaults `/dev/shm` to 64 MB. Added a `medium: Memory` emptyDir (`dshm`) |
| `--memory=24g` | Archive profile: limit raised 16Gi → 24Gi. Validator profile: **cut to 4Gi** — 24Gi on a 5810Mi node is fiction and would evict other teams (section 4.1) |
| `--dns 8.8.8.8 --dns 1.1.1.1` (Issue 2) | Added a commented `dnsConfig` block + §9 troubleshooting row |
| Prometheus/ets crashes (Issues 4, 11) | §9 note + 2.9.4.1 downgrade path |
| Hugepages for RandomX (Issue 1) | Documented why `enable randomx_large_pages` must **not** be copied on AKS |
| Corporate firewall blocks 1984 (Issue 8) | Reinforces the internal-LB default in `05-service.yaml` |

**Three things in their doc I did not carry over, deliberately:**

1. **`start_from_latest_state: true`** (their fix for Issue 4). On an empty `data_dir` this exits 1 — permanent CrashLoopBackOff on a fresh PVC. It's a manual recovery step for a populated disk, not a config setting.
2. **`enable: ["graphql"]`** (their Issue 7). Not a real option — the valid `enable` atoms don't include it. Harmless, because unrecognised atoms are silently ignored, but it does nothing. GraphQL is served regardless.
3. **Renaming the wallet to `wallet.json`** and deleting the `wallets/` directory. Arweave looks for `[data_dir]/wallets/arweave_keyfile_[address].json`. Fine while `mine: false` (no key is read), but it will fail the moment anyone enables mining.

*(Minor: their Directory Structure section says `/1Disk73/node-arweave/` while every command uses `/1Disk73/arweave-docker/`. Worth fixing in their doc so the next person doesn't create the wrong tree.)*

---

## 2. What a complete archive actually costs

The expensive part isn't chain state — it's the **weave**, the bulk transaction data. You download exactly the partitions you ask for, and nothing else. Configure zero storage modules and you download essentially nothing.

**Live numbers, measured 2026-08-12** (the docs' own "373 TB / 103 partitions" is stamped November 2025 and already stale):

| | |
|---|---|
| `weave_size` | 389,567,299,756,278 bytes = **389.6 TB** |
| Partition size | 3,600,000,000,000 bytes (3.6 TB) |
| Partitions | **109** (0–108) |
| Reserve per partition | 4 TB (docs: allow ~10% for merkle proofs) |
| **Full replica** | **~436 TB across 109 PVCs** |
| Download time | *"if your download bandwidth is 1 Gbps it will take over 34 days"* |

Azure disk cost for a full replica, 109 × 4 TiB, **disk alone**, before compute or egress:

| Class | ≈ /month |
|---|---|
| `Standard_LRS` (HDD) | **~$18k** |
| `StandardSSD_LRS` | **~$33k** |
| `Premium_LRS` | **~$60k** |

Plus: every chunk you sync needs 1–2 RandomX packing operations, because *"whichever peer you sync the data from will likely return it to you packed to its own address"*. Unpacked syncing is **not** CPU-free.

**So a complete archive is expressible in these manifests but is not a thing this cluster can do.** The manifests ship with **partitions 0 and 1** — literally the start of the weave, 8 TiB — and scale one partition at a time (section 10.1). Decide the target with whoever owns the budget; the manifests don't care whether it's 2 or 109.

**If you want to prove the stack works first:** set `"storage_modules": []`, drop the weave PVCs and mounts, cut requests to `500m`/`4Gi`. That's a headers-only Validator — joins in minutes, costs almost nothing, serves the full HTTP API. It just isn't an archive. Good phase 1.

---

## 3. Build and push the image to ACR

**No Docker needed anywhere** — `az acr build` runs the build in Azure. Build context is `Arweave/v2/image/` (Dockerfile + `vm.args.src`).

**Cloud Shell (bash)** — or any machine with `az`:

```bash
ACR_NAME="<your-acr-name>"

# Upload the build context from your local clone, or recreate the two files there.
cd Arweave/v2/image

az acr build \
  --registry "$ACR_NAME" \
  --image arweave:2.9.5.1 \
  --file Dockerfile \
  .
```

That streams build logs and pushes on success. The Dockerfile downloads the release tarball and **verifies its SHA256** (`dcacd52b…c968f`) — a mismatch fails the build rather than shipping a corrupt binary.

Confirm the tag landed:

```bash
az acr repository show-tags --name "$ACR_NAME" --repository arweave --output table
```

Attach ACR to AKS so the kubelet can pull without an imagePullSecret (once per cluster):

```bash
az aks update --name <AKS_CLUSTER> --resource-group <RG> --attach-acr "$ACR_NAME"
```

Then set the image reference in the manifest:

```powershell
# 04-statefulset.yaml — replace <ACR_NAME> with your registry name
(Get-Content .\manifests\04-statefulset.yaml) `
  -replace '<ACR_NAME>', '<your-acr-name>' |
  Set-Content .\manifests\04-statefulset.yaml

Select-String -Path .\manifests\04-statefulset.yaml -Pattern 'image:'
```

> **Why `az acr build` and not the v1 approach:** v1's Option A told you to `docker build` in Cloud Shell. Cloud Shell has a 5 GB quota and its Docker daemon is unreliable; `az acr build` has neither problem and needs no daemon at all.

---

## 4. Cluster prerequisites

### 4.1 Node pool — check this before anything else

```powershell
kubectl get nodes -o custom-columns='NAME:.metadata.name,VM:.metadata.labels.node\.kubernetes\.io/instance-type,CPU:.status.allocatable.cpu,MEM:.status.allocatable.memory'
kubectl describe nodes | Select-String -Pattern "Allocated resources" -Context 0,8
```

The pod requests **2 CPU / 8 Gi**. Your existing pool is `Standard_D2s_v3` (~1.9 vCPU / 5.79 Gi allocatable, mostly committed) — **this pod will not schedule on it.** Arweave's documented minimum is 8 GB RAM, and sync is RandomX-bound.

Add a dedicated pool:

```bash
az aks nodepool add \
  --resource-group <RG> --cluster-name <AKS_CLUSTER> \
  --name arweave --node-count 1 \
  --node-vm-size Standard_D8s_v3 --node-osdisk-size 128 \
  --labels workload=arweave \
  --node-taints workload=arweave:NoSchedule
```

Then uncomment the `nodeSelector` + `tolerations` block in `04-statefulset.yaml`.

### 4.2 NSG — only if you enable the public LoadBalancer

Default posture is outbound-only (internal LB), which works fine. If you uncomment `arweave-public` in `05-service.yaml`, add the matching NSG rule on the AKS **node subnet** — inbound Allow TCP **1984** from Any. Do both or neither: advertising an address peers can't dial gets you deprioritised and dropped.

---

## 5. Deploy

```powershell
# Pick ONE profile — see section 0. Validator = parity with the team's node.
cd .\Arweave\v2\manifests-validator
# cd .\Arweave\v2\manifests-archive

kubectl apply -f 00-namespace.yaml
kubectl apply -f 01-storageclass.yaml
kubectl apply -f 02-configmap.yaml
kubectl apply -f 03-pvc.yaml
kubectl apply -f 04-statefulset.yaml
kubectl apply -f 05-service.yaml
```

Watch it come up:

```powershell
kubectl get pods -n arweave -w
```

Expected timeline:

| Phase | Duration | What's happening |
|---|---|---|
| `Pending` | 2–6 min | Azure provisions disks (validator: 1 × 500Gi; archive: 1 Ti + 4 Ti + 4 Ti). PVCs stay `Pending` until the pod schedules — that's `WaitForFirstConsumer`, not a fault |
| `ContainerCreating` | 1–3 min | Node pulls the ACR image (~250 MB) |
| `Running 0/1` | 5–15 min | Downloading the block index (~170 MB) and joining. `/recent` returns 503 here — correct |
| `Running 1/1` | ✅ | Joined. Chunk sync starts and runs for days/weeks |

```powershell
kubectl get pvc -n arweave
kubectl logs -n arweave arweave-0 -f
```

Look for the join confirmation:

```powershell
kubectl logs -n arweave arweave-0 | Select-String -Pattern "joined|Joined|block_index|height"
```

---

## 6. Verify

```powershell
# In-cluster (works regardless of LB state)
kubectl port-forward -n arweave pod/arweave-0 1984:1984
```

Second window:

```powershell
# Node status — ALWAYS 200, even before joining.
# height: -1 and current: "not_joined" means it has not joined yet.
curl.exe -s http://localhost:1984/info | ConvertFrom-Json | Format-List

# Readiness gate — 503 {"error":"not_joined"} until joined, 200 after.
curl.exe -s -o NUL -w "%{http_code}`n" http://localhost:1984/recent

# Peers actually connected
curl.exe -s http://localhost:1984/peers | ConvertFrom-Json | Measure-Object

# Storage modules / sync coverage
curl.exe -s http://localhost:1984/data_sync_record | ConvertFrom-Json
```

Via the internal LB, once assigned:

```powershell
$IP = kubectl get svc arweave-internal -n arweave -o jsonpath='{.status.loadBalancer.ingress[0].ip}'
Write-Host "Arweave: http://${IP}:1984"
curl.exe -s "http://${IP}:1984/info"
```

A healthy joined node reports `network: "arweave.N.1"`, a `height` matching the public chain (compare `curl.exe -s https://arweave.net/info`), and a non-zero `peers`.

---

## 7. Monitor sync

> **Validator profile:** only the first of the two below applies. With no `storage_modules` there is no weave sync at all — `/data_sync_record` and `/storage_modules` stay empty and `v2_index_data_size_by_packing` stays at zero. That is correct, not a fault. A low `blocks` count is also expected (`header_sync_jobs` defaults to 1); their bare-metal node sits at 208. **Do not wait for `blocks == height` — it is not a completion condition**, despite what the v1 docker doc's conclusion says.

Two independent things are syncing. Don't conflate them.

**Block headers** — `blocks` vs `height` in `/info`:

```powershell
while ($true) {
  try {
    $i = curl.exe -s http://localhost:1984/info | ConvertFrom-Json
    $pct = if ($i.height -gt 0) { [math]::Round(($i.blocks / $i.height) * 100, 2) } else { 0 }
    Write-Host "$(Get-Date -Format s) | height $($i.height) | headers $($i.blocks) ($pct%) | peers $($i.peers)"
  } catch { Write-Host "$(Get-Date -Format s) | unreachable" }
  Start-Sleep -Seconds 60
}
```

> Don't panic at a low percentage. Most live peers sit at 30–45% of headers, and plenty run fine under 3%. `header_sync_jobs: 10` in the ConfigMap raises this deliberately (the default is 1).

**Weave chunk data** — use Prometheus, **not `du`**. Docs: *"Diskspace measuring tools (e.g. `du`, `ls -l`) will not be able to give an accurate measurement"* — `chunk_storage` uses ~2 GB **sparse** files.

```powershell
curl.exe -s http://localhost:1984/metrics | Select-String -Pattern "v2_index_data_size_by_packing"
```

Expect a partition to **stall short of 3.6 TB** — that's normal. Docs: *"you're never able to download a full 3.6TB partition"* (content policies + never-seeded data). Published estimates: partition 0 ≈ 2.08 TB, partition 1 ≈ 1.91 TB.

Disk and pod pressure:

```powershell
kubectl exec -n arweave arweave-0 -- df -h /data /data/storage_modules/storage_module_0_unpacked /data/storage_modules/storage_module_1_unpacked
kubectl top pod -n arweave
```

---

## 8. Change configuration

**A ConfigMap edit does not restart the pod, and Arweave reads config only at boot.** Always follow with a rollout restart.

```powershell
kubectl edit configmap arweave-config -n arweave
kubectl rollout restart statefulset/arweave -n arweave
kubectl rollout status statefulset/arweave -n arweave --timeout=20m
```

Verify what the pod actually has:

```powershell
kubectl exec -n arweave arweave-0 -- cat /etc/arweave/config.json
```

> **An unknown JSON key is fatal.** `ar_config.erl` ends its parser with `parse_options([Opt|_], _) -> {error, unknown, Opt}` — one typo and the node refuses to start. Treat the schema as closed; don't paste keys from blog posts.
>
> And the **CLI and JSON forms differ**: CLI `peer` → JSON `"peers"`; CLI `storage_module` → JSON `"storage_modules"`; CLI `vdf_server_trusted_peer` → JSON `"vdf_server_trusted_peers"`. Don't copy CLI examples straight in.

**Do not add `start_from_latest_state`.** On a fresh/empty `data_dir` it exits 1 with *"The local state is empty, consider joining the network via the trusted peers"* → CrashLoopBackOff. Normal restarts don't need it; the default path rejoins via trusted peers.

**Do not add `packing_rate`.** It's a no-op in 2.9.5.1 and was **removed** upstream — it would be a fatal unknown key on any future build.

---

## 9. Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `ImagePullBackOff` | ACR not attached, or tag mismatch | `az aks update --attach-acr`; `az acr repository show-tags --name <ACR> --repository arweave -o table` |
| Pod `Pending`, PVCs `Pending` | Not a storage fault — `WaitForFirstConsumer` holds disks until the pod schedules. The pod can't fit | §4.1. `kubectl describe pod arweave-0 -n arweave` → look for `Insufficient cpu/memory` |
| CrashLoop, `{error, unknown, <<"...">>}` | Typo'd config key | §8 |
| CrashLoop, invalid node name / `${ARNAME...}` in error | `RELX_OUT_FILE_PATH` dir missing → `bin/arweave` silently fell back to writing `vm.args` into the read-only release dir and left the variable unexpanded | Confirm the `runtime` emptyDir mounts at `/run/arweave` |
| Read-only filesystem error at boot | Something wrote outside the mounts | Temporarily set `readOnlyRootFilesystem: false`, find the writer, fix properly |
| `peers: 0` after joining | Public LB enabled but NSG closed — or expected, if outbound-only | §4.2. Outbound-only still syncs; check `/peers` is non-empty |
| Internal LB stuck `<pending>` | Public IP quota / LB permissions | `kubectl describe svc arweave-internal -n arweave` |
| Pod OOMKilled | 24 Gi limit hit; `sync_jobs` too high for the box | Lower `sync_jobs`, or raise the limit — docs warn high `sync_jobs` grows RAM via backed-up jobs until OOM |
| Node "Running" but stuck | No liveness probe by design | See below |
| `Peer [IP]:1984 is not available.` repeating, never joins | **DNS** — every `peers` entry is a hostname. Pod can't resolve `*.arweave.xyz` | Uncomment `dnsConfig` in `04-statefulset.yaml`. Test: `kubectl exec -n arweave arweave-0 -- getent hosts peers.arweave.xyz` |
| `randomx_alloc_cache failed` | On bare metal this means hugepages; **on AKS it means memory** | Raise the memory limit. Do **not** add `enable randomx_large_pages` — AKS nodes have no hugepages and you can't sysctl them without a privileged DaemonSet |
| `badarg` in `ets:lookup` for `prometheus_gauge_table`, or `[os_mon] cpu supervisor port (cpu_sup): Erlang has closed` | Known 2.9.5.1 instability — the team hit this on bare metal (their Issues 4 & 11) | See "2.9.5.1 instability" below |

**Diagnostics:**

```powershell
kubectl describe pod arweave-0 -n arweave
kubectl logs -n arweave arweave-0 --tail=200
kubectl logs -n arweave arweave-0 --previous     # after a restart
kubectl get events -n arweave --sort-by=.lastTimestamp | Select-Object -Last 30
```

**There is deliberately no livenessProbe** — same call as thornode v2 here, for a stronger reason. Restarting a node that's merely behind doesn't help it catch up, and each restart risks interrupting a RocksDB compaction on a multi-TB partition. A wedged node still shows up: readiness fails and the pod goes `0/1`. Restart it deliberately:

```powershell
kubectl rollout restart statefulset/arweave -n arweave
```

> Never `kubectl delete pod --force --grace-period=0`. Docs: *"if you kill the node abruptly it can cause rocksdb corruption that can be difficult to recover from. In the worst case you may need to resync and repack a partition."* Resyncing a partition is days. The 600 s `terminationGracePeriodSeconds` exists precisely so a normal delete is safe.

### 2.9.5.1 instability — watch for this

The team hit repeated crashes on 2.9.5.1 on bare metal (`badarg` in `ets:lookup` on `prometheus_gauge_table`; `[os_mon] cpu supervisor port (cpu_sup): Erlang has closed`) and their fix was to downgrade to **2.9.4.1**.

I found independent corroboration in the 2.9.5.1 source: `ar_info:get_info/0` does a hard match on `ets:lookup(ar_header_sync, synced_blocks)` and reads `prometheus_gauge:value(arweave_peer_count)`, either of which can be unpopulated early in startup — so **`/info` can return HTTP 500 during boot.**

The manifests are already shaped defensively against this: readiness probes **`/recent`, not `/info`**, the startup probe on `/info` has a 15-minute tolerance, and there is **no liveness probe**, so a transient `/info` 500 cannot trigger a restart loop. That is the main reason not to "helpfully" add a liveness probe on `/info`.

If it still crash-loops, downgrade — build and roll in one step:

```bash
# Cloud Shell. Get the SHA256 from the 2.9.4.1 release's checksums.txt first.
az acr build --registry "$ACR_NAME" --image arweave:2.9.4.1 \
  --build-arg ARWEAVE_VERSION=2.9.4.1 \
  --build-arg ARWEAVE_TAG=N.2.9.4.1 \
  --build-arg ARWEAVE_ASSET=arweave-2.9.4.1.ubuntu22.x86_64.tar.gz \
  --build-arg ARWEAVE_SHA256=<sha> \
  --file Dockerfile .
```

```powershell
kubectl set image statefulset/arweave arweave=<ACR_NAME>.azurecr.io/arweave:2.9.4.1 -n arweave
kubectl rollout status statefulset/arweave -n arweave --timeout=30m
```

> Weigh it first: 2.9.4.1 is from 2025-04-03, a year older. Also sanity-check `release` in `/info` against a public peer (`curl.exe -s https://arweave.net/info`) — the team's capture showed `release: 89` while live peers report `93`, so confirm you aren't downgrading into a protocol version the network has moved past.

---

## 10. Scale and expand

### 10.1 Add weave partition N

Three edits, all required — the manifests use standalone PVCs (not `volumeClaimTemplates`) specifically so this is a plain `apply` instead of an orphan-delete-and-recreate dance.

1. `02-configmap.yaml` → add `"N,unpacked"` to `storage_modules`
2. `03-pvc.yaml` → copy the template block, name it `arweave-weave-pN`
3. `04-statefulset.yaml` → add **both** the `volumes` entry and the `volumeMounts` entry

```powershell
kubectl apply -f .\manifests\
kubectl rollout restart statefulset/arweave -n arweave
kubectl rollout status statefulset/arweave -n arweave --timeout=30m
```

> **The mount path must match the config string byte for byte.** `"N,unpacked"` → `/data/storage_modules/storage_module_N_unpacked`. Arweave derives the directory name from the string and looks nowhere else. If they drift, the node **doesn't error** — it syncs into the `data_dir` PVC and fills it.
>
> Adding an explicit size changes the name: `"0,3600000000000,unpacked"` → `storage_module_3600000000000_0_unpacked`.

Confirm the modules registered:

```powershell
kubectl exec -n arweave arweave-0 -- ls -la /data/storage_modules/
```

### 10.2 Expand a PVC

`allowVolumeExpansion: true` is set on both storage classes. Grow only — Azure disks cannot shrink.

```powershell
kubectl patch pvc arweave-data -n arweave -p '{\"spec\":{\"resources\":{\"requests\":{\"storage\":\"2Ti\"}}}}'
kubectl get pvc arweave-data -n arweave -w
```

Expand `arweave-data` as headers accumulate. The weave PVCs are fixed-size by design — a partition caps at 3.6 TB.

If you run `pvc-watcher` on this cluster, register **`arweave-data` only**:

```powershell
kubectl edit configmap pvc-watcher-script -n pvc-watcher
# NODE_CONFIG entry: "arweave|arweave-0|/data|arweave-data|80|1024|4096"
```

---

## 11. Tuning knobs

| Knob | Where | Notes |
|---|---|---|
| `sync_jobs` | ConfigMap | Default **100**. Docs: *"Setting sync_jobs to 200 or even 400 is unlikely to cause any issues"* — but too high gets you rate-limited by peers and grows RAM until OOM. Raise once on a dedicated node |
| `header_sync_jobs` | ConfigMap | Default **1**; we ship **10**. This is what backfills historical headers. Not a documented recommendation — tune off `blocks` vs `height` |
| `+A` / `+SDio` | image `vm.args.src` | `+A1024 +SDio1024` = ~2048 OS threads, sized for a many-partition miner. Drop both to `128` on a small node or if you hit a pids limit |
| `+sbwt` | image `vm.args.src` | Already `none` — upstream's own container recommendation. Upstream default `very_long` busy-waits and burns CPU at idle in a cgroup |
| `skuName` | `01-storageclass.yaml` | Weave partitions: `Standard_LRS` (HDD, cheapest) ↔ `Premium_LRS`. Per-PVC decision; Arweave doesn't care |
| memory limit | StatefulSet | 16 Gi. `+MBlmbcs 410629` makes a single allocator carrier ~401 MiB, so RSS is lumpy |

Changing `vm.args.src` requires an image rebuild (§3) and a rollout.

---

## 12. Upgrade

Check for a newer stable release — **skip prereleases** (`N.2.9.6-alpha1/2` exist; `N.2.9.5.1` is the newest stable):

```bash
curl -s https://api.github.com/repos/ArweaveTeam/arweave/releases/latest | jq -r '.tag_name, .published_at'
```

```bash
# Cloud Shell — get the new SHA256 from the release's checksums.txt first
az acr build --registry "$ACR_NAME" --image arweave:<NEW_VERSION> \
  --build-arg ARWEAVE_VERSION=<NEW_VERSION> \
  --build-arg ARWEAVE_TAG=N.<NEW_VERSION> \
  --build-arg ARWEAVE_ASSET=arweave-<NEW_VERSION>.ubuntu22.x86_64.tar.gz \
  --build-arg ARWEAVE_SHA256=<sha> \
  --file Dockerfile .
```

```powershell
kubectl set image statefulset/arweave arweave=<ACR_NAME>.azurecr.io/arweave:<NEW_VERSION> -n arweave
kubectl rollout status statefulset/arweave -n arweave --timeout=30m
curl.exe -s http://localhost:1984/info | ConvertFrom-Json | Select-Object release, height
```

> The `COPY vm.args.src /opt/arweave/releases/${ARWEAVE_VERSION}/vm.args.src` path is version-stamped — the `ARWEAVE_VERSION` build-arg keeps it correct. If a future release also changes upstream's `vm.args.src`, re-diff ours against it rather than carrying ours forward blind.

---

## 13. Reference

**Endpoints** (all on 1984 — one port for API, gossip, chunk sync, VDF, metrics):

| Path | Purpose |
|---|---|
| `/info` | Height, headers stored, peers, release. Always 200 |
| `/recent` | **503 until joined**, 200 after — the readiness gate |
| `/peers` | Connected peers |
| `/data_sync_record` | Synced weave ranges |
| `/storage_modules` | Per-module status — empty until `storage_modules` is configured |
| `/metrics` | Prometheus — `v2_index_data_size_by_packing` is the real sync gauge |
| `/block/current` | Head block |
| `/block/hash/{height}` | Block by height — 404s until that height's header is synced |
| `/graphql` | GraphQL. Enabled by default; returns empty for unsynced ranges |

**Docs** (the site was reorganised; the `/developers/mining/...` URLs you had now redirect):

- https://docs.arweave.org/mining/setup/configuration
- https://docs.arweave.org/mining/overview/node-types
- https://docs.arweave.org/mining/overview/syncing-and-packing
- https://docs.arweave.org/mining/overview/trusted-peers
- https://docs.arweave.org/mining/operations/entrypoint — `bin/start` vs `bin/arweave`
- Docs source (more reliable than the rendered site): https://github.com/ArweaveTeam/docs.arweave.org-info
- **The exhaustive option list is in source, not docs:** `apps/arweave/src/ar.erl` (`show_help/0`) and `apps/arweave/src/ar_config.erl` (`parse_options/2`)
- Peer status: https://status.arweave.xyz/
