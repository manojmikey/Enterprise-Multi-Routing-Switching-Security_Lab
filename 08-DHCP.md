# DHCP — Dynamic Host Configuration Protocol

## What Is DHCP?

**DHCP (Dynamic Host Configuration Protocol)** automatically assigns network configuration to devices when they join a network. Without DHCP, every PC, phone, printer, and IoT device would need to be manually configured with:

- IP address
- Subnet mask
- Default gateway
- DNS server

In an enterprise with hundreds of devices, manual configuration is impractical and error-prone. DHCP automates this entirely.

DHCP uses **UDP** — port **67** (server) and port **68** (client). It operates at Layer 7 (application layer) but relies on Layer 2/3 broadcast for initial discovery.

---

## The DHCP Process — DORA

DHCP uses a 4-packet handshake called **DORA**:

```
Client                          Server
  |                                |
  |——— DISCOVER (broadcast) ——————→|   "Is there a DHCP server here?"
  |                                |
  |←—— OFFER (unicast/broadcast) ——|   "Yes, here's an available IP"
  |                                |
  |——— REQUEST (broadcast) ————————→|   "I'd like that IP, please"
  |                                |
  |←—— ACKNOWLEDGMENT (unicast) ———|   "Confirmed, it's yours"
  |                                |
```

### 1. DHCP Discover
The client has no IP yet (source IP: `0.0.0.0`). It broadcasts a Discover packet to `255.255.255.255` to find any available DHCP server.

Key fields:
- DHCP Message Type: 1 (Discover)
- Client Identifier (MAC address)
- Hostname (optional)

### 2. DHCP Offer
The server sees the Discover, selects an available IP from its pool, and sends an Offer back.

Key fields:
- DHCP Message Type: 2 (Offer)
- Server Identifier (server's IP)
- Offered IP address
- IP lease time
- Subnet mask, default router, DNS server

### 3. DHCP Request
The client broadcasts a Request to formally ask for the offered IP. It's a broadcast because there might be multiple DHCP servers — the Request tells all servers which one's offer was accepted (using the **Server Identifier** option). The other servers release their offered IPs back to their pools.

Key fields:
- DHCP Message Type: 3 (Request)
- Requested IP Address (Option 50)
- Server Identifier (Option 54) — which server's offer is accepted

### 4. DHCP Acknowledgment (ACK)
The server confirms the lease with an ACK. The client can now use the IP.

After receiving the ACK, the client sends a **Gratuitous ARP** to check if any other device is already using that IP (duplicate IP detection). If no one replies, the IP is used.

---

## DHCP Lease and Renewal

The IP assignment is not permanent — it's a **lease** with an expiry time.

- At **50% of lease time**: Client sends a unicast Request directly to the server to renew
- At **87.5% of lease time**: Client broadcasts a Request (in case the original server is unreachable)
- At **100% (lease expiry)**: Client must release the IP and start DORA again from scratch

---

## DHCP Options

DHCP packets carry configuration data in **options** — TLV (Type-Length-Value) encoded fields:

| Option | Meaning |
|---|---|
| 1 | Subnet Mask |
| 3 | Default Router (gateway) |
| 6 | DNS Server |
| 12 | Hostname |
| 50 | Requested IP Address |
| 51 | IP Lease Time |
| 53 | DHCP Message Type |
| 54 | Server Identifier |
| 61 | Client Identifier |
| 82 | Relay Agent Information |

---

## DHCP Relay Agent

**Problem:** DHCP Discover is a broadcast. Routers don't forward broadcasts between subnets. So if your DHCP server is centralized in one subnet, clients in other subnets can't reach it by broadcast.

**Solution: DHCP Relay Agent (ip helper-address)**

A router interface is configured with the DHCP server's IP. When the router receives a DHCP broadcast on that interface, it:
1. Changes the source IP to its own interface IP
2. Adds **Option 82** (relay agent info) with the client's subnet info
3. Forwards the packet as **unicast** directly to the DHCP server

The DHCP server uses the relay agent's IP (or Option 82 info) to determine which IP pool to assign from.

```
ip helper-address <dhcp-server-IP>
```

This command goes on the **client-facing interface** of the router — not the server-facing side.

---

## DHCP in This Lab

DHCP was configured for the **Branch Office** to automatically assign IPs to PC9, PC10, and PC11 (connected to IOU3 switch). R2 acts as the DHCP server for the branch network (192.168.1.0/24).

The gateway is R2's G1/0 interface (`192.168.1.254`), which is also the HSRP virtual IP for the branch.

---

## Configuration Applied

```
! Exclude the gateway and any static IPs from the pool
ip dhcp excluded-address 172.20.10.1 172.20.10.10

! Create the pool
ip dhcp pool VLAN10
 network 172.20.10.0 255.255.255.0
 default-router 172.20.10.1
 dns-server 8.8.8.8
```

For all VLANs:

```
ip dhcp excluded-address 172.20.10.1 172.20.10.10
ip dhcp excluded-address 172.20.20.1 172.20.20.10
ip dhcp excluded-address 172.20.30.1 172.20.30.10
ip dhcp excluded-address 172.20.40.1 172.20.40.10

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
```

**Configuration breakdown:**
- `excluded-address` — Reserves these IPs so DHCP won't hand them out (use for gateway, servers, printers with static IPs)
- `network` — The subnet from which IPs will be assigned
- `default-router` — The gateway clients will receive (matches the HSRP virtual IP)
- `dns-server` — Google DNS (8.8.8.8) used here; production would use internal DNS

---

## APIPA — What Happens When DHCP Fails

If a client can't reach a DHCP server, it assigns itself an **APIPA (Automatic Private IP Addressing)** address in the `169.254.0.0/16` range. This allows communication with other APIPA devices on the same segment, but no internet access and no routing possible.

Seeing a `169.254.x.x` address on a device is a clear sign the device couldn't reach a DHCP server.

---

## Verification Commands

```
show ip dhcp binding
```
Shows all current IP-to-MAC leases — which device has been assigned which IP.

```
show ip dhcp pool
```
Shows pool configuration and utilization — how many IPs are available, how many are leased.

```
show ip dhcp conflict
```
Lists any IP conflicts detected (from gratuitous ARP responses). DHCP stops offering conflicting IPs until conflicts are cleared.

```
show running-config | section dhcp
```
View the DHCP configuration in the running config.

---

## Common DHCP Issues

| Problem | Likely Cause | Fix |
|---|---|---|
| Client gets 169.254.x.x | Can't reach DHCP server | Check connectivity; check relay agent config |
| DHCP pool exhausted | Too many clients or short lease time | Increase pool size or adjust excluded range |
| Wrong gateway assigned | Wrong `default-router` in pool | Fix `default-router` to match correct VIP |
| Client in remote subnet not getting IP | No relay agent | Configure `ip helper-address` on router interface |
| Duplicate IPs | Manual IPs conflicting with DHCP pool | Exclude manually assigned IPs with `excluded-address` |
