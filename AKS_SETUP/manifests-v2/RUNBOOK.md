# RUNBOOK — tear down v1, deploy v2, verify

Every command, in order, for Windows PowerShell on the Azure jumpbox.
Run them top to bottom. Nothing is optional unless marked so.

Set this once per PowerShell session — every later step uses it:

```powershell
$NS  = "thorchain"
$V2  = "C:\Users\usa-vishesgupta\thorchain\manifest-agent\manifests-v2"
```

---

## PHASE 0 — Connect

```powershell
az login
az aks get-credentials --resource-group azrg-cus-coinia-aks-dev --name cuscoiniadevaks --overwrite-existing
kubelogin convert-kubeconfig -l azurecli
kubectl get nodes
kubectl config current-context
```

Stop if `kubectl get nodes` fails. Nothing below will work.

---

## PHASE 1 — Capture v1 state before destroying it

This is your only record of what the broken node looked like, and PHASE 7
needs the disk ID from step 1.4.

```powershell
# 1.1 everything currently in the namespace
kubectl get all -n $NS
kubectl get pvc -n $NS
kubectl get cm -n $NS
kubectl get svc -n $NS -o wide

# 1.2 the storage class you are about to replace
kubectl get sc thorchain-storage -o yaml

# 1.3 final status of the stuck node (ignore errors if the pod is already gone)
kubectl exec -n $NS thornode-0 -c thornode -- curl -s http://localhost:27147/status > "$HOME\thornode-v1-final-status.json"
kubectl logs -n $NS thornode-0 -c thornode --tail=2000 > "$HOME\thornode-v1-final-logs.txt"

# 1.4 RECORD THIS — PHASE 7 deletes the Azure disk using the DISK column
kubectl get pv -o custom-columns=NAME:.metadata.name,STATUS:.status.phase,CAPACITY:.spec.capacity.storage,RECLAIM:.spec.persistentVolumeReclaimPolicy,CLAIM:.spec.claimRef.name,DISK:.spec.csi.volumeHandle
```

Copy the output of 1.4 somewhere you will still have it in an hour.

---

## PHASE 2 — Tear down v1

```powershell
# 2.1 removes sts, pod, pvc, configmap, and ALL THREE v1 services at once
#     (including svc/thornode-p2p, which v2 does not define and which a
#      plain `kubectl apply` would have left behind holding an ILB IP)
kubectl delete namespace $NS --wait=true

# 2.2 StorageClass is cluster-scoped so 2.1 misses it, and its parameters
#     are IMMUTABLE (v1 cachingMode=ReadWrite -> v2 None). `apply` would
#     fail with "updates to parameters are forbidden". Delete it.
kubectl delete storageclass thorchain-storage
```

`2.1` takes 1–3 minutes (v1 had `terminationGracePeriodSeconds: 120`).

Verify the teardown:

```powershell
kubectl get ns $NS                      # expect: NotFound
kubectl get sc                          # expect: no thorchain-storage
kubectl get pv | Select-String thornode # expect: STATUS = Released (disk kept, by design)
```

If the namespace hangs in `Terminating` for more than 5 minutes:

```powershell
kubectl get ns $NS -o json | Select-String finalizer
kubectl api-resources --verbs=list --namespaced -o name | ForEach-Object { kubectl get $_ -n $NS --ignore-not-found }
```

---

## PHASE 3 — Files

```powershell
# 3.1 stop v1 from ever being applied by accident (v1 and v2 share object names)
Rename-Item -Path "C:\Users\usa-vishesgupta\thorchain\manifest-agent\manifests" -NewName "manifests-v1-BROKEN"

# 3.2 confirm the v2 folder is present on the jumpbox with all 6 manifests
cd $V2
Get-ChildItem
```

Expect exactly these: `00-namespace.yaml`, `01-storageclass.yaml`, `02-pvc.yaml`,
`03-configmap-scripts.yaml`, `04-statefulset.yaml`, `05-service.yaml`
(plus `README.md`, `RUNBOOK.md`).

Sanity-check the two things that mattered most, before applying:

```powershell
Select-String -Path 04-statefulset.yaml -Pattern "image:"
Select-String -Path 04-statefulset.yaml -Pattern "PERSISTENT_PEERS"
```

`image:` must show `mainnet-3.19.3` on both containers — not `3.18.1`.

---

## PHASE 4 — Deploy v2

Order matters: namespace before anything namespaced, storageclass before the PVC,
configmap before the StatefulSet that mounts it.

```powershell
cd $V2
kubectl apply -f 00-namespace.yaml
kubectl apply -f 01-storageclass.yaml
kubectl apply -f 02-pvc.yaml
kubectl apply -f 03-configmap-scripts.yaml
kubectl apply -f 04-statefulset.yaml
kubectl apply -f 05-service.yaml
```

