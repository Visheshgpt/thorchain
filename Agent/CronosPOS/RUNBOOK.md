# Cronos POS on AKS — runbook

`NS=cronospos`, pod `chain-maind-0`.

---

## 1. Deploy

```bash
NS=cronospos
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
First boot is **20–50 min**, almost all of it the streamed snapshot.

```bash
kubectl logs -n $NS chain-maind-0 -c restore-snapshot -f     # streamed snapshot (the long one)
kubectl logs -n $NS chain-maind-0 -c install-chain-maind    # binary + genesis + passwd shim
kubectl logs -n $NS chain-maind-0 -c configure              # chain-maind init + TOML patch
kubectl logs -n $NS chain-maind-0 -c chain-maind -f         # the node
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

```bash
NS=cronospos
X="kubectl exec -n $NS chain-maind-0 -c chain-maind --"

# 1. peers > 0
$X wget -qO- http://localhost:26657/net_info | sed -n 's/.*"n_peers": *"\([0-9]*\)".*/peers=\1/p'

# 2. height advances (expect >>12 in 60s while catching up; ~12 at tip)
A=$($X wget -qO- http://localhost:26657/status | sed -n 's/.*"latest_block_height": *"\([0-9]*\)".*/\1/p')
sleep 60
B=$($X wget -qO- http://localhost:26657/status | sed -n 's/.*"latest_block_height": *"\([0-9]*\)".*/\1/p')
echo "start=$A end=$B blocks_in_60s=$((B-A))"

# 3. gap to network tip is SHRINKING (run twice, a few minutes apart)
L=$($X wget -qO- http://localhost:26657/status | sed -n 's/.*"latest_block_height": *"\([0-9]*\)".*/\1/p')
R=$(curl -s https://rpc.mainnet.crypto.org/status | sed -n 's/.*"latest_block_height": *"\([0-9]*\)".*/\1/p')
echo "local=$L network=$R behind=$((R-L))"

# 4. no fatal patterns
kubectl logs -n $NS chain-maind-0 -c chain-maind --tail=4000 \
  | grep -Ei "wrong Block.Header.AppHash|CONSENSUS FAILURE|panic|OOMKilled|no addresses to dial" || echo "clean"

# 5. rendered config is what we intended
$X grep -E '^(laddr|seeds|external_address|prometheus) ' /chain-home/config/config.toml
$X grep -E '^(minimum-gas-prices|pruning)' /chain-home/config/app.toml
$X cat /chain-home/.snapshot-restored
$X df -h /chain-home
```

**Pass:** peers > 0, `blocks_in_60s` well above 12, `behind` shrinking, no fatal matches,
`pruning = "custom"`, `external_address` empty.

If check 2 passes but check 3 does not, the node is running and **losing ground** — that is
an under-resourced node or a slow disk, not a config bug. Check `kubectl top pod` first.

## 4. Catch-up expectation

The snapshot lands within ~24 h of tip, so the gap is roughly **17,000 blocks**. A healthy
node replays several hundred blocks/min, so expect **tip within 1–3 hours**.

