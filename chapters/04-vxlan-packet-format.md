# Chapter 4: VXLAN Packet Format — Deep Dive into Encapsulation

## Why Packet Format Matters

You might think packet format is dry. It's not. When you're troubleshooting a VXLAN fabric at 2 AM and packets are being silently dropped, the answer is almost always in the headers. MTU issues, ECMP hashing problems, firewall blocks — they all live here.

For the CCIE lab, you **will** be asked to interpret packet captures. You need to know every byte.

## The Complete VXLAN Packet

Here's the full encapsulation, outer to inner:

```
┌──────────────────────────────────────────────────────────────────────┐
│ OUTER ETHERNET HEADER (14 bytes)                                      │
├──────────────────────────────────────────────────────────────────────┤
│ OUTER IP HEADER (20 bytes)                                            │
├──────────────────────────────────────────────────────────────────────┤
│ OUTER UDP HEADER (8 bytes)                                            │
├──────────────────────────────────────────────────────────────────────┤
│ VXLAN HEADER (8 bytes)                                                │
├──────────────────────────────────────────────────────────────────────┤
│ ORIGINAL (INNER) ETHERNET FRAME (variable)                            │
│   ┌────────────────────────────────────────────────────────────┐     │
│   │ Inner Ethernet Header (14 bytes)                            │     │
│   │ Inner IP Header (20 bytes)                                  │     │
│   │ Inner Transport (TCP/UDP)                                   │     │
│   │ Inner Payload                                               │     │
│   └────────────────────────────────────────────────────────────┘     │
└──────────────────────────────────────────────────────────────────────┘
```

**Total overhead: 50 bytes** (14 + 20 + 8 + 8) added to every frame.

## Layer-by-Layer Breakdown

### Outer Ethernet Header (14 bytes)

```
┌─────────────────┬─────────────────┬──────────────────┐
│ Dst MAC (6B)    │ Src MAC (6B)    │ EtherType (2B)   │
│ Next-hop MAC    │ VTEP's MAC      │ 0x0800 (IPv4)    │
└─────────────────┴─────────────────┴──────────────────┘
```

- **Dst MAC**: The MAC address of the next-hop router/switch toward the remote VTEP (resolved via ARP on the underlay)
- **Src MAC**: The local VTEP's outgoing interface MAC
- **EtherType**: 0x0800 for IPv4 underlay (0x86DD if IPv6 underlay)

This header is **rewritten at every hop** — standard Ethernet behavior. The spine replaces it with its own MAC and the next-hop's MAC.

### Outer IP Header (20 bytes)

```
┌─────┬─────┬──────────┬──────────────────────────────┐
│ Ver │ IHL │   ToS    │      Total Length             │
│  4  │  5  │  (DSCP)  │  (inner + 50 bytes)          │
├─────┴─────┼──────────┼──────────┬───────────────────┤
│ Identification       │ Flags    │ Fragment Offset   │
├──────────────────────┼──────────┴───────────────────┤
│ TTL                  │ Protocol: 17 (UDP)           │
├──────────────────────┼──────────────────────────────┤
│ Header Checksum      │                              │
├──────────────────────┼──────────────────────────────┤
│ Source IP            │ VTEP-A loopback (e.g. 10.0.0.1)│
├──────────────────────┼──────────────────────────────┤
│ Destination IP       │ VTEP-B loopback (e.g. 10.0.0.2)│
└──────────────────────┴──────────────────────────────┘
```

Critical fields:
- **Source IP**: The VTEP's loopback (NVE source-interface). This is how the remote VTEP knows who sent it.
- **Destination IP**: The remote VTEP's loopback. This is how the underlay routes it.
- **Protocol**: Always 17 (UDP).
- **TTL**: Typically 64 or 255. Decremented at each underlay hop.
- **DSCP/ToS**: Can be copied from inner packet or set by policy. Important for QoS.

### Outer UDP Header (8 bytes)

```
┌──────────────────────┬──────────────────────┐
│ Source Port (2B)     │ Dest Port (2B)       │
│ Entropy / ECMP hash  │ 4789 (VXLAN)         │
├──────────────────────┼──────────────────────┤
│ Length (2B)          │ Checksum (2B)        │
│ (UDP + VXLAN + inner)│ 0x0000 (usually)     │
└──────────────────────┴──────────────────────┘
```

**Source Port (the interesting one):**
- Range: 49152–65535 (by convention)
- Value: Computed as a **hash of the inner frame** (typically inner src/dst MAC + inner src/dst IP + inner L4 ports)
- Purpose: Provides **entropy** for ECMP hashing in the underlay
- Without this, all VXLAN packets between two VTEPs would have the same 5-tuple → all hash to the same spine link → terrible load balancing

**Destination Port:**
- **4789**: IANA-assigned VXLAN port (RFC 7348)
- Historical note: Linux initially used 8472. Some older implementations may differ. Always verify.

**Checksum:**
- Usually set to **0x0000** (no UDP checksum)
- UDP checksums over IPv4 are optional
- Computing checksums over the entire inner frame in hardware is expensive
- Some implementations do compute it (check your platform)

### VXLAN Header (8 bytes)

