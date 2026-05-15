# IPsec — IP Security

## What Is IPsec?

**IPsec** is a suite of protocols that provides **security at the network layer (Layer 3)**. It authenticates and encrypts each IP packet in a data stream.

GRE tunnels carry traffic, but they provide **no encryption** — anyone who can intercept the packets can read them. IPsec solves this by wrapping GRE traffic in an encrypted, authenticated envelope.

Together, **GRE + IPsec** is the standard enterprise site-to-site VPN approach:
- GRE handles multicast and routing protocols
- IPsec handles encryption and integrity

---

## IPsec Is a Framework, Not a Single Protocol

IPsec is composed of multiple protocols working together:

| Component | Purpose |
|---|---|
| **IKE / ISAKMP** | Key exchange and SA negotiation |
| **AH (Authentication Header)** | Integrity + authentication (no encryption) |
| **ESP (Encapsulating Security Payload)** | Encryption + integrity + authentication |

In practice, **ESP is almost always used** — AH alone doesn't encrypt, so it's rarely sufficient.

---

## How IPsec Works — Two Phases

IPsec negotiation happens in **two phases**:

### Phase 1 — ISAKMP SA (Management Channel)
This phase establishes a **secure, authenticated channel** between the two peers. It is used only for negotiating Phase 2 — not for carrying actual data traffic.

Both sides negotiate:
- **Encryption algorithm** (AES, 3DES, DES)
- **Hash algorithm** (SHA, MD5)
- **Authentication method** (pre-shared key or certificates)
- **Diffie-Hellman group** (determines key strength)
- **Lifetime** of the SA

Once agreed, a **ISAKMP SA** is established. You can verify this with:
```
show crypto isakmp sa
```

### Phase 2 — IPsec SA (Data Channel)
Once Phase 1 is complete, Phase 2 negotiates the actual **IPsec Security Associations (SAs)** that will protect the data traffic.

Phase 2 negotiates:
- **IPsec protocol** (AH or ESP — we use ESP)
- **Encryption and hash algorithms** for data traffic
- **Traffic selectors** — which traffic to encrypt (from the ACL)
- **Lifetime** of the SA

Two unidirectional SAs are created — one for each direction of traffic.

```
show crypto ipsec sa
```

---

## Key IPsec Concepts

### Security Association (SA)
An SA is a **one-way agreement** between two peers about how to protect traffic — which algorithms, keys, and parameters to use. IPsec always creates SAs in pairs (one for each direction).

### Security Parameter Index (SPI)
A 32-bit number that identifies an SA. When a packet arrives, the receiver uses the SPI to look up the correct SA and decrypt/verify the packet.

### Transform Set
Defines the encryption and hashing algorithms used in Phase 2. Example:
```
crypto ipsec transform-set VPN esp-aes esp-sha-hmac
```
- `esp-aes` — Encrypt with AES
- `esp-sha-hmac` — Authenticate with SHA HMAC

### Crypto Map
Ties everything together — which peer, which transform set, and which ACL to match:
```
crypto map CMAP 10 ipsec-isakmp
 set peer 192.168.12.2
 set transform-set VPN
 match address 100
```

---

## IPsec Modes — Tunnel vs Transport

| Mode | What's Encrypted | Use Case |
|---|---|---|
| **Tunnel Mode** | Entire original IP packet (new outer IP added) | Site-to-site VPN |
| **Transport Mode** | Only the payload (original IP header preserved) | Host-to-host, used with GRE |

In this lab, **transport mode** is used because GRE already provides the outer IP header. IPsec in transport mode encrypts the GRE packet payload (the inner IP + data).

```
crypto ipsec transform-set VPN esp-aes esp-sha-hmac
 mode transport
```

---

## Configuration Applied

### Step 1 — ISAKMP Policy (Phase 1)

Same on both R1 and R2:
```
crypto isakmp policy 10
 encryption aes 256
 hash sha256
 authentication pre-share
 group 2
```

- `aes 256` — AES with 256-bit key for Phase 1 encryption
- `sha256` — SHA-256 for integrity
- `pre-share` — Use pre-shared keys for authentication
- `group 2` — Diffie-Hellman group 2 (1024-bit) for key exchange

### Step 2 — Pre-Shared Keys

```
! On R1:
crypto isakmp key cisco address 192.168.12.2

! On R2:
crypto isakmp key cisco address 192.168.12.1
```

The key `cisco` must match on both sides. The address is the peer's physical IP.

### Step 3 — Transform Set (Phase 2 Algorithms)

```
crypto ipsec transform-set VPN esp-aes esp-sha-hmac
```

### Step 4 — ACL (Traffic to Encrypt)

```
access-list 100 permit gre host 192.168.12.1 host 192.168.12.2
```

This ACL says: encrypt GRE traffic between R1's and R2's physical IPs. Only GRE traffic (the tunnel) gets encrypted — not all traffic on the interface.

### Step 5 — Crypto Map

```
crypto map CMAP 10 ipsec-isakmp
 set peer 192.168.12.2
 set transform-set VPN
 match address 100
```

### Step 6 — Apply to Interface

```
interface g3/0
 crypto map CMAP
```

The crypto map must be applied on the **physical interface** — not the tunnel interface. The GRE packets exit via G3/0, and that's where IPsec inspects and encrypts them.

---

## How GRE + IPsec Flow Works

```
1. PC in HQ sends data to Branch
2. Data routed to R1's tunnel0 interface
3. R1 encapsulates in GRE (adds outer IP: src 192.168.12.1, dst 192.168.12.2)
4. GRE packet exits G3/0 — crypto map triggers
5. ACL 100 matches (GRE from .12.1 to .12.2) → IPsec kicks in
6. IPsec encrypts the GRE packet with AES-256
7. Encrypted packet sent to R2
8. R2 decrypts (IPsec) → decapsulates (GRE) → delivers inner packet
```

---

## Verification Commands

```
show crypto isakmp sa
```
Expected: `QM_IDLE` state means Phase 1 is up and idle (healthy).

```
show crypto ipsec sa
```
Look for increasing `encaps` and `decaps` packet counters — means traffic is flowing and being encrypted/decrypted.

```
show crypto map
```
Shows the crypto map configuration and which interface it's applied to.

---

## Common IPsec Issues

| Problem | Likely Cause | Fix |
|---|---|---|
| Phase 1 won't establish | Policy mismatch or pre-shared key mismatch | Match policy exactly on both sides |
| Phase 2 won't establish | Transform set mismatch or ACL issue | Match transform set; verify ACL |
| Phase 1 up but no data flows | ACL not matching actual traffic | Check `match address` ACL entries |
| Packet counter not increasing | Wrong interface for crypto map | Apply map on physical egress interface |
| Tunnel works but slow | DH group too weak or CPU load | Upgrade to group 14/19/20 for production |
