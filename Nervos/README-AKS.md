# Nervos CKB on AKS — why it wasn't syncing, and what changed

Cluster `cuscoiniadevaks` / RG `azrg-cus-coinia-aks-dev`, namespace `nervos`.
Commands below are PowerShell-safe.

---

## 1. The diagnosis

Your log is the whole story:

```
INFO ckb_launcher  chain genesis hash: 0x92b197aa1fba...   <- mainnet, correct
INFO ckb_network::network  Listen on address: /ip4/0.0.0.0/tcp/8115
INFO ckb_rpc::server  Listen HTTP RPCServer on address: 0.0.0.0:8114
INFO ckb_bin::subcommand::run  CKB service started ...
INFO ckb_stop_handler::stop_register  Waiting exit signal...
```

Then nothing. Not one dial, not one header, not one block. That is not a
crashing node or a misconfigured one — it is a **healthy node with zero
peers**. Two independent causes, and you need both fixed.

### Cause A — the config file was never mounted

In the old `manifest/03-statefulset.yaml`, both halves of the config mount
were commented out:

```yaml
# TODO: Uncomment after creating the ConfigMap (see README.md Step 0):
# - name: ckb-config
#   mountPath: /var/lib/ckb/ckb.toml
```

No ConfigMap ever existed. So on first boot the image entrypoint ran
`ckb init --chain mainnet` and **ckb-0 has been running a stock config
nobody in this repo wrote** — none of your bootnodes, none of your tuning,
no RichIndexer. Your `ckb.toml` was never in play on AKS at all.

Confirm it yourself in 10 seconds:

```powershell
kubectl exec -n nervos ckb-0 -c ckb -- cat /var/lib/ckb/ckb.toml
```

If that does not show the 15 bootnodes, it is the generated file.

### Cause B — egress is blocked. CONFIRMED, and it is now the blocker.

You already found this on Docker: the VM only started syncing once outbound
8114 was opened. The same is true here, and the test in section 2 has now
**confirmed** it — every CKB bootnode is unreachable from a pod in `nervos`
on both 8114 and 8115.

This cluster is documented as restricting exactly this:

| Source | What it says |
|---|---|
| `AKS_SETUP/SETUP-WIKI.md:83` | "**AKS blocks outbound TCP on random ports.**" |
| `AKS_SETUP/manifests/03-statefulset.yaml:195` | "Port 27147 direct to validator IPs is blocked by AKS NSG — do NOT use." |

THORChain hit the identical wall on 27147 and had to route genesis over 443
instead. **CKB has no such escape hatch** — P2P is a continuous binary
protocol to many peers, not a one-time file fetch.

**No manifest change fixes this.** Section 2 works out which *kind* of block
it is and gives you the exact wording to request.

### Your port observation is correct, and now it's written down

You were right, and it is the thing most likely to confuse the next person:

```
8114 = JSON-RPC on OUR node        (rpc.listen_address)
8115 = P2P listener on OUR node    (network.listen_addresses)

but the official mainnet BOOTNODES publish THEIR P2P on 8114.

=> we LISTEN on 8115 and DIAL OUT to 8114.
=> the firewall rule that matters is OUTBOUND TCP 8114.
```

That is backwards from every Cosmos chain in this repo. It is now documented
at the top of `ckb.toml`, `03-configmap-ckb.yaml` and `03-statefulset.yaml`.

### And the `ckb.toml` in this repo was not the working one

`Nervos/ckb.toml` was **not** the config running on the VM. Compared against
the proven one in `docker-prod-guide.md`, it had **1 bootnode instead of 15**
— and that one was on `tcp/8115`, where the real bootnodes are on `tcp/8114`.
So even mounting it as-is would have left you at zero peers. It also carried
`[store]` and `connect_outbound_count` keys that are not CKB config fields.

It has been replaced with the proven config. The old one is kept as
`ckb.toml.orig-broken` for reference.

---

## 2. Egress — SOLVED: outbound TCP 8114 is blocked, and nothing else

### 2a. The finding

Three tests, each holding one variable fixed:

| Test | Result | Rules out |
|---|---|---|
| **A** — `portquiz.net` (one name, many ports) | 443 OPEN · 80 OPEN · 26656 OPEN · **8115 OPEN** · **8114 BLOCKED** | a broad port block |
| **B** — same host+port, name vs raw IP | `34.143.175.3:26656` OPEN · `35.180.139.74:443` OPEN | FQDN/DNS-proxy filtering |
| **C** — same bootnode IP, two ports | `:443` OPEN · `:8114` BLOCKED (×3 IPs) | the remote host being down |

**Outbound TCP 8114 is blocked. Every other port tested is open, including
8115. Raw IPs are fine.** That matches the VM exactly — opening outbound 8114
there is what made it sync.

Supporting facts, verified from an unrestricted network:

