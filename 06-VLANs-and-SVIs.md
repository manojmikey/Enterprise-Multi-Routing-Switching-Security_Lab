# VLANs & SVIs — Network Segmentation and Inter-VLAN Routing

## What Is a VLAN?

A **VLAN (Virtual Local Area Network)** is a logical subdivision of a physical LAN. It allows you to group devices together at Layer 2 regardless of their physical location — and more importantly, to **isolate groups of devices from each other** even if they share the same physical switch.

Without VLANs, every device on a switch is in the same **broadcast domain**. Every ARP request, every DHCP discover, every broadcast packet hits every device. This creates security risks and unnecessary traffic.

**VLANs solve this by:**
- Creating separate broadcast domains on a single switch
- Isolating departments (HR, Finance, IT, Servers) from each other at Layer 2
- Reducing broadcast traffic
- Improving security (VLAN 10 devices can't directly talk to VLAN 20 devices without a router)

---

## How VLANs Work

Each VLAN is identified by a **VLAN ID** — a number from 1 to 4094. VLAN 1 is the default VLAN.

### Access Ports
An **access port** belongs to exactly one VLAN. End devices (PCs, servers, printers) connect to access ports. The device has no idea VLANs exist — it just sends normal Ethernet frames.

```
switchport mode access
switchport access vlan 10
```

### Trunk Ports
A **trunk port** carries **multiple VLANs** simultaneously. Used between switches, and between switches and routers. Trunk ports use **802.1Q tagging** — a 4-byte tag is inserted into the Ethernet frame identifying which VLAN the frame belongs to.

```
switchport mode trunk
switchport trunk allowed vlan 10,20,30,40
```

### 802.1Q Tag Format
```
[ Destination MAC ] [ Source MAC ] [ 802.1Q Tag ] [ EtherType ] [ Payload ] [ FCS ]
                                        ↓
                         [ TPID: 0x8100 ] [ PCP ] [ DEI ] [ VLAN ID (12 bits) ]
```

The 12-bit VLAN ID field allows 4096 possible VLANs (0 and 4095 are reserved → 4094 usable).

---

## Inter-VLAN Routing

Devices in different VLANs are in different Layer 2 broadcast domains. To communicate across VLANs, traffic **must be routed at Layer 3**.

There are three ways to do inter-VLAN routing:

### 1. Router on a Stick (Legacy)
One physical interface on a router, configured with sub-interfaces (one per VLAN). The switch trunk connects to the router.

```
interface g0/0.10
 encapsulation dot1q 10
 ip address 172.20.10.254 255.255.255.0
```

Downside: All inter-VLAN traffic passes through a single physical link (bottleneck).

### 2. Layer 3 Switch with SVIs ← Used in This Lab
A **Layer 3 switch** (multilayer switch) can route between VLANs internally — no external router needed.

Each VLAN gets a **Switched Virtual Interface (SVI)** — a virtual interface on the switch that acts as the default gateway for that VLAN.

```
ip routing                               ← Enable Layer 3 routing on the switch

interface vlan 10
 ip address 172.20.10.254 255.255.255.0
 no shutdown
```

This is much more efficient — inter-VLAN routing happens in the switch's hardware (ASIC) at line rate.

### 3. Dedicated Layer 3 Links (Modern)
Each access switch has a routed (Layer 3) uplink to the distribution layer. VLANs don't span switches. More scalable, but more complex.

---

## VLANs in This Lab

Four VLANs were created for the HQ network to segment end users:

| VLAN | Network | Description |
|---|---|---|
| VLAN 10 | 172.20.10.0/24 | User group 1 (PC1, PC2) |
| VLAN 20 | 172.20.20.0/24 | User group 2 (PC3, PC4) |
| VLAN 30 | 172.20.30.0/24 | User group 3 (PC5, PC6) |
| VLAN 40 | 172.20.40.0/24 | User group 4 (PC7, PC8) |

Two Layer 3 switches (SW1/IOU1 and SW2/IOU2) handle inter-VLAN routing via SVIs.

---

## Configuration Applied

### Create VLANs (both switches)
```
vlan 10
 name VLAN10_USERS
vlan 20
 name VLAN20_USERS
vlan 30
 name VLAN30_USERS
vlan 40
 name VLAN40_USERS
```

### SVIs — SW1
```
ip routing

interface vlan10
 ip address 172.20.10.254 255.255.255.0
 no shutdown

interface vlan20
 ip address 172.20.20.254 255.255.255.0
 no shutdown
```

### SVIs — SW2
```
ip routing

interface vlan30
 ip address 172.20.30.254 255.255.255.0
 no shutdown

interface vlan40
 ip address 172.20.40.254 255.255.255.0
 no shutdown
```

### Issue Faced — Wrong Command

When configuring SVIs, the wrong command was typed:

```
! Wrong:
ip add 172.20.10.254 255.255.255.0

! Correct:
ip address 172.20.10.254 255.255.255.0
```

Cisco IOS CLI accepts abbreviations like `ip addr` but not `ip add`. Always use `ip address` or at minimum `ip addr`.

---

## Verification Commands

```
show vlan brief
```
Shows all VLANs and which access ports are assigned to each VLAN.

```
show interfaces vlan 10
```
Shows SVI status — must be up/up for inter-VLAN routing to work.

```
show ip interface brief | include Vlan
```
Quick status of all SVI interfaces.

```
show ip route
```
Should show connected routes for each VLAN subnet (marked `C`).

```
ping 172.20.20.1 source 172.20.10.254
```
Test inter-VLAN routing between SVIs.

---

## Common VLAN Issues

| Problem | Likely Cause | Fix |
|---|---|---|
| SVI is down/down | VLAN not created in VLAN database | Create VLAN with `vlan X` command |
| SVI is up/down | No active access port in that VLAN | Connect a device to an access port in that VLAN |
| Inter-VLAN traffic not routing | `ip routing` not enabled | Run `ip routing` on the Layer 3 switch |
| Devices can't communicate across switches | Trunk not configured | Set trunk between switches with `switchport mode trunk` |
| VLAN missing on trunk | Trunk pruned | Add VLAN with `switchport trunk allowed vlan add X` |
