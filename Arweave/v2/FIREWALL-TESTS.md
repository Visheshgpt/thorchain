# Reproducing the issue — 3 tests, ~2 minutes

Cluster `cuscoiniadevaks`, namespace `arweave`. Everything below runs in a
throwaway pod that deletes itself. Nothing is installed or changed.

`40.160.31.17` is a public Arweave peer. Its hostname is
`vin-1.east.us.north-america.arweave.xyz` — same machine, both ways of
addressing it.

---

## The test

```
kubectl run fwtest -n arweave --image=alpine:3.19 --restart=Never --rm -it -- sh -c "echo '== 1. RAW TCP to 40.160.31.17:1984'; nc -z -w 5 40.160.31.17 1984 && echo '   RESULT: TCP CONNECTS' || echo '   RESULT: TCP BLOCKED'; echo; echo '== 2. HTTP to the SAME ip:port'; wget -T 8 -q -O /dev/null http://40.160.31.17:1984/info && echo '   RESULT: HTTP OK' || echo '   RESULT: HTTP REJECTED'; echo; echo '== 3. SAME request + Host header'; wget -T 8 -q -O /dev/null --header Host:vin-1.east.us.north-america.arweave.xyz http://40.160.31.17:1984/info && echo '   RESULT: HTTP OK' || echo '   RESULT: HTTP REJECTED'"
```

## What we get today

```
== 1. RAW TCP to 40.160.31.17:1984
   RESULT: TCP CONNECTS                <- port is open

== 2. HTTP to the SAME ip:port
wget: server returned error: HTTP/1.1 503 Service Unavailable
   RESULT: HTTP REJECTED               <- rejected once it's HTTP

== 3. SAME request + Host header
   RESULT: HTTP OK                     <- works with a hostname
```

Same destination, same port, seconds apart. The only variable between 2 and 3
is whether the request carries a hostname.

---

## Two supporting details

**The rejection is generated locally, not by the peer.**

```
from inside the cluster :  HTTP 503 in 0.02 s
from outside            :  HTTP 200 in 0.64 s
```

0.02 s is far too fast to be a reply from that server — the request never left
the network.

**The peer is healthy.** Run from any machine outside the cluster:

```
curl -s -o /dev/null -w "%{http_code} %{time_total}s\n" http://40.160.31.17:1984/info
```

Returns `200`.

---

## Why this affects only the Arweave node

| Node | Peer protocol | Inspected as HTTP? | Working? |
|---|---|---|---|
| Thorchain, Mantra, Fetch, Archway, … | binary (Tendermint p2p etc.) | no | yes |
| **Arweave** | **plain HTTP on port 1984** | **yes** | **no** |

Arweave is the only node here whose peer protocol is ordinary HTTP, so it is
the only one matched by application/FQDN rules. It resolves peer hostnames to
IP addresses once at startup and connects by IP from then on, so its requests
never carry a hostname to match against.

Peer IPs cannot be allow-listed — the node discovers new peers continuously
while running, from operators worldwide.

---

## The request

Allow **outbound TCP 1984** from the AKS subnet as a **network rule**, so it is
permitted before application/FQDN rule evaluation.

Cluster config for reference:

```
outboundType : userDefinedRouting
route table  : RT-CUS   (AZRG-ALL-ITS-CORE-SYS)
NSG          : NSG-CUS  (AZRG-ALL-ITS-CORE-SYS)
```

Scope: outbound only, TCP 1984 only, from the AKS subnet. Nothing new is
exposed inbound — the node accepts no connections from the internet.

---

## Verifying the fix

```
kubectl run fwtest -n arweave --image=alpine:3.19 --restart=Never --rm -it -- wget -T 10 -qO- http://40.160.31.17:1984/info
```

**Working:** a line of JSON containing `"network":"arweave.N.1"` and a block
height around 1,983,000.

**Still blocked:** `wget: server returned error: HTTP/1.1 503`.
