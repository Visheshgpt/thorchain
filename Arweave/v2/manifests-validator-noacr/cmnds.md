# Arweave on AKS — run these in order

Every command is one line. Copy one, run it, check the result, move on.

---

## 1. Create the node  (~4 minutes)

```powershell
az aks nodepool add --resource-group azrg-cus-coinia-dev --cluster-name cuscoiniadevaks --name arweave --node-count 1 --node-vm-size Standard_E2s_v5 --node-osdisk-size 64 --labels workload=arweave --node-taints workload=arweave:NoSchedule
```

If this fails, stop and send me the error.

---

## 2. Confirm the node exists

```powershell
kubectl get nodes -l workload=arweave
```

Expect one node, `STATUS: Ready`.

---

## 3. Delete the old broken deployment

```powershell
kubectl delete namespace arweave
```

Takes ~30 seconds.

---

## 4. Deploy

```powershell
cd C:\Users\usa-vishesgupta\arweave\manifest
```

```powershell
kubectl apply -f 00-namespace.yaml
```

```powershell
kubectl apply -f 01-storageclass.yaml
```

```powershell
kubectl apply -f 02-pvc.yaml
```

```powershell
kubectl apply -f 03-statefulset.yaml
```

```powershell
kubectl apply -f 04-service.yaml
```

---

## 5. Watch it start  (~5 minutes)

```powershell
kubectl get pod -n arweave -w
```

Expect: `Pending` → `Init:0/1` → `PodInitializing` → `Running`, and `READY` becomes `1/1`.

Press `Ctrl+C` when it shows `1/1`.

---

## 6. Check it is syncing

```powershell
kubectl port-forward -n arweave pod/arweave-node-0 1984:1984
```

Leave that running. Open a **second** PowerShell window:

```powershell
curl.exe -s http://localhost:1984/info
```

Expect JSON with `"network":"arweave.N.1"`, a `height` around 1.98 million, and `peers` above 0.

**Done — the node is running.**

---

# If something goes wrong

Send me the output of these three:

```powershell
kubectl get pod -n arweave
```

```powershell
kubectl describe pod arweave-node-0 -n arweave
```

```powershell
kubectl logs -n arweave arweave-node-0 -c arweave-node --tail=50
```

---

# Useful later

Watch sync progress:

```powershell
curl.exe -s http://localhost:1984/info
```

`blocks` will be a small number (around 200) and climb slowly. **That is correct** — this node stores block headers only, not the 390 TB of file data. It is not broken.

Memory usage:

```powershell
kubectl top pod arweave-node-0 -n arweave
```

Remove everything when finished:

```powershell
kubectl delete namespace arweave
```

```powershell
az aks nodepool delete --resource-group azrg-cus-coinia-dev --cluster-name cuscoiniadevaks --name arweave
```
