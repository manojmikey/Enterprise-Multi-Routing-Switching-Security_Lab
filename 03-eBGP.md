# eBGP — External Border Gateway Protocol

## What Is eBGP?

**eBGP** is BGP running between routers in **different Autonomous Systems**. This is how the internet works — every ISP, cloud provider, and large enterprise exchanges routing information with each other using eBGP.

An **Autonomous System (AS)** is a collection of IP networks and routers under the control of a single organization that presents a common routing policy to the internet. Each AS has a unique **AS Number (ASN)** — either 16-bit (1–65535) or 32-bit.

BGP path-vector routing means routers don't just know the metric to a destination — they know the **entire sequence of ASes** the traffic will traverse. This AS_PATH attribute is what prevents routing loops between autonomous systems.

---

## eBGP vs iBGP — The Core Difference

| | iBGP | eBGP |
|---|---|---|
| Routers in | Same AS | Different AS |
| next-hop | Unchanged | Set to self (the peering IP) |
| AS_PATH | AS not added | Own AS prepended before advertising |
| Route re-advertisement | Blocked by split-horizon | Freely re-advertised |
| TTL | 255 | 1 (must be directly connected by default) |
| AD | 200 | 20 |

---

## How BGP Selects the Best Path

BGP doesn't use metrics like OSPF. It uses a decision process with **multiple attributes**, evaluated in order:

1. **Weight** (Cisco proprietary, local to router — higher is better)
2. **Local Preference** (higher is better, shared within AS)
3. **Locally originated** (network statement or redistribute wins over learned)
4. **AS_PATH length** (shorter is better)
5. **Origin** (IGP > EGP > Incomplete)
6. **MED** (Multi-Exit Discriminator — lower is better)
7. **eBGP over iBGP** (prefer eBGP-learned routes)
8. **Lowest IGP metric to next-hop**
9. **Oldest eBGP route** (prefer stable route)
10. **Lowest Router ID**
11. **Shortest cluster list**
12. **Lowest neighbor IP**

In this lab, only one eBGP path exists for each prefix, so the best-path selection process doesn't come into play much — but understanding it is critical for interviews.

---

## eBGP in This Lab

**R3 (AS100)** forms an eBGP session with **R5 (AS200)**.

```
AS100                    AS200
R3 (11.11.35.3) ←——eBGP——→ R5 (11.11.35.5)
```

R3 advertises internal HQ networks (like 11.11.13.0/24) to R5. R5 advertises its own loopback (5.5.5.5/32) and connected networks back to R3.

---

## Configuration Applied

### R3
```
router bgp 100
 neighbor 11.11.35.5 remote-as 200
 network 11.11.13.0 mask 255.255.255.0
 redistribute ospf 1
```

`remote-as 200` tells R3 this is an eBGP neighbor (different AS).

`network 11.11.13.0 mask 255.255.255.0` advertises this exact prefix to all BGP peers. The prefix **must already exist in the routing table** for BGP to advertise it.

`redistribute ospf 1` redistributes all OSPF-learned routes into BGP so they can be advertised to R5.

### R5
```
router bgp 200
 bgp router-id 5.5.5.5
 neighbor 11.11.35.3 remote-as 100
 network 5.5.5.5 mask 255.255.255.255
 network 11.11.35.0 mask 255.255.255.0
```

R5 advertises its loopback and the connected subnet to R3.

---

## Issues Faced & Explained

### Issue 1 — R5 Automatically Learned R3's Routes
**Confusion:** Only the `neighbor` statement was configured on R5, but R5's routing table showed:
```
B    11.11.13.0/24 [20/0] via 11.11.35.3
```

**Explanation:** This is exactly how BGP works. R3 had `network 11.11.13.0 mask 255.255.255.0` configured, which told R3 to advertise that prefix to **all** its BGP neighbors — including R5. As soon as the eBGP session came up, R3 pushed that advertisement to R5, and R5 installed it automatically. No extra configuration needed on R5 to receive it.

### Issue 2 — R3 Was Not Learning R5's Routes
**Cause:** R5 was not advertising anything. The `network` commands were missing on R5.

```
network 5.5.5.5 mask 255.255.255.255
```

After adding this, R3's BGP table showed:
```
B    5.5.5.5/32 [20/0] via 11.11.35.5
```

**Key concept:** BGP is not like OSPF. OSPF automatically shares connected interface networks with neighbors. BGP **only advertises what you explicitly tell it to advertise** (via `network` statements or `redistribute`).

### Issue 3 — Ping to 11.11.13.1 from R5 Failed
**Symptom:** R5 had a BGP route to 11.11.13.0/24, but pinging 11.11.13.1 failed.

**Explanation:** BGP only provides the **forward path**. R5 knew how to reach 11.11.13.1 (forward path via 11.11.35.3). But R1 (at 11.11.13.1) had no route back to R5's source IP (11.11.35.5 or 11.11.45.5).

Traffic went: R5 → R3 → R1 ✅
Return traffic: R1 had no route to R5 ❌

**Solution:** Redistribute BGP into OSPF on R1, so that internal OSPF routers learn about BGP-reachable networks:
```
router ospf 1
 redistribute bgp 100 subnets
```

---

## Why the `network` Command Works the Way It Does

In BGP, the `network` command is an **exact match** — the prefix and mask must match exactly what's in the routing table. This is different from OSPF where `network` with a wildcard enables OSPF on interfaces.

```
router bgp 100
 network 11.11.13.0 mask 255.255.255.0
```

This says: "If 11.11.13.0/24 exists in my IP routing table, advertise it to my BGP peers." If the route is missing from the table (even if the subnet exists but as a different prefix length), BGP won't advertise it.

---

## Verification Commands

```
show ip bgp summary              → Peer states and prefix counts
show ip bgp                      → Full BGP table
show ip bgp 5.5.5.5              → BGP table for a specific prefix
show ip bgp neighbors            → Detailed neighbor information
show ip route bgp                → Only BGP-installed routes in RIB (marked 'B')
```

---

## Common eBGP Issues

| Problem | Likely Cause | Fix |
|---|---|---|
| Neighbor stuck in Active | No L3 reachability or wrong peer IP | Ping test; verify interface IPs |
| Route not advertised | Prefix not in routing table | Add static or ensure IGP redistributes it |
| One-way routing (ping fails) | Return path missing | Redistribute BGP into IGP or add static on far end |
| Route learned but not in RIB | AD or more specific route from IGP wins | Normal behavior; check `show ip route` |
| Session keeps resetting | MTU or TCP issue | Check for ACLs blocking port 179 |
