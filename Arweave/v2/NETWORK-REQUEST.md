# Arweave node on AKS — network access request

**Cluster:** `cuscoiniadevaks` (centralus) · **Namespace:** `arweave`

**We are NOT asking for a closed port to be opened.** TCP 1984 already connects
fine. Our traffic is being inspected as HTTP and rejected by an FQDN rule.

**Ask in one line:** allow outbound TCP 1984 via a **network rule**, so it is
permitted before application/FQDN rules are evaluated.

---

## 1. What we are running

An **Arweave blockchain node**. Like any blockchain node it must talk to other
computers running the same software ("peers") on the internet, over **TCP port
1984**.

---

## 2. Why this looks different from the other blockchain nodes

There are 10+ blockchain nodes on this cluster and none of them has this
problem. That is not a coincidence — it is the clue that identifies the cause.

| | Protocol | Inspected as HTTP? | Works? |
|---|---|---|---|
| Thorchain, Mantra, Fetch, etc. | custom **binary** (Tendermint p2p, etc.) | no | yes |
| **Arweave** | **plain HTTP** on port 1984 | **yes** | **no** |

Arweave is the only node here whose peer protocol is ordinary HTTP. So it is
the only one the firewall recognises and applies **application / FQDN rules**
to.

And here is the mismatch: those rules match on **hostname**, but the Arweave
software connects to peers by **IP address**. It resolves each peer's name once
at startup, then talks only to the number. So every request arrives with an IP
where the rule expects a name, matches nothing, and is rejected.

> **Analogy.** Security lets couriers through if they name the company they are
> visiting. Deliveries in unmarked vans are waved past without question. Our
> courier arrives in a marked van but only knows the building number — so he is
> the only one who gets stopped.

---

## 3. Proof

Run from a test container **inside the cluster, in the `arweave` namespace**,
against the same server, IP and port — changing only the layer being used.

```
Test 1  raw TCP connect to 40.160.31.17:1984          ->  TCP-OK
Test 2  HTTP GET  http://40.160.31.17:1984/info       ->  HTTP 503
Test 3  same request + Host: vin-1...arweave.xyz      ->  HTTP 200 OK
```

Reproduce with:

```
kubectl run t -n arweave --image=alpine:3.19 --restart=Never --rm -it -- sh -c "echo ==1-TCP-CONNECT; nc -z -w 5 40.160.31.17 1984 && echo TCP-OK || echo TCP-FAIL; echo ==2-HTTP-BY-IP; wget -T 8 -q -O /dev/null http://40.160.31.17:1984/info && echo HTTP-OK || echo HTTP-FAIL; echo ==3-HTTP-BY-NAME; wget -T 8 -q -O /dev/null --header Host:vin-1.east.us.north-america.arweave.xyz http://40.160.31.17:1984/info && echo NAME-OK || echo NAME-FAIL"
```

### What each line establishes

- **Test 1 (TCP-OK)** — the port is open and routable. This is not a blocked
  port, and not a firewall "deny" at the network layer.
- **Test 2 (503)** — the same connection, once it carries HTTP, is refused.
  A `503` is an **HTTP status code**, so something is reading our HTTP and
  answering on the server's behalf.
- **Test 3 (200 OK)** — identical request plus the hostname, and it succeeds.
  The **only** variable is the hostname.

Two further confirmations:

- The `503` returns in **0.02 seconds**. A genuine reply from that server takes
  about **0.6 seconds**. The response is produced inside our network; the
  request never reaches the internet.
- The same bare-IP request from **outside** this cluster returns **HTTP 200**,
  so the destination server is healthy and does accept IP-based requests.

---

## 4. The request

> Please add a **network rule** permitting **outbound TCP port 1984** from the
> AKS cluster `cuscoiniadevaks` to the internet.
>
> In Azure Firewall, network rules are evaluated **before** application rules,
> so this exempts the traffic from HTTP/FQDN inspection — which is what is
> currently rejecting it.
>
> An **application / FQDN rule will not work**, because the Arweave client
> connects to peers by IP address, not hostname.

### Why we cannot supply a list of IPs to allow

Blockchain peers are not a fixed set. The node **discovers new peers while it
runs** and connects to whatever the network provides — hundreds of addresses,
changing continuously, operated independently worldwide. Any static list is
stale within hours.

### Scope of the change

| | |
|---|---|
| Direction | **Outbound only.** Nothing new is exposed inbound |
| Port | TCP 1984 only |
| Source | AKS cluster subnet |
| Data | Public blockchain data. Arweave's peer protocol is public by design |

The node accepts no connections from the internet — it is reachable only
inside the cluster.

---

## 5. Why we are asking rather than working around it

We built a temporary workaround — a small proxy in the pod that adds the
hostname to each request so the firewall permits it. It worked; the node joined
the blockchain successfully.

**We have switched it off and are asking for the rule instead**, for three
reasons:

1. **It defeats a control that was put there deliberately.** Technically it is
   just an HTTP proxy, but the effect is to get around FQDN filtering. We do
   not want to run that on a managed cluster without your agreement.
2. **It only half works.** The node reaches only the handful of peers hardcoded
   into the proxy — about 6, where a healthy node sees hundreds. Every peer
   discovered at runtime is still blocked.
3. **It hides the requirement.** The same issue will recur on the next
   HTTP-protocol node, and again in production.

---

## 6. Impact if not opened

The node starts but cannot function:

- It joins the blockchain once, then **stops following it** — the block height
  freezes and never advances.
- It cannot reach the wider peer network, so it cannot independently verify
  what it is told.
- Any future node using an HTTP-based protocol will hit the same wall.

---

## 7. Verifying the fix

After the rule is applied, this should return blockchain data instead of an
error:

```
kubectl run nettest -n arweave --image=alpine:3.19 --restart=Never --rm -it -- \
  wget -T 10 -qO- http://40.160.31.17:1984/info
```

Expected: output containing `"network":"arweave.N.1"` and a block height around
1,983,000.

Today it returns `HTTP 503`.

---

## Appendix — summary for the firewall engineer

| Item | Detail |
|---|---|
| Application | Arweave blockchain node, release 2.9.5.1 |
| Cluster / namespace | `cuscoiniadevaks` / `arweave` |
| Symptom | Outbound HTTP on TCP 1984 to peer IPs returns synthetic `HTTP 503` in 0.02s |
| TCP layer | **Open.** Raw TCP connect to the same IP:port succeeds |
| Trigger | Request carries `Host: <ip>`; matches no FQDN rule |
| Control test | Same IP:port with `Host: <valid-hostname>` returns HTTP 200 |
| External control | Same bare-IP request from outside the cluster returns HTTP 200 |
| Why other nodes are unaffected | Their p2p protocols are binary, so they are not matched by application rules |
| Requested | Network rule: outbound TCP 1984 to Any |
| Not sufficient | Application/FQDN rule — client connects by IP |
| Inbound | None required |