```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|R|R|R|R|I|R|R|R|            Reserved                           |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                VXLAN Network Identifier (VNI)       | Reserved|
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

Byte-by-byte:
- **Bits 0-3**: Reserved (R). Must be 0.
- **Bit 4 (I-flag)**: **VNI Valid**. If set to 1, the VNI field is valid. MUST be set for unicast. For multicast, may be 0 in some implementations.
- **Bits 5-7**: Reserved.
- **Bytes 1-3**: Reserved (24 bits). Must be 0 on transmit, ignored on receive.
- **Bytes 4-6**: **VNI** (24 bits). The segment identifier.
- **Byte 7**: Reserved. Must be 0.

**The I-flag is the only flag that matters.** If it's not set, the VNI is meaningless.

### Inner Frame

The original Ethernet frame, **completely unchanged**. Every bit preserved:
- Inner source/destination MAC
- Inner EtherType (could be IPv4, IPv6, ARP, anything)
- Inner VLAN tags (802.1Q) — yes, you can carry Q-in-Q inside VXLAN
- Inner IP header
- Inner L4 and payload

The inner frame is opaque to the underlay. It's just bytes inside a UDP payload.

## MTU: The Silent Killer

This is the #1 VXLAN deployment issue. Let's do the math:

```
Standard Ethernet MTU:     1500 bytes
VXLAN overhead:            +  50 bytes
─────────────────────────────────────
Required underlay MTU:     1550 bytes minimum
```

But wait — if the inner frame has an 802.1Q tag:
```
Inner frame with VLAN tag: 1504 bytes
VXLAN overhead:            +  50 bytes
─────────────────────────────────────
Required underlay MTU:     1554 bytes
```

**Best practice: Set underlay MTU to 9216 (jumbo frames).** This gives headroom for:
- VXLAN overhead (50 bytes)
- Inner VLAN tags
- Future encapsulation additions
- TCP segmentation offload (TSO) / jumbo inner frames

```
interface Ethernet1/1
  mtu 9216

system jumbomtu 9216
```

**Symptoms of MTU issues:**
- Small packets work, large packets fail (ping 100 works, ping 1500 fails)
- TCP connections establish but stall on data transfer
- "Works for SSH but not for file transfers"
- Asymmetric MTU (one direction works, other doesn't)

**Debugging:**
```
ping 10.0.0.2 size 1500 df-bit
ping vrf overlay 10.1.1.6 size 1400 df-bit
```

## ECMP Hashing and the Source Port

The UDP source port is VXLAN's secret weapon for load balancing. Here's why:

Without entropy (all packets same 5-tuple):
```
VTEP-A → VTEP-B: All packets use src=10.0.0.1, dst=10.0.0.2, sport=X, dport=4789
Result: ECMP hash → same spine → one link saturated, others idle
```

With entropy (source port varies per flow):
```
Flow 1: sport=49152 → hashes to Spine-1
Flow 2: sport=51234 → hashes to Spine-2
Flow 3: sport=53891 → hashes to Spine-3
Flow 4: sport=55012 → hashes to Spine-4
```

The hash inputs typically include:
- Inner source MAC
- Inner destination MAC
- Inner source IP
- Inner destination IP
- Inner L4 source port
- Inner L4 destination port

This means different VM-to-VM flows take different paths through the fabric. Beautiful.

**CCIE gotcha:** If you see uneven ECMP distribution, check:
1. Is the source port actually varying? (packet capture)
2. What fields does the underlay hash on? (`show policy-map interface`)
3. Are there too few flows? (elephant flow problem)

## Packet Capture Interpretation

Here's what you'd see in Wireshark for a VXLAN packet:

```
Frame 1: 1564 bytes
  Ethernet II, Src: 00:1e:49:aa:bb:01, Dst: 00:1e:49:cc:dd:01
    Type: IPv4 (0x0800)
  Internet Protocol Version 4
    Src: 10.0.0.1
    Dst: 10.0.0.2
    Protocol: UDP (17)
    TTL: 254
    DSCP: CS0
  User Datagram Protocol
    Src Port: 51234
    Dst Port: 4789
    Length: 1522
  Virtual eXtensible Local Area Network
    Flags: 0x08 (I-flag set)
    VNI: 100
  Ethernet II (inner)
    Src: 00:aa:aa:aa:aa:aa
    Dst: 00:bb:bb:bb:bb:bb
    Type: IPv4 (0x0800)
  Internet Protocol Version 4 (inner)
    Src: 10.1.1.5
    Dst: 10.1.1.6
  Transmission Control Protocol (inner)
    ...
```

## Common Packet Format Issues (Exam Favorites)

| Issue | Cause | Symptom |
|-------|-------|---------|
| Fragmentation | Underlay MTU too small | Intermittent large-packet loss |
| No ECMP balance | Source port not varying / hash config | One spine link hot |
| Firewall drops | UDP 4789 blocked | VXLAN tunnels don't form |
| Wrong VNI | Misconfiguration | Traffic black-holed |
| I-flag not set | Bug or non-standard impl | VTEP drops packet |
| TTL expiry | Underlay loop / too many hops | Traceroute shows TTL exceeded |
| DSCP mismatch | QoS policy | Voice/video quality issues in overlay |

## Key Takeaways

- VXLAN adds exactly 50 bytes of overhead. Plan your MTU accordingly.
- The UDP source port provides ECMP entropy — it's a hash of inner headers.
- Destination UDP port is 4789. Firewalls must allow it.
- The I-flag in the VXLAN header must be set for the VNI to be valid.
- The inner frame is preserved completely — it's just payload to the underlay.
- Outer headers are rewritten at every hop; inner headers never change.

## What's Next

Chapter 5 explores the data plane in action — how unicast, multicast, and BUM (Broadcast, Unknown unicast, Multicast) traffic actually flows through a VXLAN fabric.