- All 15 bootnodes are **up and answering on 8114** — the list is not stale.
  Expect this to be the network team's first question.
- The bootnodes listen on **8114 only**; `:8115` is closed on all of them.

Three earlier verdicts are withdrawn: `1.1.1.1:443 BLOCKED` did not mean
general egress failure (`8.8.8.8:443` is open); `168.63.129.16:80 BLOCKED` is
normal AKS pod hardening against IMDS credential theft, not a NetworkPolicy;
and the v3 pod's `node:` line printed the *pod* name, since `cat /etc/hostname`
in a container is not the node — the `nodeName` pin itself is a hard
assignment and did place it on `...vmss00000p`.

### 2b. The ask — one port

> Please allow **outbound TCP 8114** from the AKS node subnet
> `10.202.17.192/27` (cluster `cuscoiniadevaks`, RG `azrg-cus-coinia-aks-dev`)
> to the internet.
>
> Nervos CKB peer-to-peer dials its bootnodes on TCP 8114. We have isolated
> this to that single port: from this subnet, outbound TCP 80, 443, 8115 and
> 26656 all succeed, to both named hosts and raw IPs, while 8114 is dropped
> (8s timeout) to every destination tested — including the *same IP* that
> accepts a connection on 443. The destination hosts are confirmed up and
> answering on 8114 from an unrestricted network.
>
> Without it the node holds 0 peers and never syncs. The equivalent rule on
> the standalone VM is what made that node start syncing.

**8115 does not need to be requested — it is already open.**

Useful context to attach:

```powershell
az aks show -g azrg-cus-coinia-aks-dev -n cuscoiniadevaks `
  --query "networkProfile.{outboundType:outboundType,plugin:networkPlugin,policy:networkPolicy}" -o table
```

### 2c. Workaround that needs no firewall change at all

You may not have to wait for the ticket.

**CKB's default P2P port is 8115** — that is why our own node listens there.
Only the official *bootnodes* use 8114. Most ordinary nodes on the network
are on 8115, and **8115 is already open outbound from this cluster.**

So the node does not need 8114 to sync. It needs *one reachable peer* to
bootstrap discovery, and any live 8115 peer will do. Your VM is syncing right
now and is holding a list of exactly those.

**Step 1 — harvest live 8115 peers from the VM** (run on the VM, not Windows):

```bash
curl -s -X POST http://127.0.0.1:8114 \
  -H 'Content-Type: application/json' \
  -d '{"id":1,"jsonrpc":"2.0","method":"get_peers","params":[]}' \
| python3 -c '
import sys, json, ipaddress, re
peers = json.load(sys.stdin)["result"]
seen, out = set(), []
for p in peers:
    nid = p.get("node_id")
    cands = [a["address"] for a in p.get("addresses", [])]
    if p.get("connected_addr"): cands.append(p["connected_addr"])
    for addr in cands:
        if "/tcp/8115" not in addr: continue
        m = re.search(r"/ip4/([0-9.]+)/", addr)
        if not m: continue
        try:
            if not ipaddress.ip_address(m.group(1)).is_global: continue
        except ValueError: continue
        if "/p2p/" not in addr and nid: addr = addr.rstrip("/") + "/p2p/" + nid
        if "/p2p/" not in addr: continue
        if addr in seen: continue
        seen.add(addr); out.append(addr)
