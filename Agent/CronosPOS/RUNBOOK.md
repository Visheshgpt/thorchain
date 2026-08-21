# Cronos POS on AKS — runbook (PowerShell)

All commands are **PowerShell**. Two things that bite on Windows:

- `curl` is an alias for `Invoke-WebRequest`. Always write **`curl.exe`** when you want real curl.
- There are no bash heredocs. Multi-line YAML goes into a here-string written out with
  `-Encoding ascii` — `utf8` adds a BOM in PowerShell 5.1 and kubectl rejects it.

```powershell
$NS = "cronospos"
```

---

## 1. Deploy

```powershell
kubectl apply -f 00-namespace.yaml
kubectl apply -f 01-storageclass.yaml
kubectl apply -f 02-pvc.yaml
kubectl apply -f 03-statefulset.yaml
kubectl apply -f 04-service.yaml

kubectl get pvc -n $NS      # Pending is CORRECT until the pod schedules
kubectl get pods -n $NS -w
```

## 2. Watch it come up

Expected: `Init:0/3` → `Init:2/3` → `PodInitializing` → `Running`.
First boot is dominated by the snapshot stream — around 5–10 min at 30 MB/s.

```powershell
kubectl logs -n $NS chain-maind-0 -c restore-snapshot -f      # streamed snapshot (the long one)
kubectl logs -n $NS chain-maind-0 -c install-chain-maind      # binary + genesis + passwd shim
kubectl logs -n $NS chain-maind-0 -c configure                # chain-maind init + TOML patch
kubectl logs -n $NS chain-maind-0 -c chain-maind -f           # the node
```

Milestones, in order:

```
[restore] selected cryptocom_<height>.tar.lz4 (height …, ~8.7 GB)
[restore] extract verified
[restore] DONE at height <height>
[install] sha256 OK  ->  [install] installed chain-maind v8.0.0
[configure] DONE
```

## 3. Acceptance — all five must pass

```powershell
$NS = "cronospos"
function Node-Status {
  kubectl exec -n $NS chain-maind-0 -c chain-maind -- wget -q -O - http://localhost:26657/status |
    ConvertFrom-Json
}

# 1. peers > 0
(kubectl exec -n $NS chain-maind-0 -c chain-maind -- wget -q -O - http://localhost:26657/net_info |
  ConvertFrom-Json).result.n_peers

# 2. height advances (expect >>12 in 60s while catching up; ~12 at tip)
$a = (Node-Status).result.sync_info.latest_block_height
Start-Sleep -Seconds 60
$b = (Node-Status).result.sync_info.latest_block_height
"start=$a  end=$b  blocks_in_60s=$([int]$b - [int]$a)"

# 3. gap to network tip is SHRINKING (run twice, a few minutes apart)
$local  = (Node-Status).result.sync_info.latest_block_height
$remote = (curl.exe -s "https://rpc.mainnet.crypto.org/status" | ConvertFrom-Json).result.sync_info.latest_block_height
"local=$local  network=$remote  behind=$([int]$remote - [int]$local)"

# 4. no fatal patterns
kubectl logs -n $NS chain-maind-0 -c chain-maind --tail=4000 |
  Select-String -Pattern "wrong Block.Header.AppHash|CONSENSUS FAILURE|panic|OOMKilled|no addresses to dial"

# 5. rendered config is what we intended
kubectl exec -n $NS chain-maind-0 -c chain-maind -- sh -c "grep -E '^(laddr|seeds|external_address|prometheus) ' /chain-home/config/config.toml"
kubectl exec -n $NS chain-maind-0 -c chain-maind -- sh -c "grep -E '^(minimum-gas-prices|pruning)' /chain-home/config/app.toml"
kubectl exec -n $NS chain-maind-0 -c chain-maind -- cat /chain-home/.snapshot-restored
kubectl exec -n $NS chain-maind-0 -c chain-maind -- df -h /chain-home
kubectl top pod chain-maind-0 -n $NS --containers
```

**Pass:** peers > 0, `blocks_in_60s` well above 12, `behind` shrinking, check 4 returns nothing,
`pruning = "custom"`, `external_address` empty.

If check 2 passes but check 3 does not, the node is running and **losing ground** — that is an
under-resourced node or a slow disk, not a config bug. Look at `kubectl top pod` first.

## 4. Catch-up expectation

The snapshot lands within ~24 h of tip, so the gap is roughly **17,000 blocks**. A healthy node
replays several hundred blocks/min, so expect **tip within 1–3 hours**.

