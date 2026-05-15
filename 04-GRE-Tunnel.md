# GRE Tunnel — Generic Routing Encapsulation

## What Is a GRE Tunnel?

A **GRE tunnel** is a logical point-to-point connection that encapsulates one network protocol inside another. It creates a **virtual link** between two routers across an existing IP network — as if they were directly connected.

GRE was developed by Cisco and is defined in **RFC 2784**.

---

## Why Do We Need Tunnels?

Consider a simple scenario: HQ and Branch are connected over the internet (or a WAN). You want:
- Dynamic routing protocols (OSPF, EIGRP) to run between HQ and Branch
- Private IP traffic to flow between the two sites
- All of this over an existing public/routed IP path

Problem: OSPF uses multicast (224.0.0.5 / 224.0.0.6). If there's a router or ISP in between that doesn't run OSPF, multicast won't pass through, and the OSPF session can't form.

**GRE solves this** by creating a tunnel interface. OSPF sees the tunnel interface as a direct point-to-point link, and GRE handles encapsulating the multicast packets inside unicast IP packets that cross the WAN.

---

## How GRE Encapsulation Works

When a packet is sent over a GRE tunnel, the original packet is wrapped with:

```
[ Original IP Header ] [ Original Payload ]
            ↓  GRE encapsulation
[ Outer IP Header ] [ GRE Header ] [ Original IP Header ] [ Original Payload ]
```

- **Outer IP Header** — Uses the physical tunnel source and destination IPs (the real routable addresses)
- **GRE Header** — 4 bytes minimum; indicates the protocol type of the encapsulated packet
- **Inner IP Header + Payload** — The original packet, completely untouched

The receiving router strips the outer IP and GRE headers and processes the inner packet normally.

### GRE Header Format
```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|C|       Reserved0       | Ver |         Protocol Type         |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|      Checksum (optional)      |       Reserved1 (optional)    |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

Protocol Type 0x0800 = IPv4 being encapsulated.

---

## GRE Tunnel Properties

| Property | Value |
|---|---|
| Protocol Number | IP Protocol 47 |
| Overhead | 24 bytes (20 IP + 4 GRE) |
| MTU | Reduces effective MTU by 24 bytes (1500 - 24 = 1476 bytes) |
| Multicast support | Yes ✅ |
| Encryption | No ❌ (that's what IPsec is for) |
| Any protocol support | Yes — can carry IPv4, IPv6, non-IP |

---

## GRE in This Lab

In this lab, a GRE tunnel connects **R1 (HQ Core)** and **R2 (Branch Router)**. The tunnel runs over the physical link between them (192.168.12.0/24).

```
R1                        R2
Physical: 192.168.12.1    Physical: 192.168.12.2
Tunnel:   10.10.10.1      Tunnel:   10.10.10.2
```

The tunnel gives R1 and R2 a logical direct link (10.10.10.0/24) even though in a real scenario they'd be separated by a WAN. This tunnel interface is then protected by IPsec encryption.

---

## Configuration Applied

### R1
```
interface tunnel0
 ip address 10.10.10.1 255.255.255.0
 tunnel source 192.168.12.1
 tunnel destination 192.168.12.2
```

### R2
```
interface tunnel0
 ip address 10.10.10.2 255.255.255.0
 tunnel source 192.168.12.2
 tunnel destination 192.168.12.1
```

**Configuration breakdown:**
- `ip address` — The tunnel interface gets its own IP address (used for routing over the tunnel)
- `tunnel source` — The physical IP on this router that GRE will use as the outer source IP
- `tunnel destination` — The physical IP of the remote router (outer destination IP)

By default, `tunnel mode gre ip` is already the default tunnel mode on Cisco IOS — no need to specify it unless using a different mode (like GRE IPv6).

---

## MTU Consideration

GRE adds 24 bytes of overhead. If the underlying physical link has MTU 1500, the effective MTU through the tunnel is **1476 bytes** (1500 - 20 IP - 4 GRE = 1476).

This can cause issues with large packets. Solutions:
- Set the tunnel interface MTU: `ip mtu 1476`
- Enable TCP MSS clamping: `ip tcp adjust-mss 1452`

---

## Verification Commands

```
show interfaces tunnel0
```
Check the tunnel is up/up. If the tunnel is up but the line protocol is down, the physical path between the tunnel endpoints may be broken.

```
show ip interface brief | include Tunnel
```
Quick status of all tunnel interfaces.

```
ping 10.10.10.2 source 10.10.10.1
```
Ping the far-end tunnel IP to confirm the GRE tunnel itself is working.

---

## Common GRE Issues

| Problem | Likely Cause | Fix |
|---|---|---|
| Tunnel is up but no traffic | Routing issue — don't know how to reach tunnel destination | Add route for tunnel destination via physical interface |
| Recursive routing loop | Tunnel traffic routed back through tunnel | Add specific static route for tunnel destination via physical next-hop |
| Large packets dropped | MTU mismatch | Set `ip mtu 1476` and `ip tcp adjust-mss 1452` on tunnel |
| Tunnel line protocol down | Physical path unreachable | Verify `tunnel source` IP is reachable and up |

---

## GRE vs Other Tunnel Types

| Tunnel Type | Encryption | Overhead | Use Case |
|---|---|---|---|
| **GRE** | None | 24 bytes | Carry multicast/routing protocols |
| **IPsec (transport)** | Yes | ~50 bytes | Encrypt IP traffic directly |
| **GRE + IPsec** | Yes (on GRE) | ~58 bytes | Best of both — routing protocols + encryption |
| **DMVPN** | Yes (mGRE + IPsec) | Variable | Scalable hub-and-spoke VPN |

This lab uses **GRE + IPsec** — the most common enterprise site-to-site VPN pattern.
