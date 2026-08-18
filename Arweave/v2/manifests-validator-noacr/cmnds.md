# Arweave on AKS — clean rebuild

Run in order. Step 3's wait is not optional — it is the rate-limit cooldown.

---

## 1. Stop the node

```powershell
kubectl scale statefulset arweave-node -n arweave --replicas=0
```

---

## 2. Delete everything

```powershell
kubectl delete namespace arweave
```

Takes ~30 seconds.

```powershell
kubectl get pv | Select-String "arweave"
```

If a `Released` PV appears, delete it (the disk holds no synced data):

```powershell
kubectl delete pv <released-pv-name>
```

The Azure Disk survives (`reclaimPolicy: Retain`). Remove it so it stops billing:

```powershell
az disk list --query "[?diskState=='Unattached'].{name:name,rg:resourceGroup,gb:diskSizeGb}" -o table
```

```powershell
az disk delete --name <disk-name> --resource-group <disk-rg> --yes
```

---

## 3. WAIT 10 MINUTES

Nothing to run. This is the point of the rebuild.

The peers returned `HTTP 503` to this cluster's egress IP because the
crash-looping node had been re-probing ~30 peers every restart for hours, and
all AKS egress shares one public address. Arweave rate limits per IP. The
limit clears on its own once the traffic stops.

Redeploying immediately just re-trips it.

---

## 4. Verify the cooldown worked

```powershell
kubectl create namespace arweave
```

```powershell
kubectl run nettest -n arweave --image=alpine:3.19 --restart=Never --rm -it -- sh -c 'for h in dal-1.east.us.north-america.arweave.xyz den-1.west.us.north-america.arweave.xyz vin-1.east.us.north-america.arweave.xyz bhs-1.ca.north-america.arweave.xyz; do printf "%-46s " $h; wget -T 5 -S -q -O /dev/null http://$h:1984/info 2>&1 | grep -m1 "HTTP/" || echo "NO RESPONSE"; done'
```

**Need `200 OK` on most of them before continuing.**

Still `503`? Wait another 10 minutes and retest. Do not deploy on 503s — you
will just repeat the loop.

---

## 5. Deploy

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

## 6. Watch

```powershell
kubectl get pod -n arweave -w
```

`Pending` -> `Init:0/1` -> `PodInitializing` -> `Running`. Ctrl+C at `Running`.

```powershell
kubectl logs -n arweave arweave-node-0 -c arweave-node -f
```

**Good signs:** RandomX init, hashing benchmark, then `Joined the Arweave
network successfully at ... height <N>`.

**Bad sign:** `Peer ... is not available` repeating -> still rate limited, go
back to step 1.

---

## 7. Confirm

```powershell
kubectl port-forward -n arweave pod/arweave-node-0 1984:1984
```

Second window:

```powershell
curl.exe -s http://localhost:1984/info
```

Expect `"network":"arweave.N.1"`, a `height` near 1.98 million, `peers` above 0.

```powershell
kubectl top pod arweave-node-0 -n arweave
```

Should settle below the 8Gi limit.

---

# If it fails again

Send all three:

```powershell
kubectl get pod -n arweave
```

```powershell
kubectl logs -n arweave arweave-node-0 -c install-arweave
```

```powershell
kubectl logs -n arweave arweave-node-0 -c arweave-node | Select-Object -First 40
```

The **first 40 lines** matter, not the tail — the tail is wreckage from the
supervision tree collapsing, and it hides the original cause.

---

# What changed in the config for this rebuild

- **Peers**: now North-America-first, individually named (verified 200 OK),
  instead of the six global DNS pools. This cluster is in `centralus`, and
  Arweave gives peer probes a **1-second** connect timeout — nearby peers have
  the widest margin.
- **`transaction_blacklist_urls`**: removed. It was the only HTTPS call, and
  `ubuntu:22.04` ships no CA certificates, so Erlang's TLS crashed and took the
  HTTP connection supervisor down with it — which broke the plain-HTTP peer
  connections too.

---

# Expected once running

`blocks` stays low (around 200) and climbs slowly while `height` sits near
1.98 million. **That is correct.** This node stores block headers only, not
the 390 TB of file data, so `blocks` never catches up to `height` and block 0
stays unavailable for a long time. Your team's Docker node behaves the same way.
