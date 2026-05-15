# HSRP — Hot Standby Router Protocol

## What Is HSRP?

**HSRP (Hot Standby Router Protocol)** is a Cisco proprietary **First Hop Redundancy Protocol (FHRP)**. Its job is to eliminate the **single point of failure at the default gateway**.

Here's the problem it solves: Every end device (PC, server, phone) needs a **default gateway** — one IP address it sends traffic to when the destination is outside its subnet. If that gateway goes down, the device is completely cut off from the rest of the network — even if alternative paths exist.

HSRP fixes this by making a **group of routers or Layer 3 switches share a single virtual IP and virtual MAC address**. The end devices point to the virtual IP as their default gateway, and HSRP ensures one physical device is always actively handling traffic for it.

---

## How HSRP Works

HSRP assigns roles to devices in a group:

| Role | Description |
|---|---|
| **Active Router** | The router currently forwarding traffic for the virtual IP |
| **Standby Router** | The backup router — monitoring the active router |
| **Other routers** | Listen and wait; can become standby or active |

### Virtual IP and Virtual MAC

The group shares:
- A **Virtual IP (VIP)** — configured by you (e.g., `172.20.10.1`)
- A **Virtual MAC** — automatically generated: `0000.0C07.ACxx` (where `xx` is the group number in hex)

End devices use the VIP as their default gateway. They ARP for the VIP and receive the virtual MAC. Traffic flows to whichever physical router is currently active — transparently.

---

## HSRP State Machine

HSRP routers go through these states before becoming active:

```
Initial → Learn → Listen → Speak → Standby → Active
```

- **Initial** — HSRP just started
- **Learn** — Waiting to hear the virtual IP from the active router
- **Listen** — Knows the VIP but is not active or standby
- **Speak** — Sending hello messages, participating in election
- **Standby** — Backup router — monitoring the active
- **Active** — Currently forwarding traffic ✅

---

## HSRP Election — Priority

The **Active** router is elected based on **priority** (default: 100). Higher priority wins.

```
standby 10 priority 111     ← This router will win and become Active
standby 10 priority 90      ← This router will become Standby
```

**Tiebreaker:** If priorities are equal, the router with the **highest IP address** on the interface wins.

### Preemption

By default, if the active router recovers after a failure, it does **not** automatically take back the active role. To enable this:

```
standby 10 preempt
```

With preempt enabled, a router with higher priority will take over the active role as soon as it comes back online. Without it, the lower-priority router stays active even after the higher-priority one recovers.

---

## HSRP Hello Packets

HSRP routers send **hello packets** to each other using:
- **Multicast address:** `224.0.0.2`
- **UDP port:** 1985
- **Default hello timer:** 3 seconds
- **Default hold timer:** 10 seconds (3 missed hellos = failover)

If the standby router stops receiving hellos from the active router for 10 seconds, it promotes itself to active and begins forwarding traffic. The failover is **transparent to end devices** — their default gateway IP never changes.

---

## HSRP vs VRRP vs GLBP

| Feature | HSRP | VRRP | GLBP |
|---|---|---|---|
| Standard | Cisco proprietary | Open standard (RFC 5798) | Cisco proprietary |
| Roles | Active / Standby | Master / Backup | AVG + AVFs |
| Load balancing | No (only one active) | No | Yes (multiple active) |
| Virtual MAC | `0000.0C07.ACxx` | `0000.5E00.01xx` | `0007.B400.xxyy` |
| Default priority | 100 | 100 | 100 |
| Preempt default | Disabled | Enabled | Disabled |

> **VRRP** is the open-standard equivalent of HSRP — same concept, works across vendors. **GLBP** goes further and load-balances across all routers in the group simultaneously.

---

## HSRP in This Lab

In this lab, **SW1 (IOU1)** and **SW2 (IOU2)** both have SVIs for VLAN 10 and VLAN 20. Without HSRP, you'd have to pick one switch as the default gateway — and if it goes down, all hosts in those VLANs lose connectivity.

**HSRP makes SW1 the active gateway for VLAN 10 and VLAN 20**, while SW2 is the standby. If SW1 fails, SW2 automatically takes over — end devices never notice because the virtual IP stays the same.

```
SW1 (Active)  ←—— Virtual IP: 172.20.10.1 ——→  End Devices
SW2 (Standby)     (Ready to take over)
```

---

## Configuration Applied

### SW1 (IOU1) — Active for VLAN 10 and VLAN 20

```
interface vlan10
 standby 10 ip 172.20.10.1
 standby 10 priority 111
 standby 10 preempt

interface vlan20
 standby 20 ip 172.20.20.1
 standby 20 priority 111
 standby 20 preempt
```

### SW2 (IOU2) — Standby for VLAN 10 and VLAN 20

```
interface vlan10
 standby 10 ip 172.20.10.1
 standby 10 priority 90
 standby 10 preempt

interface vlan20
 standby 20 ip 172.20.20.1
 standby 20 priority 90
 standby 20 preempt
```

**Configuration breakdown:**
- `standby 10` — HSRP group number 10 (should match the VLAN for clarity)
- `ip 172.20.10.1` — The virtual IP that end devices will use as their default gateway
- `priority 111` — SW1 wins the election (111 > 90)
- `preempt` — SW1 will take back the active role if it recovers from a failure

> **Note:** The virtual IP (`172.20.10.1`) must be in the same subnet as the SVI IPs (`172.20.10.254` on SW1, etc.) but different from any physical interface IP.

---

## Verification Commands

```
show standby brief
```
Quick view of all HSRP groups — shows state (Active/Standby), priority, virtual IP, and active router address.

Expected output:
```
                     P indicates configured to preempt.
                     |
Interface   Grp  Pri P State   Active          Standby         Virtual IP
Vl10        10   111 P Active  local           172.20.10.254   172.20.10.1
Vl20        20   111 P Active  local           172.20.20.254   172.20.20.1
```

```
show standby vlan 10
```
Detailed HSRP information for a specific group — includes hello/hold timers, virtual MAC, preempt status.

```
show standby
```
Full detail for all HSRP groups on the device.

---

## Testing HSRP Failover

1. Ping the virtual IP from a PC continuously:
```
ping 172.20.10.1 repeat 10000
```

2. Shut down the active switch's VLAN interface:
```
SW1(config)# interface vlan10
SW1(config-if)# shutdown
```

3. Watch the standby switch take over — pings may miss 1-2 packets during failover (within the hold timer period), then resume.

---

## Common HSRP Issues

| Problem | Likely Cause | Fix |
|---|---|---|
| Both routers think they're active | Network partition / no communication | Check physical connectivity between switches |
| Failover not happening | Preempt not configured | Add `standby X preempt` |
| Wrong router is active after recovery | Preempt disabled | Add `standby X preempt` |
| Virtual IP not reachable | SVI down or VLAN not active | Check `show interfaces vlan X` status |
| HSRP group not forming | Mismatched group number or VIP | Verify both switches use same group number and VIP |
