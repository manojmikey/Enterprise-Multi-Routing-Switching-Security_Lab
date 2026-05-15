# WALKTHROUGH — Enterprise Network Lab: Complete Step-by-Step Build

This document covers the complete configuration of the Enterprise Network Lab from scratch — every command, every protocol, every issue hit along the way, and how it was resolved.

---

## Lab Overview

**Simulator:** GNS3 / EVE-NG  
**Devices used:** 5 Routers (R1–R5), 2 Layer 3 Switches (IOU1/SW1, IOU2/SW2), 1 Layer 2 Switch (IOU3), 11 PCs (VPCS)

### Network Zones

| Zone | Devices | Purpose |
|---|---|---|
| Headquarters (HQ) | R1, SW1, SW2, IOU4, PC1–PC8 | Core enterprise site with OSPF, VLANs, HSRP |
| Branch Office | R2, IOU3, PC9–PC11 | Remote site connected via GRE+IPsec VPN |
| AS100 Backbone | R3, R4 | Internal BGP routers |
| External AS | R5 | Simulates internet / external AS200 |

---

## STEP 1 — Configure IP Addressing on All Routers

The first step is always interface configuration. Every router needs IPs before any protocol will work.

### R1 (Headquarters Core Router)

```
enable
configure terminal
hostname R1

interface g1/0
 ip address 11.11.13.1 255.255.255.0
 no shutdown

interface g2/0
 ip address 11.11.14.1 255.255.255.0
 no shutdown

interface g3/0
 ip address 192.168.12.1 255.255.255.0
 no shutdown

interface loopback0
 ip address 1.1.1.1 255.255.255.255

end
```

### R2 (Branch Router)

```
enable
configure terminal
hostname R2

interface g1/0
 ip address 192.168.1.254 255.255.255.0
 no shutdown

interface g2/0
 ip address 192.168.12.2 255.255.255.0
 no shutdown

interface loopback0
 ip address 2.2.2.2 255.255.255.255

end
```

### R3 (AS100 — eBGP peer to R5)

```
enable
configure terminal
hostname R3

interface g1/0
 ip address 11.11.35.3 255.255.255.0
 no shutdown

interface g2/0
 ip address 11.11.34.3 255.255.255.0
 no shutdown

interface g3/0
 ip address 11.11.13.3 255.255.255.0
 no shutdown

end
```

### R4 (AS100 — iBGP peer to R3)

```
enable
configure terminal
hostname R4

interface g1/0
 ip address 11.11.45.4 255.255.255.0
 no shutdown

interface g2/0
 ip address 11.11.34.4 255.255.255.0
 no shutdown

interface g3/0
 ip address 11.11.14.4 255.255.255.0
 no shutdown

end
```

### R5 (AS200 — External Router)

```
enable
configure terminal
hostname R5

interface g1/0
 ip address 11.11.35.5 255.255.255.0
 no shutdown

interface g2/0
 ip address 11.11.45.5 255.255.255.0
 no shutdown

interface loopback0
 ip address 5.5.5.5 255.255.255.255

end
```

### Verification

```
show ip interface brief     ← All configured interfaces should show up/up
```

---

## STEP 2 — Configure OSPF

OSPF is the internal routing protocol for the HQ backbone. R1, R3, and R4 all participate in OSPF Area 0.

**Why OSPF?** Without it, R1 would need static routes to reach R3 and R4's networks — and those routes would have to be manually updated any time the network changes. OSPF handles this dynamically.

### R1

```
configure terminal

router ospf 1
 router-id 1.1.1.1
 network 11.11.13.0 0.0.0.255 area 0
 network 11.11.14.0 0.0.0.255 area 0
 network 192.168.12.0 0.0.0.255 area 0

end
```

### R3

```
configure terminal

router ospf 1
 router-id 3.3.3.3
 network 11.11.13.0 0.0.0.255 area 0

end
```

### R4

```
configure terminal

router ospf 1
 router-id 4.4.4.4
 network 11.11.14.0 0.0.0.255 area 0

end
```

### Verification

```
show ip ospf neighbor
```

Expected — R1 should show R3 and R4 as neighbors in **Full** state:

```
Neighbor ID   Pri   State       Dead Time   Address         Interface
3.3.3.3         1   FULL/DR     00:00:38    11.11.13.3      GigabitEthernet1/0
4.4.4.4         1   FULL/BDR    00:00:36    11.11.14.4      GigabitEthernet2/0
```

```
show ip route ospf          ← Look for 'O' routes in the routing table
show ip ospf database       ← View the Link-State Database
```

---