## 5. Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `tar: ...: Cannot change ownership to uid 1001, gid 1001` | tarball carries the producer's uid; tar as root defaults to `--same-owner` and this container has no `CAP_CHOWN`. **Not cosmetic** — GNU tar exits 2, so the script deletes the extract and retries until it gives up | fixed by `--no-same-owner`. Seeing it means an older manifest is deployed — re-apply and use §7b |
| `install-chain-maind`: `sha256 mismatch` | upstream re-cut the release, or a corrupt transfer | re-hash upstream `checksums.txt`; update `SHA` in `03-statefulset.yaml` |
| `restore-snapshot`: `could not discover a snapshot filename` | Polkachu page markup changed, or egress to `polkachu.com:443` blocked | open the page by hand, then §7 |
| `FATAL: data is populated but has no sentinel` | working as designed — refusing to delete data it did not create | if the data is good: `touch /chain-home/.snapshot-restored`. If it is a dead restore: §7b |
| PVC stuck `Pending` | `WaitForFirstConsumer` — normal until the pod schedules | if it persists, `kubectl describe pod` — it is a CPU/memory fit problem, not a disk one |
| Pod `Pending` | 500m/2Gi does not fit | `kubectl describe node` for allocatable; the node pool is undersized |
| `OOMKilled` | 4Gi limit too low under load | raise the limit — there is no manifest-side fix |
| peers = 0 | egress 26656 blocked | see README §4 |
| height frozen, peers healthy | blocksync reactor stalled | the liveness probe restarts it after 30 min; that is what it is for |
| `EXTERNAL-IP <pending>` on `chain-maind-rpc` | subnet has no free address | use `<node-ip>:30657`; nodePorts are pinned for exactly this |

## 6. Expand the disk

`allowVolumeExpansion: true`, so this is online. Note the PowerShell quoting — the JSON patch
must survive the shell:

```powershell
kubectl patch pvc chain-maind-data -n $NS --type merge -p '{\"spec\":{\"resources\":{\"requests\":{\"storage\":\"256Gi\"}}}}'
```

Never shrink — Azure disks cannot.

## 7. Restart the node

```powershell
kubectl scale statefulset chain-maind -n $NS --replicas=0
kubectl wait --for=delete pod/chain-maind-0 -n $NS --timeout=700s   # the 600s grace matters
kubectl scale statefulset chain-maind -n $NS --replicas=1
```

## 7b. Recover from an interrupted restore

A restore that dies part-way leaves a populated `data/` with no sentinel. From the
`--no-same-owner` fix onward the restore marks its own work with
`/chain-home/.restore-in-progress` and clears it automatically on the next start.

A partial extract that predates that fix has no marker, so the guard refuses it — correctly,
since it cannot tell a half-download from migrated prod data. Place the marker once:

```powershell
$NS = "cronospos"
kubectl scale statefulset chain-maind -n $NS --replicas=0
kubectl wait --for=delete pod/chain-maind-0 -n $NS --timeout=700s

$pod = @'
apiVersion: v1
kind: Pod
metadata:
  name: mark-partial
  namespace: cronospos
spec:
  restartPolicy: Never
  securityContext:
    runAsUser: 0
    runAsGroup: 0
    fsGroup: 1025
  containers:
    - name: mark
      image: alpine:3.19
      command: ["sh","-c","touch /chain-home/.restore-in-progress; ls -la /chain-home"]
      securityContext:
        allowPrivilegeEscalation: false
        capabilities:
          drop: ["ALL"]
      volumeMounts:
        - name: d
          mountPath: /chain-home
  volumes:
    - name: d
      persistentVolumeClaim:
        claimName: chain-maind-data
'@

# -Encoding ascii on purpose: utf8 writes a BOM in PowerShell 5.1 and kubectl rejects it.
$pod | Out-File -Encoding ascii mark-partial.yaml
kubectl apply -f mark-partial.yaml
kubectl wait --for=jsonpath='{.status.phase}'=Succeeded pod/mark-partial -n $NS --timeout=120s
kubectl logs -n $NS mark-partial
kubectl delete pod mark-partial -n $NS
Remove-Item mark-partial.yaml

kubectl apply -f 03-statefulset.yaml          # picks up --no-same-owner
kubectl scale statefulset chain-maind -n $NS --replicas=1
kubectl logs -n $NS chain-maind-0 -c restore-snapshot -f
```

Expect `[restore] partial extract from a previous attempt -- clearing`, then a clean download.

> The `@'` must end its line and `'@` must start at **column 0** with no indentation, or
> PowerShell will not close the here-string. Single quotes (not `@"`) stop PowerShell
> expanding `$` inside the YAML.

## 8. Teardown

```powershell
kubectl delete -f 04-service.yaml -f 03-statefulset.yaml -f 02-pvc.yaml
```

`reclaimPolicy: Retain` means the Azure disk **survives** and the PV goes `Released`. To
reclaim it:

```powershell
kubectl get pv -o custom-columns=NAME:.metadata.name,STATUS:.status.phase,DISK:.spec.csi.volumeHandle
kubectl delete pv <released-pv>
az disk delete --ids "<volumeHandle>" --yes
```