## 5. Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `install-chain-maind`: `sha256 mismatch` | upstream re-cut the release, or a corrupt transfer | re-hash upstream `checksums.txt`; update `SHA` in `03-statefulset.yaml` |
| `restore-snapshot`: `tar: ...: Cannot change ownership to uid 1001, gid 1001` | the tarball carries the producer's uid; tar as root defaults to `--same-owner` and this container has no `CAP_CHOWN`. **Not cosmetic** — GNU tar exits 2, so the script deletes the extract and retries until it gives up | fixed by `--no-same-owner` in `03-statefulset.yaml`. If you see it, you are running an older copy of the manifest — re-apply and restart |
| `restore-snapshot`: `could not discover a snapshot filename` | Polkachu page markup changed, or egress to `polkachu.com:443` blocked | open the page by hand, then §7 |
| `FATAL: data is populated but has no sentinel` | working as designed — refusing to delete data it did not create | if the data is good: `touch /chain-home/.snapshot-restored`. If not, delete the PVC. |
| PVC stuck `Pending` | `WaitForFirstConsumer` — normal until the pod schedules | if it persists, `kubectl describe pod` — it is a CPU/memory fit problem, not a disk one |
| Pod `Pending` | 500m/2Gi does not fit | `kubectl describe node` for allocatable; the node pool is undersized |
| `OOMKilled` | 4Gi limit too low under load | raise the limit — there is no manifest-side fix |
| peers = 0 | egress 26656 blocked | see README §4 |
| height frozen, peers healthy | blocksync reactor stalled | the liveness probe restarts it after 30 min; that is what it is for |
| `EXTERNAL-IP <pending>` on `chain-maind-rpc` | subnet has no free address | use `<node-ip>:30657`; nodePorts are pinned for exactly this |

## 6. Expand the disk

`allowVolumeExpansion: true`, so this is online:

```bash
kubectl patch pvc chain-maind-data -n cronospos \
  -p '{"spec":{"resources":{"requests":{"storage":"256Gi"}}}}'
```

Never shrink — Azure disks cannot.

## 7. Re-restore the snapshot

```bash
kubectl scale statefulset chain-maind -n cronospos --replicas=0
# wait for full termination -- the 600s grace period matters
kubectl scale statefulset chain-maind -n cronospos --replicas=1
```

To force a fresh snapshot, delete the sentinel **and** the data dir first — from a throwaway
pod mounting the PVC, not from the node pod. Deleting the sentinel alone is not enough:
`bootstrap.sh` will refuse to overwrite a populated data dir, by design.

## 7b. Recover from an interrupted restore

A restore that dies part-way leaves a populated `data/` with no sentinel. From the
`--no-same-owner` fix onward the restore marks its own work with
`/chain-home/.restore-in-progress` and clears it automatically on the next start.

If the partial extract predates that fix it has no marker, so the guard refuses it (correctly
— it cannot tell your half-download from migrated prod data). Place the marker once:

```bash
NS=cronospos
kubectl scale statefulset chain-maind -n $NS --replicas=0
kubectl wait --for=delete pod/chain-maind-0 -n $NS --timeout=700s

kubectl apply -f - <<'EOF'
apiVersion: v1
kind: Pod
metadata: { name: mark-partial, namespace: cronospos }
spec:
  restartPolicy: Never
  securityContext: { runAsUser: 0, runAsGroup: 0, fsGroup: 1025 }
  containers:
    - name: mark
      image: alpine:3.19
      command: ["sh","-c","touch /chain-home/.restore-in-progress && ls -la /chain-home"]
      securityContext:
        allowPrivilegeEscalation: false
        capabilities: { drop: ["ALL"] }
      volumeMounts: [{ name: d, mountPath: /chain-home }]
  volumes:
    - name: d
      persistentVolumeClaim: { claimName: chain-maind-data }
EOF

kubectl logs -n $NS mark-partial
kubectl delete pod mark-partial -n $NS

kubectl apply -f 03-statefulset.yaml          # picks up --no-same-owner
kubectl scale statefulset chain-maind -n $NS --replicas=1
kubectl logs -n $NS chain-maind-0 -c restore-snapshot -f
```

Expect `[restore] partial extract from a previous attempt -- clearing`, then a clean
download.

## 8. Teardown

```bash
kubectl delete -f 04-service.yaml -f 03-statefulset.yaml \
                -f 02-pvc.yaml
```

`reclaimPolicy: Retain` means the Azure disk **survives** and the PV goes `Released`. To
reclaim it:

```bash
kubectl get pv -o custom-columns=NAME:.metadata.name,STATUS:.status.phase,DISK:.spec.csi.volumeHandle
kubectl delete pv <released-pv>
az disk delete --ids "<volumeHandle>" --yes
```
