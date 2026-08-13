# Arweave on AKS — command sheet (PowerShell-safe)

> **Every `az` command here is ONE LINE on purpose.**
> Bash uses `\` to continue a line; PowerShell does not. Pasting a
> backslash-continued command makes PowerShell run `az ... \` on its own and
> then treat each `--flag` line as a separate command, which fails with
> *"is misspelled or was not recognized"*. If you need to wrap a line in
> PowerShell the continuation character is a backtick `` ` `` — but copying
> single lines is safer.

Replace every `<placeholder>` before running.

---

## Variables (set once per session)

```powershell
$ACR_NAME = "coiniadevcr"
$AKS_NAME = "cuscoiniadevaks"
$AKS_RG   = "azrg-cus-coinia-dev"
$VNET_RG  = "AZRG-ALL-ITS-VNET-SYS"
$VNET     = "VNET-CUS-COINIAAKSDEV-10.202.17.192-27"
```

---

## STEP 0 — Prerequisites (check BEFORE deleting anything)

A new node needs one IP from the cluster subnet, and that subnet was already
full when the internal LoadBalancer failed with `SubnetIsFull`. If there is no
free address, Step 2 will fail the same way and this becomes a networking
request, not a Kubernetes one.

```powershell
az network vnet list-usage --resource-group $VNET_RG --name $VNET --output table
```

Compare `CurrentValue` against `Limit` for `default-0`.

If you lack permission on the networking resource group, get the subnet from
the cluster instead and inspect it directly:

```powershell
az aks show --name $AKS_NAME --resource-group $AKS_RG --query "agentPoolProfiles[0].vnetSubnetId" -o tsv
```

```powershell
az network vnet subnet show --ids "<subnet-id-from-above>" --query "{prefix:addressPrefix,used:length(ipConfigurations)}"
```

Quota check:

```powershell
az vm list-usage --location centralus --output table | Select-String "ESv5|DSv6"
```

---

## STEP 1 — Clean slate

```powershell
kubectl delete namespace arweave
```

```powershell
kubectl get pv | Select-String "arweave"
```

`reclaimPolicy: Retain` means the Azure Disk **survives** the namespace
delete. If a `Released` PV appears, remove it and then its disk — nothing has
synced, so there is no data to lose.

```powershell
kubectl delete pv <released-pv-name>
```

```powershell
az disk list --query "[?diskState=='Unattached'].{name:name,rg:resourceGroup,gb:diskSizeGb}" -o table
```

```powershell
az disk delete --name <disk-name> --resource-group <disk-rg> --yes
```

---

## STEP 2 — Create the dedicated node pool

Standard_E2s_v5 = 2 vCPU / 16 GiB (~13 GiB allocatable). Memory-optimized,
which is what this workload needs — no `storage_modules` means no chunk sync,
so no RandomX unpacking, and VDF comes from `vdf-server-3`.

```powershell
az aks nodepool add --resource-group $AKS_RG --cluster-name $AKS_NAME --name arweave --node-count 1 --node-vm-size Standard_E2s_v5 --node-osdisk-size 64 --labels workload=arweave --node-taints workload=arweave:NoSchedule
```

If `Standard_E2s_v5` quota is unavailable, `Standard_E2as_v5` (AMD) is the same
shape and usually cheaper. No manifest change either way.

---

## STEP 3 — Verify the node

```powershell
kubectl get nodes -l workload=arweave
```

```powershell
kubectl describe node <new-node-name> | Select-String -Pattern "Taints|memory:" -Context 0,2
```

Expect the `workload=arweave` label, the `NoSchedule` taint, and ~13 GiB
allocatable.

---

## STEP 4 — Deploy

```powershell
cd C:\Users\usa-vishesgupta\arweave\manifest
```

```powershell
kubectl apply -f 00-namespace.yaml
kubectl apply -f 01-storageclass.yaml
kubectl apply -f 02-pvc.yaml
kubectl apply -f 03-statefulset.yaml
kubectl apply -f 04-service.yaml
```

```powershell
kubectl get pod -n arweave -o wide -w
```

Expect `Pending` → `Init:0/1` → `PodInitializing` → `Running` on the new node.

---

## STEP 5 — Verify it runs

Init container: download, sha256, extract, vm.args patch (~60s on a fresh PVC).

```powershell
kubectl logs -n arweave arweave-node-0 -c install-arweave -f
```

The node itself — watch for `Joined the Arweave network successfully`.

```powershell
kubectl logs -n arweave arweave-node-0 -c arweave-node -f
```

```powershell
kubectl port-forward -n arweave pod/arweave-node-0 1984:1984
```

Second window:

```powershell
curl.exe -s http://localhost:1984/info | ConvertFrom-Json | Format-List
```

```powershell
curl.exe -s -o NUL -w "%{http_code}`n" http://localhost:1984/recent
```

`/recent` returns **503 until joined, 200 after** — that is the readiness gate.

```powershell
kubectl top pod arweave-node-0 -n arweave
```

Should settle below the 8Gi limit.

---

## Monitoring

Header backfill runs newest → oldest. `blocks` climbing is the proof it works.

```powershell
while ($true) { try { $i = curl.exe -s http://localhost:1984/info | ConvertFrom-Json; Write-Host "$(Get-Date -Format s) | height $($i.height) | headers $($i.blocks) | peers $($i.peers)" } catch { Write-Host "$(Get-Date -Format s) | unreachable" }; Start-Sleep -Seconds 60 }
```

> **Expected, not a fault:** `blocks` stays low and climbs slowly (the
> reference node sits at ~208 of 1.98M), and block 0 will not be found for a
> long time. This is a Validator — no `storage_modules`, and `header_sync_jobs`
> defaults to 1. `blocks == height` is **not** a completion condition.

---

## Troubleshooting

```powershell
kubectl describe pod arweave-node-0 -n arweave | Select-String -Pattern "Events" -Context 0,15
```

```powershell
kubectl get events -n arweave --sort-by=.lastTimestamp | Select-Object -Last 25
```

```powershell
kubectl logs -n arweave arweave-node-0 -c arweave-node --previous --tail=100
```

```powershell
kubectl get pod arweave-node-0 -n arweave -o jsonpath='{.status.containerStatuses[0].lastState.terminated.reason}{"\n"}'
```

| Symptom | Cause | Fix |
|---|---|---|
| `Pending`, `didn't match node affinity/selector` | node pool missing or label wrong | Step 2; check `kubectl get nodes -l workload=arweave` |
| `Init:CrashLoopBackOff` | no egress to github.com | `kubectl logs ... -c install-arweave` |
| `OOMKilled` | 8Gi limit hit | raise the limit in `03-statefulset.yaml` (up to ~10Gi on this node) |
| `Evicted` | node-level pressure | should not happen on a dedicated node — check nothing else scheduled there |
| `connection refused` on probes | node still starting | normal for the first few minutes |

---

## Capacity checks (any time)

```powershell
kubectl top nodes
```

```powershell
kubectl describe nodes | Select-String -Pattern "^Name:|Allocated resources" -Context 0,8
```

```powershell
kubectl top pods -A --sort-by=memory | Select-Object -First 15
```

---

## Teardown (after the demo)

```powershell
kubectl delete namespace arweave
```

```powershell
az aks nodepool delete --resource-group $AKS_RG --cluster-name $AKS_NAME --name arweave
```

Then delete the retained disk (see Step 1) or it keeps billing.