Immediate check:

```powershell
kubectl get all -n $NS
kubectl get pvc -n $NS
```

The PVC stays `Pending` until the pod is scheduled — that is correct, the
StorageClass is `WaitForFirstConsumer`.

---

## PHASE 5 — Watch it come up

```powershell
kubectl get pods -n $NS -w
```

Expected, in order (Ctrl+C to stop watching):

```
Init:0/3   init-passwd     ~5 s
Init:1/3   init-keys       ~10 s
Init:2/3   init-snapshot   30-60 min   <- 40 GB download + extract + chmod
Running    thornode
```

If the pod is stuck in `Pending`, `ImagePullBackOff`, or `Init:Error`:

```powershell
kubectl describe pod thornode-0 -n $NS
```

- `Pending` + `Insufficient cpu/memory` → node pool too small. The pod requests
  4 CPU / 8 Gi. Add a node; do **not** shrink the requests.
- `ImagePullBackOff` → egress to `registry.gitlab.com` is blocked.
- `CreateContainerConfigError` → the ConfigMap did not apply; re-run step 4.

Per-container logs:

```powershell
kubectl logs -n $NS thornode-0 -c init-passwd
kubectl logs -n $NS thornode-0 -c init-keys
kubectl logs -n $NS thornode-0 -c init-snapshot -f
```

Track the snapshot download/extract from a second window:

```powershell
kubectl exec -n $NS thornode-0 -c init-snapshot -- df -h /data
```

Once `Running`, follow the node itself:

```powershell
kubectl logs -n $NS thornode-0 -c thornode -f
```

In the first minutes you want to see `render-config` report seeds:

```
found N thorchain seeds
found N p2p seeds
```

A handful of `failed to get node status` errors in that phase are expected —
2 of 95 validators have unhealthy RPC. They are logged at error level but are
not fatal.

---

## PHASE 6 — Verify it is actually syncing

This is the part that was silently broken before. Run all four.

```powershell
# 6.1 PEERS — this was 0 on v1. It must not be 0 now.
kubectl exec -n $NS thornode-0 -c thornode -- curl -s http://localhost:27147/net_info | ConvertFrom-Json | ForEach-Object result | Select-Object n_peers
```

```powershell
# 6.2 HEIGHT MUST INCREASE. Run this; it samples twice, 60 s apart.
$a = (kubectl exec -n $NS thornode-0 -c thornode -- curl -s http://localhost:27147/status | ConvertFrom-Json).result.sync_info.latest_block_height
Start-Sleep -Seconds 60
$b = (kubectl exec -n $NS thornode-0 -c thornode -- curl -s http://localhost:27147/status | ConvertFrom-Json).result.sync_info.latest_block_height
"start=$a  end=$b  blocks_in_60s=$([int]$b - [int]$a)"
```

`blocks_in_60s` must be well above 10 while catching up (it replays far faster
than the 6 s block time). `0` means it is still stuck — go to PHASE 8.

```powershell
# 6.3 HOW FAR BEHIND
$local  = (kubectl exec -n $NS thornode-0 -c thornode -- curl -s http://localhost:27147/status | ConvertFrom-Json).result.sync_info.latest_block_height
$remote = (curl.exe -s "https://gateway.liquify.com/chain/thorchain_rpc/status" | ConvertFrom-Json).result.sync_info.latest_block_height
"local=$local  network=$remote  behind=$([int]$remote - [int]$local)"
```

```powershell
# 6.4 NO FATAL ERRORS
kubectl logs -n $NS thornode-0 -c thornode --tail=4000 | Select-String -Pattern "wrong Block.Header.AppHash|Consensus Failure|panic|OOM|found 0 p2p seeds"
```

`6.4` must return nothing.

Confirm the config that v1 got wrong is now right:

```powershell
kubectl exec -n $NS thornode-0 -c thornode -- sh -c "grep -E '^(seeds|persistent_peers|external_address|pex|addr_book_strict|laddr) ' /data/.thornode/config/config.toml"
kubectl exec -n $NS thornode-0 -c thornode -- sh -c "grep -E '^(pruning|min-retain-blocks|halt-height)' /data/.thornode/config/app.toml"
```

`seeds` must be non-empty, `persistent_peers` must hold the 20 shipped nodes,
`external_address` must be empty, `pruning` must be `"default"`.

Which snapshot it was seeded from, and disk headroom:

```powershell
kubectl exec -n $NS thornode-0 -c thornode -- cat /data/.thornode/.snapshot-restored
kubectl exec -n $NS thornode-0 -c thornode -- df -h /data
kubectl top pod thornode-0 -n $NS --containers
```

Declare success when `catching_up` flips to `false`:

```powershell
kubectl exec -n $NS thornode-0 -c thornode -- curl -s http://localhost:27147/status | ConvertFrom-Json | ForEach-Object result | ForEach-Object sync_info | Select-Object latest_block_height, latest_block_time, catching_up
```

---

## PHASE 7 — Reclaim the stranded v1 disk

**Only after PHASE 6 passes.** Until then the old disk is your rollback.

`reclaimPolicy: Retain` means the v1 Azure managed disk is still billing.

```powershell
# 7.1 find the Released PV (this is the v1 one, not the new v2 one)
kubectl get pv -o custom-columns=NAME:.metadata.name,STATUS:.status.phase,CAPACITY:.spec.capacity.storage,CLAIM:.spec.claimRef.name,DISK:.spec.csi.volumeHandle

# 7.2 delete the Released PV object  (substitute the name from 7.1)
kubectl delete pv <released-pv-name>

# 7.3 delete the underlying Azure disk  (substitute the DISK value from 7.1)
az disk delete --ids "<volumeHandle>" --yes

# 7.4 confirm nothing orphaned is left
kubectl get pv
az disk list --resource-group MC_azrg-cus-coinia-aks-dev_cuscoiniadevaks_centralus --output table
```

The AKS node resource group in 7.4 follows the `MC_<rg>_<cluster>_<region>`
pattern — confirm the real name with `az aks show -g azrg-cus-coinia-aks-dev -n cuscoiniadevaks --query nodeResourceGroup -o tsv`.

---

## PHASE 8 — If it is still not syncing

Run in this order; each command distinguishes a different cause.

```powershell
# 8.1 is it crashing? (v1 had 0 restarts — a nonzero count here is a NEW problem)
kubectl get pod thornode-0 -n $NS -o wide
kubectl describe pod thornode-0 -n $NS | Select-String -Pattern "Restart|Last State|Reason|Exit Code|OOM"
```

```powershell
# 8.2 peers again
kubectl exec -n $NS thornode-0 -c thornode -- curl -s http://localhost:27147/net_info | ConvertFrom-Json | ForEach-Object result | Select-Object n_peers
```

```powershell
# 8.3 egress — probe FOUR validators, never one (2 of 95 return 503 on their own)
kubectl run netcheck -n $NS --rm -it --restart=Never --image=alpine:3.19 -- sh -c "apk add --no-cache curl >/dev/null 2>&1; echo '== proxy env =='; env | grep -i proxy || echo NONE; for ip in 167.235.109.114 173.234.136.17 80.91.65.181 45.63.27.5; do echo -n \"\$ip 27146:\"; nc -z -w 8 \$ip 27146 && echo -n ' OPEN' || echo -n ' BLOCKED'; echo -n ' 27147-http:'; curl -s -o /dev/null -w '%{http_code}' --max-time 12 http://\$ip:27147/status; echo; done; echo -n '443 liquify: '; curl -s -o /dev/null -w '%{http_code}\n' --max-time 15 https://gateway.liquify.com/chain/thorchain_api/thorchain/version"
```

Read it as:

| Observation | Cause | Fix |
|---|---|---|
| 27146 OPEN on all, 27147 mostly 200 | egress is fine | look at 8.1 / 8.4 |
| 27146 BLOCKED on all | NSG / Azure Firewall | open outbound TCP 27146 |
| 27147 non-200 on **all four**, 443 = 200 | HTTP egress restricted | already covered by the shipped `persistent_peers` |
| proxy env vars present | AKS `httpProxyConfig` | Go honours `HTTP_PROXY`; raise with the platform team |

```powershell
# 8.4 fatal patterns
kubectl logs -n $NS thornode-0 -c thornode --tail=4000 | Select-String -Pattern "wrong Block.Header.AppHash|Consensus Failure|panic|OOM|dial|seed|peer"
```

```powershell
# 8.5 version drift — the network moves; your tag does not
curl.exe -s "https://gateway.liquify.com/chain/thorchain_api/thorchain/version"
kubectl get sts thornode -n $NS -o jsonpath="{.spec.template.spec.containers[0].image}"
```

If `current` is ahead of your tag, bump **both** containers and roll:

```powershell
kubectl set image sts/thornode -n $NS thornode=registry.gitlab.com/thorchain/thornode:mainnet-<version> init-keys=registry.gitlab.com/thorchain/thornode:mainnet-<version>
kubectl rollout status sts/thornode -n $NS
```

---

## PHASE 9 — Force a fresh snapshot re-restore (only if the store is corrupt)

```powershell
kubectl exec -n $NS thornode-0 -c thornode -- rm -f /data/.thornode/.snapshot-restored
kubectl delete pod thornode-0 -n $NS
kubectl logs -n $NS thornode-0 -c init-snapshot -f
```

`node_key.json` and `priv_validator_key.json` survive this, so the node keeps
its p2p identity.