## STEP 3 — Configure iBGP Between R3 and R4

R3 and R4 are both in AS100 (the enterprise AS). They need to share BGP routes with each other internally — this is iBGP.

### R3

```
configure terminal

router bgp 100
 neighbor 11.11.34.4 remote-as 100

end
```

### R4

```
configure terminal

router bgp 100
 neighbor 11.11.34.3 remote-as 100

end
```

### Issue — iBGP Neighbor Stuck in ACTIVE State

**Symptom:** `show ip bgp summary` showed `Active` instead of `Established`.

`Active` means BGP is trying to open a TCP connection to the peer but can't. BGP runs over TCP port 179.

**Troubleshooting steps:**

```
ping 11.11.34.4             ← Test basic reachability from R3
show ip route               ← Is 11.11.34.0/24 in the routing table?
show ip interface brief     ← Is G2/0 up/up on both R3 and R4?
```

**Root cause:** In this case, reachability existed but the neighbor IP needed to match exactly what was configured. After verifying both sides had matching `remote-as` and reachable neighbor IPs, the session came up.

### Verification

```
show ip bgp summary         ← State should show a number (prefixes received), not "Active"
show ip bgp neighbors 11.11.34.4
```

---

## STEP 4 — Configure eBGP Between R3 and R5

R5 is in AS200 — a completely separate autonomous system. R3 peers with R5 using eBGP to exchange routes between the two ASes.

### R3

```
configure terminal

router bgp 100
 neighbor 11.11.35.5 remote-as 200

end
```

### R5

```
configure terminal

router bgp 200
 bgp router-id 5.5.5.5
 neighbor 11.11.35.3 remote-as 100

end
```

### Verification

```
show ip bgp summary         ← Both peers should show Established
```

---

## STEP 5 — Advertise Networks in BGP

Just forming BGP sessions doesn't advertise routes. BGP only advertises what you explicitly tell it to.

### R3 — Advertise HQ Networks and Redistribute OSPF

```
configure terminal

router bgp 100
 network 11.11.13.0 mask 255.255.255.0
 redistribute ospf 1

end
```

`network 11.11.13.0 mask 255.255.255.0` — Tells BGP to advertise this exact prefix to all peers. The prefix must exist in the routing table.

`redistribute ospf 1` — Takes all OSPF-learned routes and puts them into BGP so they get advertised to R5.

### R5 — Advertise Its Own Networks

```
configure terminal

router bgp 200
 network 5.5.5.5 mask 255.255.255.255
 network 11.11.35.0 mask 255.255.255.0

end
```

### Issue — R5 Wasn't Learned by R3

**Confusion:** R5 had the neighbor configured but wasn't advertising anything to R3.

**Cause:** The `network` command was missing on R5. BGP does not auto-advertise connected interfaces like OSPF does. You must explicitly tell BGP what to advertise.

After adding `network 5.5.5.5 mask 255.255.255.255` on R5, R3's BGP table showed:

```
show ip bgp
```
```
B    5.5.5.5/32 [20/0] via 11.11.35.5
```

### Issue — R5 Could Ping R3 But Not R1

