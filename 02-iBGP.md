# iBGP — Internal Border Gateway Protocol

## What Is BGP?

BGP (Border Gateway Protocol) is the **routing protocol of the internet**. It is the only EGP (Exterior Gateway Protocol) in widespread use today. BGP is a **path-vector protocol** — unlike OSPF which uses link cost, BGP makes routing decisions based on a rich set of **path attributes** (AS path, next-hop, MED, local preference, etc.).

BGP has two modes:
- **iBGP** — Between routers in the **same** Autonomous System
- **eBGP** — Between routers in **different** Autonomous Systems

This page covers **iBGP**.

---

## Why iBGP?

When multiple routers inside one AS receive BGP routes from outside (via eBGP), they need a way to share those routes internally — **without redistributing into OSPF or ISIS** (which would flood every BGP route into the IGP).

iBGP solves this: routers inside the same AS form BGP sessions with each other and share externally-learned BGP routes **while keeping them as BGP routes**.

### Key Rule: iBGP Does Not Re-Advertise
A critical difference from eBGP — an iBGP router will **not re-advertise** a route it learned from one iBGP peer to another iBGP peer. This prevents routing loops inside the AS.

This means in a full mesh of iBGP routers, every router must peer with every other router directly (or you use **Route Reflectors** or **Confederations** to scale).

---

## iBGP vs eBGP — Key Differences

| Feature | iBGP | eBGP |
|---|---|---|
| Same or different AS? | Same AS | Different AS |
| TTL in TCP session | 255 (multi-hop by default) | 1 (direct link assumed) |
| Next-hop behavior | Next-hop unchanged | Next-hop set to self |
| Route re-advertisement | Will NOT re-advertise to other iBGP peers | Will re-advertise |
| AD (Administrative Distance) | 200 | 20 |

> iBGP has a higher AD (200) than eBGP (20) and OSPF (110) — meaning iBGP routes are less preferred than IGP routes when both exist to the same destination. This is by design.

---

## iBGP in This Lab

In this lab, **R3 and R4** are both in **AS100**. R3 has an eBGP session with R5 (AS200) and learns external routes. R4 also needs to know about those external routes — so R3 and R4 form an **iBGP session** to share this information internally.

```
R3 (AS100) ←—iBGP—→ R4 (AS100)
     ↑
  eBGP to R5 (AS200)
```

---

## Configuration Applied

### R3
```
router bgp 100
 neighbor 11.11.34.4 remote-as 100
```

### R4
```
router bgp 100
 neighbor 11.11.34.3 remote-as 100
```

Since both routers have `remote-as 100` and they are themselves in AS100, IOS automatically treats this as an iBGP session.

---

## Issue Faced — iBGP Neighbor Stuck in ACTIVE State

### Symptom
On R3:
```
show ip bgp summary
```
Output showed:
```
Neighbor        State/PfxRcd
11.11.34.4      Active
```

`Active` means BGP is actively trying to open a TCP session to the peer but failing.

### What "Active" Means
The BGP state machine:
```
Idle → Connect → Active → OpenSent → OpenConfirm → Established
```

- **Idle** — BGP not started or administratively shut down
- **Connect** — Attempting TCP connection
- **Active** — TCP connection failed, retrying
- **Established** — Peering is up ✅

`Active` is NOT a good state. It means TCP cannot complete.

### How to Troubleshoot
```
ping 11.11.34.4 source 11.11.34.3   → Is L3 reachability there?
show ip route 11.11.34.0            → Is the route in the table?
show ip interface brief             → Is the interface up/up?
```

If ping works but BGP is still Active, check:
- Correct `remote-as` number on both sides
- No ACL blocking TCP port 179
- `update-source` if using loopbacks

---

## Verification Commands

```
show ip bgp summary
```
Look for `Established` state and a number in `State/PfxRcd` (number of prefixes received).

```
show ip bgp
```
Shows the full BGP table. Routes marked with `>` are best and installed in the routing table. Routes marked `i` after the next-hop are iBGP-learned.

```
show ip bgp neighbors 11.11.34.4
```
Detailed output for a specific peer — shows state, uptime, messages sent/received, and capabilities negotiated.

---

## Common iBGP Issues

| Problem | Likely Cause | Fix |
|---|---|---|
| Stuck in Active | No TCP reachability to peer | Verify route to peer IP exists |
| Route not re-advertised | iBGP split-horizon rule | Use Route Reflectors or full mesh |
| Next-hop unreachable | iBGP doesn't change next-hop | Run `next-hop-self` on iBGP peers or redistribute next-hop |
| Routes learned but not in routing table | Higher AD / IGP already has route | Normal — IGP wins; BGP route is backup |
