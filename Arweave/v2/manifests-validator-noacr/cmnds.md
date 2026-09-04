# Arweave Validator on AKS — working configuration

**Status:** running, no workarounds. The firewall team opened outbound TCP 1984,
so the nginx proxy has been removed and the node uses standard peer hostnames.

```
network : arweave.N.1     release : 89 (2.9.5.1)
memory  : ~7.5Gi steady   peak    : ~15.5Gi during first join
```

---

## What this is

A **Validator** — the official Arweave term. It joins the network, validates
blocks and transactions, and serves the full HTTP API and GraphQL. It stores
**no weave data** (no `storage_modules`), so it holds block headers, not the
390 TB of file data.

Same node type as the team's bare-metal Docker node in
`Arweave/v1/docker-setup.md`. Mining is disabled.

**No ACR required.** An init container downloads the official release tarball,
verifies its SHA256, and extracts it to the PVC. Both images are public
(`alpine:3.19`, `ubuntu:22.04`).

---

## Architecture

```
Pod: arweave-node-0
├── init: install-arweave (alpine:3.19)
│     downloads + sha256-verifies + extracts Arweave 2.9.5.1 to the PVC
│     stages a CA bundle (ubuntu:22.04 ships none)
│
└── arweave-node (ubuntu:22.04, no packages installed)
      /opt/arweave/current/bin/arweave foreground config_file ...

PVC arweave-data-pvc (200Gi)
├── app/    -> /opt/arweave   binary (read-only in main container)
└── chain/  -> /data          data_dir
```

---

## The five root causes (in the order they were found)

Each one masked the next, which is why this took so long.

### 1. Node memory — the pod was evicted, not OOMKilled

The original 5x `Standard_D2s_v6` pool (8 GiB, ~5.8 GiB allocatable) ran at
~90% memory. The pod reached 3.6 GiB and **took the node down with it**:

```
Evicted: node was low on resource: memory. available: 2240Ki
         container was using 3039124Ki, request is 2Gi
```

Eviction is worse than an OOMKill — it destroys the pod and threatens other
teams' workloads. Fixed when the pool was replaced with 3x `Standard_D8s_v6`
(~29 GiB allocatable each).

### 2. Missing CA certificates

```
{badmatch,undefined} {pubkey_os_cacerts,get,0,...} {gun,initial_tls_handshake,3,...}
```

`ubuntu:22.04` ships every shared library Arweave needs but **no
ca-certificates**. Arweave's only HTTPS call is `transaction_blacklist_urls`.
The TLS failure killed `gun_conns_sup`, which owns **every** HTTP connection
including the plain-HTTP peer ones — so all peers appeared unreachable.

Fixed by removing `transaction_blacklist_urls`, plus staging a CA bundle from
the alpine init container as belt-and-braces.

### 3. FQDN-filtered egress — the big one

Egress is filtered by hostname. Proven from a pod, same IP and port:

```
wget http://40.160.31.17:1984/info                       -> HTTP 503
wget --header Host:vin-1...arweave.xyz  (same IP)        -> HTTP 200
```

**Arweave connects to peers by IP** — it resolves hostnames once at startup,
then dials the addresses. Every request carries `Host: <ip>`, matches no FQDN
rule, and is denied. Its HTTP client offers no Host override.

**RESOLVED.** The firewall team opened outbound TCP 1984. The nginx proxy that
worked around this has been removed; the config uses real hostnames again.

### 4. Erlang allocator tuning — self-inflicted

An earlier version of this deployment lowered the binary allocator settings to
fit a 4 GiB container:

```
+MBsbct  103424 -> 512     (101 MiB -> 512 KiB)
+MBlmbcs 410629 -> 5120    (401 MiB -> 5 MiB)
```

Upstream's own comment explains why they are high: *"multi-block carriers are
more efficient… we have so many 100MiB binary blocks."* Lowering the threshold
forces every binary over 512 KiB into its own mmap'd carrier. The wallet tree
is a stream of large binaries, so it OOMKilled at chunk 7-9 every time.

