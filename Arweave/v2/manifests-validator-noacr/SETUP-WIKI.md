# Arweave Validator — AKS Setup Wiki

**Namespace:** `arweave` | **Chain:** `arweave.N.1` | **Release:** 2.9.5.1 (`release: 89`)
**Status:** ✅ Running and in sync — `behind = 0`, 8 peers

### Use these for testing

| Service | Endpoint |
|---|---|
| HTTP API / GraphQL | `http://10.202.17.203:31984` |
| In-cluster (stable) | `http://arweave-node-api.arweave.svc.cluster.local:1984` |

```powershell
curl.exe -s --max-time 20 http://10.202.17.203:31984/info
curl.exe -s --max-time 20 http://10.202.17.203:31984/block/current
```

> **`10.202.17.203` is a NODE IP, not a service IP.** It survives reboots but **not**
> node replacement — image upgrades, autoscaler scale-down and VMSS reimage all assign
> new addresses. Re-read with `kubectl get nodes -o wide`. **Any** node IP works, not
> just this one. NodePort `31984` is pinned in `04-service.yaml`, so it is stable.
>
> In-cluster consumers should use the DNS name above, which never changes.

**Port 1984 is everything** — HTTP API, P2P, VDF and Prometheus share one port.

---

## Table of Contents

1. [Architecture](#1-architecture)
2. [Prerequisites](#2-prerequisites)
3. [Manifest Files](#3-manifest-files)
4. [Deployment](#4-deployment)
5. [Issues Encountered & Resolutions](#5-issues-encountered--resolutions)
6. [Verification](#6-verification)
7. [API Endpoints](#7-api-endpoints)
8. [Testing](#8-testing)
9. [Ops Commands](#9-ops-commands)
10. [Upgrade](#10-upgrade)
11. [Teardown](#11-teardown)

---

## 1. Architecture

**Validator** — validates and serves the API. Sets no `storage_modules`, so it stores
block *headers*, not the ~390 TB weave. Mining off.

**No ACR.** Arweave publishes no image and Gatekeeper blocks root, so an init container
downloads the official tarball, verifies its SHA256, and extracts it to the PVC.

```
Pod: arweave-node-0
├── init: install-arweave (alpine:3.19)
│     download -> sha256 verify -> extract to /opt/arweave/2.9.5.1 -> symlink current
│     also stages a CA bundle (ubuntu:22.04 ships none)
└── arweave-node (ubuntu:22.04, no packages installed)
      /opt/arweave/current/bin/arweave foreground config_file /etc/arweave/config.json

PVC arweave-data-pvc (200Gi, StandardSSD, Retain)
├── app/    -> /opt/arweave   (read-only in main)
└── chain/  -> /data          data_dir
```

Two constraints worth knowing before changing anything:

- **`ubuntu:22.04` is not interchangeable.** The bundled NIFs need `libcrypto.so.3`;
  Ubuntu 20.04 and Debian bullseye ship `libcrypto.so.1.1` and will not start.
- **`bin/arweave foreground`, never `bin/start`.** `bin/start` is a bash loop that does
  not `exec`, so as PID 1 it swallows SIGTERM and the pod is always SIGKILLed — which
  the docs warn can corrupt rocksdb.

---

## 2. Prerequisites

### Egress — outbound TCP 1984 must be allowed BY IP

Arweave's peer protocol is **plain HTTP on port 1984**, and the client connects to peers
**by IP** (it resolves hostnames once at startup). This cluster uses
`outboundType: userDefinedRouting`, so egress goes through the central firewall
(`RT-CUS` / `NSG-CUS` in `AZRG-ALL-ITS-CORE-SYS`).

**A hostname/FQDN rule does not work.** Requests carry an IP, match nothing, and are
rejected with a synthetic `HTTP 503`.

**IPs that must be reachable on 1984:**

| IP | What |
|---|---|
| `207.254.22.95` | vdf-server-3.arweave.xyz — **required** |
| `38.22.0.55` | vdf-server-4.arweave.xyz — **required** |
| `40.160.31.17` | vin-1 peer |
| `139.64.164.124` | dal-1 / den-1 / pho-1 / van-1 peer |
| `148.113.226.53` | bhs-1 peer |
| `15.235.228.134`, `15.235.234.1`, `208.69.78.61`, `167.17.70.47`, `57.129.64.209` | pool peers |

Peer IPs rotate and the node discovers more at runtime, so a broad rule is required —
an allow-list goes stale.

**Check before deploying:**

```powershell
kubectl run fwtest -n arweave --image=alpine:3.19 --restart=Never --rm -it -- sh -c "echo ==PEER; wget -T 8 -q -O /dev/null http://40.160.31.17:1984/info && echo OK || echo BLOCKED; echo ==VDF3; wget -T 8 -q -O /dev/null http://207.254.22.95:1984/info && echo OK || echo BLOCKED; echo ==VDF4; wget -T 8 -q -O /dev/null http://38.22.0.55:1984/info && echo OK || echo BLOCKED"
```

All three must be `OK`. See `../FIREWALL-TESTS.md` for the diagnostic that distinguishes
a firewall block from a node fault.

### Capacity

| | Value | Note |
|---|---|---|
| Memory request | 6Gi | Steady state ~7.5Gi |
| Memory limit | **20Gi** | Peak ~15.5Gi building the account tree on first join |
| CPU | 1 request, **no limit** | CPU limits cause CFS throttling in the Erlang VM |
| Node | ≥16 GiB RAM | Runs on `D8s_v6`, ~29Gi allocatable |
| Disk | 200Gi | Actual use is single-digit GB |

```powershell
kubectl top nodes
```

Check **actually free** memory, not free-by-requests — on this cluster one node showed
8Gi requested but 14.5Gi in use.

---

## 3. Manifest Files

| File | Contains |
|---|---|
| `00-namespace.yaml` | Namespace |
| `01-storageclass.yaml` | `arweave-storage` — StandardSSD, `Retain`, expandable |
| `02-pvc.yaml` | `arweave-data-pvc`, 200Gi |
| `03-statefulset.yaml` | ConfigMap (`config.json` + `vm.args.src`) + StatefulSet |
| `04-service.yaml` | Headless `arweave-node` + NodePort `arweave-node-api` (31984) |

### config.json

```json
{
  "peers": [
    "dal-1.east.us.north-america.arweave.xyz",
    "den-1.west.us.north-america.arweave.xyz",
    "vin-1.east.us.north-america.arweave.xyz",
    "pho-1.east.us.north-america.arweave.xyz",
    "bhs-1.ca.north-america.arweave.xyz",
    "van-1.ca.north-america.arweave.xyz",
    "north-america.peers.arweave.xyz",
    "peers.arweave.xyz"
  ],
  "data_dir": "/data",
  "log_dir": "/var/log/arweave",
  "vdf_server_trusted_peers": ["vdf-server-3.arweave.xyz", "vdf-server-4.arweave.xyz"],
  "sync_jobs": 20,
  "max_disk_pool_buffer_mb": 2000,
  "port": 1984,
  "mine": false
}
```

> **An unknown JSON key is FATAL** — `ar_config.erl` returns `{error, unknown, Opt}` and
> the node exits. One typo (`"mining"` instead of `"mine"`) = CrashLoopBackOff.

### vm.args.src

The ConfigMap also carries `vm.args.src`, **mounted over the extracted release's copy**.
It is upstream's file with only the three `+sbwt*` values changed to `none` (upstream's
own container recommendation). The binary allocator settings are upstream's — do not
lower them, see Issue 4.

---

## 4. Deployment

```powershell
cd C:\Users\usa-vishesgupta\arweave\manifest
kubectl apply -f 00-namespace.yaml
kubectl apply -f 01-storageclass.yaml
kubectl apply -f 02-pvc.yaml
kubectl apply -f 03-statefulset.yaml
kubectl apply -f 04-service.yaml
kubectl get pod -n arweave -w
```

`Pending` → `Init:0/1` → `Running 0/1` → `1/1`. `0/1` while joining is correct —
readiness is gated on `/recent`, which 503s until joined. First join ~5 min.

Expected log sequence:

```
Initialising RandomX datasets...                   (~1 min, CPU heavy)
Joining the Arweave network...
Downloaded the block index successfully.
Downloading the wallet tree, chunk 1 ... 126       <- memory peak here
The account tree has been initialized at the block height NNNNNNN.
Joined the Arweave network successfully at the block <hash>, height NNNNNNN.
```

**Then confirm it is FOLLOWING the chain** — joining once is not enough, see §8.4.

---

## 5. Issues Encountered & Resolutions

| # | Symptom | Cause | Fix |
|---|---|---|---|
| 1 | Pod `Evicted`, node destabilised | `D2s_v6` pool (5.8Gi allocatable) far too small | Pool → `D8s_v6` |
| 2 | `pubkey_os_cacerts` crash, all peers "not available" | `ubuntu:22.04` has no CA certs; the one HTTPS call killed `gun_conns_sup` | Removed `transaction_blacklist_urls` |
| 3 | Every peer unreachable, port open | Egress filtered by hostname; Arweave connects by IP | Firewall rule for TCP 1984 |
| 4 | OOMKilled at "wallet tree" on 12/16/20Gi | We had lowered the Erlang binary allocator settings | Upstream `vm.args.src` from ConfigMap |
| 5 | Fixes appeared to do nothing (3x) | StatefulSet rollout waits for a Ready pod; it was crashlooping | `kubectl delete pod` |
| 6 | Joined, then **height frozen forever** | VDF server unreachable | VDF IPs must be reachable |
| 7 | Hundreds of `prometheus_*_table badarg` | Symptom of a collapsed supervision tree | Read the **first 40** log lines |
| 8 | `blocks` ≪ `height` | Normal for a header-only validator | Not a bug |
| 9 | Release download 404 | Asset filename format differs per release | Hardcode `ASSET` |

Four of those need detail.

### Issue 3 — egress filtered by hostname

```
nc -z 40.160.31.17 1984                              -> connects
wget http://40.160.31.17:1984/info                   -> HTTP 503 in 0.02s
wget --header Host:vin-1...arweave.xyz  (same IP)    -> HTTP 200
```

TCP works; only HTTP-by-IP is rejected. The 503 arrives in 0.02s where a real reply
takes ~0.6s — generated locally, never left the network. Other chains here are
unaffected because their p2p protocols are binary and not inspected as HTTP.

**Arweave reports any non-200 as `Peer ... is not available`**, so a firewall rejection
looks identical to a dead peer.

### Issue 4 — the allocator settings we broke

We had lowered these to fit a 4Gi container:

```
+MBsbct  103424 -> 512     (101 MiB -> 512 KiB)
+MBlmbcs 410629 -> 5120    (401 MiB -> 5 MiB)
```

Upstream sets them high deliberately: *"multi-block carriers are more efficient. Since
we have so many 100MiB binary blocks…"* Lowering the threshold forces every binary over
512 KiB into its own mmap'd carrier. The wallet tree is a stream of large binaries.

After reverting: steady **~7.5Gi**, first-join peak **~15.5Gi**.

### Issue 5 — how to actually restart

| Situation | Command |
|---|---|
| `replicas=0` | `kubectl scale statefulset arweave-node -n arweave --replicas=1` |
| Pod healthy | `kubectl rollout restart statefulset/arweave-node -n arweave` |
| Pod crashlooping | `kubectl delete pod arweave-node-0 -n arweave` |

Verify any apply landed — if `AGE` did not reset, the pod was never replaced:

```powershell
kubectl get pod arweave-node-0 -n arweave -o jsonpath="{.metadata.creationTimestamp}{'  '}{range .spec.containers[*]}{.name}={.resources.limits.memory}{' '}{end}{'\n'}"
```

### Issue 6 — frozen height, everything else healthy

`height` stuck at the join block, `blocks` still climbing, `1/1 Ready`, no restarts,
clean logs, all endpoints answering `200`.

Arweave 2.6+ cannot validate a block without VDF. A **non-empty**
`vdf_server_trusted_peers` makes the node refuse to compute VDF itself and wait forever
on a server it cannot reach. Header backfill needs no VDF, which is why it looks healthy.

**Only comparing `height` against the public chain detects this.**

---

## 6. Verification

```powershell
kubectl port-forward -n arweave pod/arweave-node-0 1984:1984
```

Or skip it and use the NodePort — `http://10.202.17.203:31984`.

```powershell
# alive + status
curl.exe -s http://localhost:1984/info | ConvertFrom-Json | Format-List

# the number that matters
"local : " + (curl.exe -s http://localhost:1984/info | ConvertFrom-Json).height
"public: " + (curl.exe -s https://arweave.net/info | ConvertFrom-Json).height

# joined? 200 = yes, 503 = no
curl.exe -s -o NUL -w "%{http_code}`n" http://localhost:1984/recent

# peers (0 = it will never sync)
(curl.exe -s http://localhost:1984/peers | ConvertFrom-Json).Count
```

`/info` fields:

| Field | Meaning |
|---|---|
| `height` | **the block number** — the chain tip |
| `blocks` | headers stored locally — will be far lower, that is correct |
| `peers` | connected peers |
| `current` | head block hash |

---

## 7. API Endpoints

Base: `http://10.202.17.203:31984` or `http://arweave-node-api.arweave.svc.cluster.local:1984`

| Endpoint | Returns |
|---|---|
| `GET /info` | version, height, blocks, peers |
| `GET /recent` | recent blocks — 503 until joined |
| `GET /height` | height as a bare number |
| `GET /peers` | connected peers |
| `GET /metrics` | Prometheus metrics |
| `GET /block/current` | head block |
| `GET /block/height/{h}` | block by height |
| `GET /block/hash/{hash}` | block by hash |
| `GET /tx/{id}` | transaction header |
| `GET /tx/{id}/status` | confirmation status |
| `POST /tx` | submit a transaction |
| `GET /wallet/{addr}/balance` | balance in winston |
| `GET /wallet/{addr}/last_tx` | last transaction id |
| `GET /price/{bytes}` | storage cost in winston |
| `POST /graphql` | GraphQL |

> **Data endpoints (`/{tx_id}` raw data, `/chunk/{offset}`) return 404 here** — this node
> stores no weave data. By design. Use a gateway like `arweave.net` for content.

---

## 8. Testing

### 8.1 Smoke test

```powershell
$b = "http://10.202.17.203:31984"
"info    : " + (curl.exe -s -o NUL -w "%{http_code}" "$b/info")
"recent  : " + (curl.exe -s -o NUL -w "%{http_code}" "$b/recent")
"peers   : " + (curl.exe -s -o NUL -w "%{http_code}" "$b/peers")
"current : " + (curl.exe -s -o NUL -w "%{http_code}" "$b/block/current")
```

All four `200`.

### 8.2 Real chain data

```powershell
$b = "http://10.202.17.203:31984"
$blk = curl.exe -s "$b/block/current" | ConvertFrom-Json
"height    : $($blk.height)"
"indep_hash: $($blk.indep_hash)"
"txs       : $($blk.txs.Count)"

# balance of the miner who produced that block (always a real address)
$addr = $blk.reward_addr
"miner   : $addr"
"balance : " + (curl.exe -s "$b/wallet/$addr/balance") + " winston"

# storage price for 1 MB
"price 1MB: " + (curl.exe -s "$b/price/1048576")
```

### 8.3 GraphQL

```powershell
$q = '{"query":"{ blocks(first:3){ edges { node { id height timestamp } } } }"}'
curl.exe -s -X POST http://10.202.17.203:31984/graphql -H "Content-Type: application/json" -d $q
```

### 8.4 Sync test — the one that matters

A node can answer every endpoint above and still be broken (Issue 6).

```powershell
1..10 | ForEach-Object {
  $l = (curl.exe -s http://10.202.17.203:31984/info | ConvertFrom-Json)
  $p = (curl.exe -s https://arweave.net/info | ConvertFrom-Json)
  "$(Get-Date -Format HH:mm:ss)  local=$($l.height)  public=$($p.height)  behind=$($p.height-$l.height)  peers=$($l.peers)"
  Start-Sleep 30
}
```

| Result | Meaning |
|---|---|
| `behind` 0–3, `local` climbing | **Healthy** |
| `behind` −1 or −2 | Healthy — we saw the block first |
| `local` frozen, `behind` growing | VDF unreachable → Issue 6 |
| `peers` 0 | Egress blocked → Issue 3 |

### 8.5 In-cluster (what a consumer sees)

```powershell
kubectl run apitest -n arweave --image=alpine:3.19 --restart=Never --rm -it -- sh -c "wget -qO- http://arweave-node-api:1984/info"
```

---

## 9. Ops Commands

```powershell
# health
kubectl get pod,svc,endpoints -n arweave -o wide
kubectl top pod arweave-node-0 -n arweave

# logs — FIRST 40 lines when diagnosing, never the tail
kubectl logs -n arweave arweave-node-0 -c arweave-node | Select-Object -First 40
kubectl logs -n arweave arweave-node-0 -c arweave-node -f
kubectl logs -n arweave arweave-node-0 -c install-arweave

# what the node actually loaded
kubectl exec -n arweave arweave-node-0 -c arweave-node -- cat /etc/arweave/config.json
kubectl exec -n arweave arweave-node-0 -c arweave-node -- cat /run/arweave/vm.args

# disk
kubectl exec -n arweave arweave-node-0 -c arweave-node -- df -h /data

# node IPs (the NodePort endpoint changes when nodes are replaced)
kubectl get nodes -o wide

# stop / start
kubectl scale statefulset arweave-node -n arweave --replicas=0
kubectl scale statefulset arweave-node -n arweave --replicas=1
```

---

## 10. Upgrade

The init container is version-stamped: a new version installs alongside the old and the
`current` symlink flips. Rollback is a one-line revert with no re-download.

**1. Get the exact asset name and checksum** — the filename format is not stable between
releases, so read it rather than assuming:

```bash
curl -s https://api.github.com/repos/ArweaveTeam/arweave/releases/tags/N.<VERSION> | grep '"name"'
curl -sL https://github.com/ArweaveTeam/arweave/releases/download/N.<VERSION>/checksums.txt
```

**2. Update four values together** in `03-statefulset.yaml`:

```sh
VERSION="2.9.5.1"
TAG="N.2.9.5.1"
ASSET="arweave-2.9.5.1.ubuntu22.x86_64.tar.gz"
SHA="dcacd52be21cb6b00da438a762fc5b352d1afa410608039d5255901916bc968f"
```

Plus the version-stamped `vm.args.src` mount path:

```yaml
mountPath: /opt/arweave/<VERSION>/releases/<VERSION>/vm.args.src
```

**3. Apply and replace the pod**, then verify:

```powershell
kubectl apply -f 03-statefulset.yaml
kubectl rollout restart statefulset/arweave-node -n arweave
kubectl exec -n arweave arweave-node-0 -c arweave-node -- readlink /opt/arweave/current
```

Re-run §8.4. The node rebuilds the account tree, so expect the ~15.5Gi spike again.

---

## 11. Teardown

```powershell
kubectl delete namespace arweave
kubectl get pv | Select-String "arweave"
kubectl delete pv <released-pv-name>
```

**The Azure Disk survives** (`reclaimPolicy: Retain`) and keeps billing:

```powershell
az disk list --query "[?diskState=='Unattached'].{name:name,rg:resourceGroup,gb:diskSizeGb}" -o table
az disk delete --name <disk-name> --resource-group <disk-rg> --yes
```