**Symptom:** R5 had a BGP route to `11.11.13.0/24` (R1's network) but ping to R1 failed.

**Explanation:** BGP only gives you the **forward path**. R5 knew how to send packets toward 11.11.13.1 (via R3). But R1 had no route back to R5's source IPs (11.11.35.x or 11.11.45.x). Return traffic was dropped at R1.

**Solution:** Redistribute BGP routes into OSPF on R1 so all internal OSPF routers learn about externally reachable networks:

```
configure terminal

router ospf 1
 redistribute bgp 100 subnets

end
```

> The `subnets` keyword is critical — without it, OSPF only redistributes classful networks (e.g., /24 instead of /32 loopbacks).

### Verification

```
show ip bgp                 ← Full BGP table; '>' marks best routes installed in RIB
show ip route bgp           ← Shows only BGP routes (marked 'B')
ping 5.5.5.5 source 11.11.13.1     ← Confirm end-to-end reachability
```

---

## STEP 6 — Configure GRE Tunnel

A GRE tunnel creates a virtual point-to-point link between R1 (HQ) and R2 (Branch). This allows routing protocols and private traffic to flow between the two sites as if they were directly connected.

### R1

```
configure terminal

interface tunnel0
 ip address 10.10.10.1 255.255.255.0
 tunnel source 192.168.12.1
 tunnel destination 192.168.12.2

end
```

### R2

```
configure terminal

interface tunnel0
 ip address 10.10.10.2 255.255.255.0
 tunnel source 192.168.12.2
 tunnel destination 192.168.12.1

end
```

### Verification

```
show interfaces tunnel0
```

The tunnel should show **up/up** (line protocol up = physical path to tunnel endpoint reachable).

```
ping 10.10.10.2 source 10.10.10.1       ← Ping far-end tunnel IP
```

---

## STEP 7 — Configure IPsec VPN

The GRE tunnel carries traffic but doesn't encrypt it. IPsec adds encryption and integrity. Together, GRE+IPsec is the standard enterprise site-to-site VPN.

IPsec negotiates in **two phases**:
- **Phase 1 (ISAKMP):** Secure management channel, authenticates peers
- **Phase 2 (IPsec SA):** Negotiates actual data encryption parameters

### Step 7a — ISAKMP Policy (Phase 1) on R1 and R2

```
configure terminal

crypto isakmp policy 10
 encryption aes 256
 hash sha256
 authentication pre-share
 group 2

end
```

Same command on **both R1 and R2** — both sides must match exactly.

### Step 7b — Pre-Shared Keys

```
! On R1:
configure terminal
crypto isakmp key cisco address 192.168.12.2
end

! On R2:
configure terminal
crypto isakmp key cisco address 192.168.12.1
end
```

The key `cisco` must be identical on both sides. The address is the **peer's physical IP**.

### Step 7c — Transform Set (Phase 2 Algorithms)

```
! On both R1 and R2:
configure terminal
crypto ipsec transform-set VPN esp-aes esp-sha-hmac
 mode transport
end
```

`mode transport` is used because GRE already provides the outer IP header. IPsec in transport mode only encrypts the GRE payload — more efficient than tunnel mode here.

### Step 7d — ACL (Define What Traffic to Encrypt)

```
! On R1:
access-list 100 permit gre host 192.168.12.1 host 192.168.12.2

! On R2:
access-list 100 permit gre host 192.168.12.2 host 192.168.12.1
```

This ACL says: encrypt all GRE traffic between R1 and R2's physical IPs.

### Step 7e — Crypto Map

```
! On R1:
configure terminal

crypto map CMAP 10 ipsec-isakmp
 set peer 192.168.12.2
 set transform-set VPN
 match address 100

interface g3/0
 crypto map CMAP

end

! On R2:
configure terminal

crypto map CMAP 10 ipsec-isakmp
 set peer 192.168.12.1
 set transform-set VPN
 match address 100

interface g2/0
 crypto map CMAP

end
```

> **Critical:** The crypto map must be applied to the **physical interface** (G3/0 on R1, G2/0 on R2), not the tunnel interface. GRE packets exit via the physical interface, and that's where IPsec intercepts and encrypts them.

### Verification

```
show crypto isakmp sa       ← Should show QM_IDLE (Phase 1 up and healthy)
show crypto ipsec sa        ← Check 'encaps' and 'decaps' counters are increasing
show crypto map             ← View crypto map config and interface assignment
```

---

## STEP 8 — Configure VLANs

VLANs segment the HQ network into separate broadcast domains by department/group.

### On SW1 (IOU1) and SW2 (IOU2) — Create VLANs

```
configure terminal

vlan 10
 name VLAN10_USERS
vlan 20
 name VLAN20_USERS
vlan 30
 name VLAN30_USERS
vlan 40
 name VLAN40_USERS

end
```

### Configure Access Ports (PC-facing ports)

```
! Example: PC1 and PC2 in VLAN 10
configure terminal

interface range eth0/0 - 1
 switchport mode access
 switchport access vlan 10

end
```

### Configure Trunk Ports (Between switches)

```
configure terminal

interface eth0/0
 switchport mode trunk
 switchport trunk allowed vlan 10,20,30,40

end
```

### Verification

```
show vlan brief             ← See all VLANs and which ports are in each
show interfaces trunk       ← Verify trunk links and allowed VLANs
```

---

## STEP 9 — Configure SVIs (Inter-VLAN Routing)

SVIs (Switched Virtual Interfaces) are Layer 3 virtual interfaces on the switches — they act as the default gateway for each VLAN.

### SW1 (IOU1) — SVIs for VLAN 10 and 20

```
configure terminal

ip routing          ← CRITICAL: Enables Layer 3 routing on the switch

interface vlan10
 ip address 172.20.10.254 255.255.255.0
 no shutdown

interface vlan20
 ip address 172.20.20.254 255.255.255.0
 no shutdown

end
```

### SW2 (IOU2) — SVIs for VLAN 30 and 40

```
configure terminal

ip routing

interface vlan30
 ip address 172.20.30.254 255.255.255.0
 no shutdown

interface vlan40
 ip address 172.20.40.254 255.255.255.0
 no shutdown

end
```

### Issue — Wrong Command Entered

During SVI configuration, the wrong abbreviated command was typed:

```
! Wrong (not recognized by IOS):
ip add 172.20.10.254 255.255.255.0

! Correct:
ip address 172.20.10.254 255.255.255.0
```

Cisco IOS accepts `ip addr` (short form) but not `ip add`. Always use `ip address` to be safe.

### Verification

```
show ip interface brief | include Vlan    ← SVI status (must be up/up)
show ip route connected                   ← Should show 172.20.x.0 networks as 'C'
```

---

## STEP 10 — Configure HSRP (Gateway Redundancy)

HSRP provides default gateway redundancy. SW1 and SW2 both have SVIs for VLAN 10 and 20. HSRP makes SW1 the active gateway while SW2 is the standby — automatically taking over if SW1 fails.

### SW1 (IOU1) — Active for VLAN 10 and 20

```
configure terminal

interface vlan10
 standby 10 ip 172.20.10.1
 standby 10 priority 111
 standby 10 preempt

interface vlan20
 standby 20 ip 172.20.20.1
 standby 20 priority 111
 standby 20 preempt

end
```

### SW2 (IOU2) — Standby for VLAN 10 and 20

```
configure terminal

interface vlan10
 standby 10 ip 172.20.10.1
 standby 10 priority 90
 standby 10 preempt

interface vlan20
 standby 20 ip 172.20.20.1
 standby 20 priority 90
 standby 20 preempt

end
```

The virtual IPs (`172.20.10.1`, `172.20.20.1`) are what end devices use as their default gateway. SW1 wins the election with priority 111 vs SW2's 90.

### Verification

```
show standby brief          ← Quick view: Active/Standby status, virtual IP
show standby vlan 10        ← Detailed info for VLAN 10 HSRP group
```

Expected output on SW1:

```
Interface   Grp  Pri P State   Active          Standby         Virtual IP
Vl10        10   111 P Active  local           172.20.10.254   172.20.10.1
Vl20        20   111 P Active  local           172.20.20.254   172.20.20.1
```

---

## STEP 11 — Configure DHCP

DHCP automatically assigns IP addresses to PCs. Configured on R2 for the branch office, and optionally on the core switches for HQ VLANs.

### DHCP for All VLANs

```
configure terminal

! Exclude static IPs from pool (gateway, servers, printers)
ip dhcp excluded-address 172.20.10.1 172.20.10.10
ip dhcp excluded-address 172.20.20.1 172.20.20.10
ip dhcp excluded-address 172.20.30.1 172.20.30.10
ip dhcp excluded-address 172.20.40.1 172.20.40.10

! Create pools for each VLAN
ip dhcp pool VLAN10
 network 172.20.10.0 255.255.255.0
 default-router 172.20.10.1
 dns-server 8.8.8.8

ip dhcp pool VLAN20
 network 172.20.20.0 255.255.255.0
 default-router 172.20.20.1
 dns-server 8.8.8.8

ip dhcp pool VLAN30
 network 172.20.30.0 255.255.255.0
 default-router 172.20.30.1
 dns-server 8.8.8.8

ip dhcp pool VLAN40
 network 172.20.40.0 255.255.255.0
 default-router 172.20.40.1
 dns-server 8.8.8.8

end
```

### For Branch PCs (on R2)

```
configure terminal

ip dhcp excluded-address 192.168.1.254

ip dhcp pool BRANCH
 network 192.168.1.0 255.255.255.0
 default-router 192.168.1.254
 dns-server 8.8.8.8

end
```

### Verification

```
show ip dhcp binding        ← See which IPs are leased to which MACs
show ip dhcp pool           ← Pool utilization stats
show ip dhcp conflict       ← Any IP conflicts detected
```

---

## Final Verification — End-to-End Testing

### Check All OSPF Neighbors

```
show ip ospf neighbor       ← R1 should show R3 and R4 in Full state
```

### Check All BGP Sessions

```
show ip bgp summary         ← All peers should show Established (number, not "Active")
show ip bgp                 ← Full BGP table — check for R5's 5.5.5.5 route
```

### Check Routing Tables

```
show ip route               ← Should see O (OSPF), B (BGP), C (connected) routes
```

### Check GRE + IPsec

```
show interfaces tunnel0
show crypto isakmp sa       ← QM_IDLE = healthy Phase 1
show crypto ipsec sa        ← Verify encaps/decaps counters increment
ping 10.10.10.2 source 10.10.10.1
```

### Check HSRP

```
show standby brief          ← SW1 should show Active, SW2 Standby
```

### Check VLANs

```
show vlan brief             ← All VLANs present and ports assigned
show interfaces trunk       ← Trunk links up with correct allowed VLANs
```

### Check DHCP

```
show ip dhcp binding        ← PCs should have received IPs
```

### End-to-End Connectivity Tests

```
! From a PC in VLAN 10 (172.20.10.x):
ping 172.20.20.1            ← Inter-VLAN routing test
ping 192.168.1.254          ← Reach Branch router (through GRE tunnel)
ping 5.5.5.5                ← Reach external AS200 router (through BGP)
```

---

## Summary of All Commands by Protocol

### OSPF
| Command | Purpose |
|---|---|
| `router ospf 1` | Enter OSPF config mode |
| `router-id 1.1.1.1` | Set OSPF router ID |
| `network x.x.x.x x.x.x.x area 0` | Enable OSPF on interfaces matching this network |
| `redistribute bgp 100 subnets` | Import BGP routes into OSPF |
| `show ip ospf neighbor` | View neighbor adjacencies |
| `show ip ospf database` | View Link-State Database |
| `show ip route ospf` | View OSPF-learned routes |

### BGP
| Command | Purpose |
|---|---|
| `router bgp 100` | Enter BGP config mode for AS100 |
| `bgp router-id 5.5.5.5` | Set BGP router ID |
| `neighbor x.x.x.x remote-as 200` | Configure BGP neighbor |
| `network x.x.x.x mask y.y.y.y` | Advertise a network via BGP |
| `redistribute ospf 1` | Import OSPF routes into BGP |
| `show ip bgp summary` | View all BGP peers and status |
| `show ip bgp` | View full BGP table |
| `show ip route bgp` | View BGP-installed routes |

### GRE Tunnel
| Command | Purpose |
|---|---|
| `interface tunnel0` | Create/enter tunnel interface |
| `ip address x.x.x.x y.y.y.y` | Assign IP to tunnel |
| `tunnel source x.x.x.x` | Set physical source IP |
| `tunnel destination x.x.x.x` | Set remote endpoint IP |
| `show interfaces tunnel0` | View tunnel status |

### IPsec
| Command | Purpose |
|---|---|
| `crypto isakmp policy 10` | Create ISAKMP Phase 1 policy |
| `crypto isakmp key cisco address x.x.x.x` | Set pre-shared key |
| `crypto ipsec transform-set VPN esp-aes esp-sha-hmac` | Define Phase 2 algorithms |
| `access-list 100 permit gre ...` | Define traffic to encrypt |
| `crypto map CMAP 10 ipsec-isakmp` | Create crypto map |
| `show crypto isakmp sa` | Check Phase 1 status |
| `show crypto ipsec sa` | Check Phase 2 status + packet counters |

### VLANs
| Command | Purpose |
|---|---|
| `vlan 10` | Create VLAN |
| `switchport mode access` | Set port as access port |
| `switchport access vlan 10` | Assign port to VLAN |
| `switchport mode trunk` | Set port as trunk |
| `switchport trunk allowed vlan 10,20` | Specify allowed VLANs on trunk |
| `show vlan brief` | View VLAN database |
| `show interfaces trunk` | View trunk port status |

### SVIs / Inter-VLAN Routing
| Command | Purpose |
|---|---|
| `ip routing` | Enable Layer 3 routing on switch |
| `interface vlan 10` | Create SVI for VLAN 10 |
| `ip address x.x.x.x y.y.y.y` | Assign IP to SVI |
| `no shutdown` | Bring SVI up |
| `show ip interface brief` | Check SVI status |

### HSRP
| Command | Purpose |
|---|---|
| `standby 10 ip x.x.x.x` | Set HSRP group virtual IP |
| `standby 10 priority 111` | Set priority (higher = active) |
| `standby 10 preempt` | Allow this router to reclaim active role |
| `show standby brief` | Quick HSRP status view |
| `show standby` | Detailed HSRP info |

### DHCP
| Command | Purpose |
|---|---|
| `ip dhcp excluded-address x.x.x.x y.y.y.y` | Reserve IPs from pool |
| `ip dhcp pool NAME` | Create DHCP pool |
| `network x.x.x.x y.y.y.y` | Define pool subnet |
| `default-router x.x.x.x` | Set gateway for clients |
| `dns-server x.x.x.x` | Set DNS for clients |
| `show ip dhcp binding` | View active leases |
| `show ip dhcp pool` | View pool stats |