Fixed by shipping `vm.args.src` **from the ConfigMap**, mounted over the
extracted release copy — upstream values, with only `+sbwt none` changed
(upstream's own container recommendation).

### 5. StatefulSet rollout stalls on an unready pod

`kubectl apply` + `kubectl rollout restart` did **not** replace the pod,
because a StatefulSet rolling update waits for the current pod to become Ready
— and it was in CrashLoopBackOff. Three separate fixes appeared "not to work"
while the old pod kept running the old spec.

**On a StatefulSet whose pod is unhealthy, use:**

```powershell
kubectl delete pod arweave-node-0 -n arweave
```

---

## Deploy from scratch

```powershell
cd C:\Users\usa-vishesgupta\arweave\manifest
kubectl apply -f 00-namespace.yaml
kubectl apply -f 01-storageclass.yaml
kubectl apply -f 02-pvc.yaml
kubectl apply -f 03-statefulset.yaml
kubectl apply -f 04-service.yaml
```

```powershell
kubectl get pod -n arweave -w
```

`Pending` -> `Init:0/1` -> `Running 0/1` -> `1/1`. First join takes ~5 min
(download + wallet tree). Ctrl+C at `1/1`.

### After ANY change to 03-statefulset.yaml

```powershell
kubectl apply -f 03-statefulset.yaml
kubectl rollout restart statefulset/arweave-node -n arweave
```

Then **verify it actually landed** — this is the check that would have saved
several hours:

```powershell
kubectl get pod arweave-node-0 -n arweave -o jsonpath="{.metadata.creationTimestamp}{'  '}{range .spec.containers[?(@.name=='arweave-node')]}{.resources.limits.memory}{end}{'\n'}"
```

If the timestamp is old, the rollout stalled — `kubectl delete pod arweave-node-0 -n arweave`.

---

## Monitoring

### Block number / sync status

```powershell
kubectl port-forward -n arweave pod/arweave-node-0 1984:1984
```

Second window:

```powershell
curl.exe -s http://localhost:1984/info | ConvertFrom-Json | Format-List
```

| Field | Meaning |
|---|---|
| `height` | **the current block number** — the chain tip this node has |
| `blocks` | how many block HEADERS this node has stored |
| `peers` | connected peers (dozens once healthy) |
| `current` | hash of the head block |

Just the block number:

```powershell
(curl.exe -s http://localhost:1984/info | ConvertFrom-Json).height
```

Compare against the public network to confirm it is keeping up:

```powershell
"local : " + (curl.exe -s http://localhost:1984/info | ConvertFrom-Json).height
"public: " + (curl.exe -s https://arweave.net/info | ConvertFrom-Json).height
```

Those should stay within a few blocks of each other. Arweave produces a block
roughly every 2 minutes.

Continuous watch:

```powershell
while ($true) { $i = curl.exe -s http://localhost:1984/info | ConvertFrom-Json; Write-Host "$(Get-Date -Format s) | height $($i.height) | headers $($i.blocks) | peers $($i.peers)"; Start-Sleep -Seconds 60 }
```

> **`blocks` will stay far below `height`, and that is correct.** This node
> stores headers only, and `header_sync_jobs` defaults to 1, so the backfill
> runs slowly newest-to-oldest. The reference bare-metal node sits at ~208.
> `blocks == height` is **not** a completion condition, and block 0 will not be
> queryable for a long time.

### Other endpoints

```powershell
curl.exe -s http://localhost:1984/block/current      # head block
curl.exe -s http://localhost:1984/peers              # connected peers
curl.exe -s -o NUL -w "%{http_code}`n" http://localhost:1984/recent   # 200 = joined
```

### Resources

```powershell
kubectl top pod arweave-node-0 -n arweave
kubectl get pod arweave-node-0 -n arweave -o wide
```

---

## Resolved: egress

The firewall team opened outbound TCP 1984. Verified from a pod against IPs
that were never in the original request, so the rule is broad rather than a
handful of allow-listed addresses:

```
148.113.226.53   OK      15.235.228.134  OK
208.69.78.61     OK      167.17.70.47    OK
207.254.22.95    OK  (vdf-server-3)
38.22.0.55       OK  (vdf-server-4)
```

The nginx `peer-proxy` sidecar, its `nginx.conf` ConfigMap key, and the
`proxy-tmp` volume are all gone. `peers` and `vdf_server_trusted_peers` now use
real hostnames.

**If peer connectivity ever breaks again**, re-run the three-test check in
`../FIREWALL-TESTS.md` before assuming it is an Arweave problem — that
distinguishes a network block from a node fault in about a minute.

## Teardown

```powershell
kubectl delete namespace arweave
kubectl get pv | Select-String "arweave"
kubectl delete pv <name>
```

`reclaimPolicy: Retain` means the Azure Disk survives — delete it or it keeps
billing:

```powershell
az disk list --query "[?diskState=='Unattached'].{name:name,rg:resourceGroup,gb:diskSizeGb}" -o table
az disk delete --name <disk-name> --resource-group <disk-rg> --yes
```
