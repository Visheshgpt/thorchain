# Arweave on AKS — run these in order

No node pool needed. The cluster now has 3x Standard_D8s_v6 (32 GiB each)
with plenty free.

---

## 1. Delete the old broken deployment

```powershell
kubectl delete namespace arweave
```

Wait ~30 seconds for it to finish.

---

## 2. Deploy

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

## 3. Watch it start  (~5 minutes)

```powershell
kubectl get pod -n arweave -w
```

Expect: `Pending` → `Init:0/1` → `PodInitializing` → `Running`, then `READY 1/1`.

Press `Ctrl+C` once it shows `1/1`.

---

## 4. Check it is working

```powershell
kubectl port-forward -n arweave pod/arweave-node-0 1984:1984
```

Leave that running. Open a **second** PowerShell window:

```powershell
curl.exe -s http://localhost:1984/info
```

Expect JSON with `"network":"arweave.N.1"`, a `height` around 1.98 million,
and `peers` above 0.

**That means the node is running and synced to the chain tip.**

---

# If something goes wrong

Send me all three outputs:

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

Memory usage (should settle well under the 12Gi limit):

```powershell
kubectl top pod arweave-node-0 -n arweave
```

Which node it landed on:

```powershell
kubectl get pod -n arweave -o wide
```

Node logs:

```powershell
kubectl logs -n arweave arweave-node-0 -c arweave-node -f
```

Remove everything when finished:

```powershell
kubectl delete namespace arweave
```

---

# One thing to expect

In `/info`, the `blocks` number will be small (around 200) and climb slowly,
while `height` sits near 1.98 million.

**This is correct, not a fault.** This node stores block headers only — not
the 390 TB of file data. `blocks` will never catch up to `height`, and block 0
will not be found for a long time. Your team's existing Docker node behaves
exactly the same way.