for a in out[:25]: print(f'      "{a}",')
print(f"# {len(out)} distinct 8115 peers found", file=sys.stderr)
'
```

**Step 2 — paste them into the `bootnodes = [` list** in
`manifest/03-configmap-ckb.yaml`, *above* the existing 15 entries. Keep the
8114 ones: they cost only a timed-out dial each, and the moment the firewall
rule lands they start working with no config change.

**Step 3 — apply and restart:**

```powershell
kubectl apply -f manifest/03-configmap-ckb.yaml
kubectl rollout restart statefulset/ckb -n nervos
```

Then check `get_peers` per section 6. Once *any* peer connects, CKB's
Discovery protocol supplies more, and it writes them to its own peer store on
the PVC (`data/network/peer_store`) — so the node becomes self-sustaining and
these seed entries stop mattering. Harvested IPs go stale, so treat them as
a bootstrap aid, not permanent config.

**Honest limits of the workaround.** Peer quality will be thinner than a
node that can also reach the 8114 bootnodes, and if a cold start ever happens
when every seeded peer has churned, it will not bootstrap. Keep pursuing 2b —
this buys you the sync time while the ticket moves, it is not a substitute.

**A second option, if the harvest comes back short:** your VM can be the
bridge peer directly. It is not currently reachable from the internet
(`20.118.15.24:8115` is closed inbound, checked externally), but that is a VM
NSG rule *you* control, and it can be scoped to a single source — the AKS
egress IP `13.86.34.113`. Then add the VM to `bootnodes` as
`/ip4/20.118.15.24/tcp/8115/p2p/<VM node id>`, where the node id comes from:

```bash
curl -s -X POST http://127.0.0.1:8114 -H 'Content-Type: application/json' \
  -d '{"id":1,"jsonrpc":"2.0","method":"local_node_info","params":[]}'
```

### 2d. Either way, apply the config fix now

Section 3 onward stands on its own. The unmounted `ckb.toml` was a genuine,
separate defect — the node cannot sync with it regardless of the firewall.
Fix config now and the node starts syncing the moment either 2b or 2c lands.

There is no way to tunnel CKB P2P over 443 the way THORChain routed genesis
over Liquify's HTTPS gateway. Genesis is a one-time file fetch; P2P is a
continuous binary protocol to many peers.

---

## 3. Your answer on the Service file: yes, you were right

> *"here we are not following the ips configuration properly, we should not
> use external ip in the cluster? because in other nodes we also dont use
> like Cronos"*

Correct, and Cronos says so in its own words:

> `Agent/CronosPOS/04-service.yaml` — "**NO LOADBALANCER ON P2P (26656), on
> purpose.** A full node needs only OUTBOUND p2p, which works from the pod IP
> via the cluster's egress path with no Service involved. […] advertising an
> address you cannot be reached on gets you deprioritised and dropped by
> peers, which is worse than advertising nothing. **We do neither.**"

THORChain, same call: "Deliberately NOT set: EXTERNAL_IP."

The old Nervos service file had a **public** LoadBalancer on P2P 8115 —
the only chain in this repo that did. Combined with the ckb.toml that would
have advertised `20.118.15.24` (the **Docker VM's** public IP, a completely
different machine), it was the worst of both worlds: paying for a public IP
while telling peers to dial someone else's box.

`04-service.yaml` now matches the Cronos pattern exactly:

| Service | Type | Ports | Change |
|---|---|---|---|
| `ckb-headless` | Headless | 8115, 8114 | added `publishNotReadyAddresses: true` |
| `ckb` | ClusterIP | 8114 | **new** — in-cluster consumers |
| `ckb-rpc` | Internal LB | 8114, nodePort **30814** | nodePort now pinned |
| ~~`ckb-p2p`~~ | ~~Public LB~~ | ~~8115~~ | **deleted** |

Two things worth knowing about this cluster:

- The AKS subnet `10.202.17.192/27` is **full**, so `ckb-rpc` may sit at
  `EXTERNAL-IP <pending>` like `thornode-rpc` and `chain-maind-rpc` already
  do. That is survivable — the pinned nodePort still works:
  `http://<any-node-ip>:30814`. Pinning it is why deleting and recreating the
  Service won't break every URL you've handed out.
- `30814` doesn't collide with Cronos (30657/31317/30909) or THORChain
  (30399/32210/30609). Keep a port registry.

**To enable inbound peering later** (optional — it improves peer quality, it
is not needed to sync) you need *both*, together: a public LoadBalancer on
8115 **and** `network.public_addresses` set to that LB's public IP. Do both
or neither. Half of it is worse than none of it.

---

## 4. Everything else that was fixed

| # | Was | Now | Why it matters |
|---|---|---|---|
| 1 | config mount commented out | ConfigMap `nervos-ckb-config`, mounted via subPath | **the fix.** Node was running an unknown config |
| 2 | `ckb.toml` had 1 bootnode on :8115 | 15 official bootnodes on :8114 | 1 unreachable bootnode = 0 peers |
| 3 | `public_addresses` = Docker VM IP | absent | advertising an unreachable address gets you dropped by peers |
| 4 | `discovery_local_address = true` | `false` | stops advertising the unroutable 10.x pod IP |
| 5 | liveness `tcpSocket:8115`, no startupProbe | startup+readiness on **8114**, no liveness | old probes SIGTERM'd the pod 150s after start — restart loop that looks like "not syncing". Probing 8115 also fed the node junk p2p connections from the kubelet every 30s |
| 6 | `limits.cpu: "2"` | removed | CFS throttling on a verification-bound chain node = "runs but never catches up". Same call as Cronos |
| 7 | `limits.memory: 4Gi` | 10Gi (request 4Gi) | 1 GiB RocksDB cache + 256 MB header_map + indexer OOMKills at 4Gi |
| 8 | `args: [run, --indexer]` | `args: [run]` | mirrors the proven VM setup; RichIndexer is enabled via `rpc.modules` |
| 9 | `runAsNonRoot: false` | UID/GID 1000, `runAsNonRoot: true` | Gatekeeper on this cluster forbids root. Image already declares USER 1000 |
| 10 | fsGroup relabel every boot | `fsGroupChangePolicy: OnRootMismatch` | kills the `secret_key permission is not less than 0o600` warning, and saves minutes of re-chown per restart on a large archive |
| 11 | no `enableServiceLinks` | `false` | k8s injects `CKB_*_PORT` vars into a process whose config env prefix is `CKB_` |
| 12 | PVC 100Gi | 512Gi | full archive + RichIndexer from genesis. Azure disks grow, never shrink |
| 13 | `terminationGracePeriodSeconds: 60` | 300 | SIGKILL mid-RocksDB-write costs a re-sync from genesis |

---

## 5. Deploy

```powershell
cd <repo>\Nervos\manifest

kubectl apply -f 00-namespace.yaml
kubectl apply -f 01-storageclass.yaml
kubectl apply -f 02-pvc.yaml
kubectl apply -f 03-configmap-ckb.yaml
kubectl apply -f 03-statefulset.yaml
kubectl apply -f 04-service.yaml
```

The old public P2P LoadBalancer is not removed by `apply` — delete it, and
release its Azure public IP:

```powershell
kubectl delete svc ckb-p2p -n nervos --ignore-not-found
```

The StatefulSet's `securityContext` and `resources` changed, so force a
fresh pod:

```powershell
kubectl rollout restart statefulset/ckb -n nervos
kubectl rollout status  statefulset/ckb -n nervos --timeout=10m
```

Confirm the config actually landed — this is the check that proves cause A
is fixed:

```powershell
kubectl exec -n nervos ckb-0 -c ckb -- grep -c "tcp/8114/p2p" /var/lib/ckb/ckb.toml
```

Expect `15`. Anything else means the mount didn't take.

No PVC wipe is needed. The existing volume is at genesis with the same
mainnet genesis hash, so it is compatible — the node just picks up with real
bootnodes.

---

## 6. Verify it is syncing

```powershell
kubectl logs -n nervos ckb-0 -c ckb -f --tail=100
```

Within ~2 minutes of a working setup you want to see peer/sync activity and
then repeating block lines. Continued silence after `Waiting exit signal...`
= still 0 peers = go back to section 2.

Peer count and tip height over RPC. First check what HTTP client the image
has:

```powershell
kubectl exec -n nervos ckb-0 -c ckb -- sh -c "command -v curl wget nc || echo NONE"
```

If `curl` is there:

```powershell
# peer count — the number that actually matters
kubectl exec -n nervos ckb-0 -c ckb -- curl -s -X POST http://localhost:8114 -H "Content-Type: application/json" -d '{\"id\":1,\"jsonrpc\":\"2.0\",\"method\":\"get_peers\",\"params\":[]}'

# tip height (hex)
kubectl exec -n nervos ckb-0 -c ckb -- curl -s -X POST http://localhost:8114 -H "Content-Type: application/json" -d '{\"id\":1,\"jsonrpc\":\"2.0\",\"method\":\"get_tip_block_number\",\"params\":[]}'

# sync state
kubectl exec -n nervos ckb-0 -c ckb -- curl -s -X POST http://localhost:8114 -H "Content-Type: application/json" -d '{\"id\":1,\"jsonrpc\":\"2.0\",\"method\":\"sync_state\",\"params\":[]}'
```

If the image has neither `curl` nor `wget`, port-forward and query from
Windows instead:

```powershell
kubectl port-forward -n nervos svc/ckb 8114:8114
# in a second window:
Invoke-RestMethod -Uri http://localhost:8114 -Method Post -ContentType application/json `
  -Body '{"id":1,"jsonrpc":"2.0","method":"get_peers","params":[]}'
```

**`get_peers` returning `[]` after 5 minutes is the single diagnostic that
matters.** Zero peers with the config confirmed mounted means egress, not
config.

Disk growth — check weekly and expand before you are within ~15% of full:

```powershell
kubectl exec -n nervos ckb-0 -c ckb -- du -sh /var/lib/ckb/data
kubectl get pvc -n nervos
```

To expand, edit `storage:` in `02-pvc.yaml` upward and re-apply. Online, no
re-sync. Never downward — Azure disks cannot shrink.

---

## 7. Once it is syncing

Turn the height-progress liveness probe on. It is written out and commented
in `03-statefulset.yaml`, matching the Cronos one: it fires only after the
tip fails to move across three consecutive 10-minute checks. Enable it
**only after** the node is confirmed syncing and `curl` is confirmed present
in the image (section 6) — a liveness probe that fails closed during a
multi-week genesis sync costs you the sync.

Also expect the sync to be slow on `StandardSSD_LRS`: flat ~500 IOPS at any
size. If it proves IOPS-bound, `Premium_LRS` scales IOPS with size — but
StorageClass `parameters` are **immutable**, so that means a new
StorageClass, a new PVC, and a restart from genesis. The only cheap moment
to make that call is now, before the sync has run for weeks.
