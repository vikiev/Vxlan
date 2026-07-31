# Chapter 14: VXLAN vs NVGRE vs Geneve vs STT

## Why This Chapter Matters

The CCIE DC exam expects you to understand the **landscape** of overlay technologies, not just VXLAN. You'll see comparison questions in DCCOR, and understanding alternatives helps you articulate *why* VXLAN won.

## The Overlay Wars (2011-2015)

In 2011-2013, four competing overlay technologies emerged:

| Technology | Champion | Transport | Header Size | Status |
|-----------|----------|-----------|-------------|--------|
| VXLAN | VMware, Cisco, Arista | UDP/IP | 8 bytes | **Won** |
| NVGRE | Microsoft, HP, Dell | GRE/IP | 4 bytes | Dead |
| STT | VMware, Nicira | TCP-like/IP | 16 bytes | Dead |
| Geneve | IETF (successor) | UDP/IP | Variable | Emerging |

## VXLAN (RFC 7348)

**Recap:** UDP encapsulation, 24-bit VNI, port 4789.

**Strengths:**
- UDP transport → ECMP-friendly (source port entropy)
- Hardware offload widely available (all major ASICs)
- Massive ecosystem (every vendor supports it)
- Simple 8-byte header → minimal overhead
- Works with existing IP networks (no GRE support needed)

**Weaknesses:**
- Fixed header → no extensibility (can't add metadata easily)
- No built-in OAM (Operations, Administration, Maintenance)
- UDP checksum usually disabled → no integrity check
- 24-bit VNI is "enough" but not future-proof

## NVGRE (Network Virtualization using GRE)

**RFC:** Draft (never became RFC). Championed by Microsoft for Hyper-V.

**Packet format:**
```
┌──────────────────────────────────────────────────────┐
│ Outer Eth │ Outer IP │ GRE Header │ Inner Frame      │
│           │          │ (with Key) │                  │
└──────────────────────────────────────────────────────┘
```

The GRE "Key" field (32 bits) carries a 24-bit Tenant Network Identifier (TNI) + 8-bit flow ID.

**Why it lost:**
- GRE is IP protocol 47 → many firewalls block it
- GRE doesn't have a natural entropy field → ECMP hashing is poor
- Hardware offload for GRE was less common than UDP
- Microsoft eventually adopted VXLAN for Azure

**Key difference from VXLAN:**
- NVGRE: GRE encapsulation (protocol 47)
- VXLAN: UDP encapsulation (protocol 17, port 4789)
- NVGRE has 8-bit flow ID in header (built-in entropy)
- VXLAN uses UDP source port for entropy

## STT (Stateless Transport Tunneling)

**Draft:** Nicira (later VMware NSX). Never standardized.

**Packet format:**
```
┌──────────────────────────────────────────────────────┐
│ Outer Eth │ Outer IP │ STT Header (16B) │ Inner Frame│
└──────────────────────────────────────────────────────┘
```

STT used a **TCP-like header** (but stateless — no connection tracking) to leverage existing NIC TCP offload engines (TSO, LRO).

**Why it lost:**
- 16-byte header (double VXLAN's overhead)
- TCP-like semantics confused middleboxes
- Looked like TCP to firewalls → unpredictable behavior
- VMware chose VXLAN for NSX instead
- Never had multi-vendor support

## Geneve (Generic Network Virtualization Encapsulation)

**RFC:** RFC 8926 (2020). The IETF's "do it right this time" standard.

**Packet format:**
```
┌──────────────────────────────────────────────────────────────┐
│ Outer Eth │ Outer IP │ UDP │ Geneve Header │ TLVs │ Inner    │
│           │          │ 6081│ (8B + TLVs)  │      │ Frame    │
└──────────────────────────────────────────────────────────────┘
```

**Geneve header:**
```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|Ver|  Opt Len  |O|C|    Rsvd.  |          Protocol Type        |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|        Virtual Network Identifier (VNI)       |    Reserved   |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                    Variable-Length Options (TLVs)             |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

**Key features:**
- **Variable-length TLV options**: Can carry arbitrary metadata (policy, tracing, security tags)
- **Protocol Type field**: Can encapsulate Ethernet, IPv4, IPv6, NSH, anything
- **UDP port 6081**: Different from VXLAN (4789)
- **O-bit**: OAM packet indicator
- **C-bit**: Critical options present (must be understood)

**Why Geneve matters:**
- It's VXLAN's successor (eventually)
- AWS uses Geneve internally
- OVN/OVS supports Geneve natively
- Extensible: can carry security group tags, tracing info, etc.
- IETF standard (unlike VXLAN which was a vendor draft first)

**Why Geneve hasn't replaced VXLAN yet:**
- Hardware support is newer (but growing)
- Existing VXLAN deployments are enormous
- EVPN integration is mature for VXLAN, newer for Geneve
- "If it ain't broke..."

## Comparison Matrix

| Feature | VXLAN | NVGRE | STT | Geneve |
|---------|-------|-------|-----|--------|
| Transport | UDP | GRE | TCP-like | UDP |
| Port/Protocol | UDP 4789 | IP Proto 47 | IP Proto 17 | UDP 6081 |
| Header size | 8 bytes | 4 bytes (+GRE) | 16 bytes | 8+ bytes (variable) |
| Segment ID | 24-bit VNI | 24-bit TNI + 8-bit flow | 24-bit | 24-bit VNI |
| Extensibility | None | Limited | Limited | **TLVs (unlimited)** |
| ECMP entropy | UDP src port | GRE flow ID (8-bit) | Pseudo-header | UDP src port |
| Hardware support | **Universal** | Rare | None | Growing |
| Firewall friendly | Yes (UDP) | No (GRE blocked) | Maybe (looks TCP) | Yes (UDP) |
| OAM support | None | None | None | **Built-in (O-bit)** |
| Multi-vendor | **Yes** | No (MS only) | No (VMware) | Yes (emerging) |
| EVPN integration | **Mature (RFC 8365)** | Draft | None | Draft |
| Current status | **Dominant** | Dead | Dead | Emerging |

## What the CCIE DC Exam Tests

You won't be asked to configure NVGRE or STT. But you should know:

1. **Why VXLAN uses UDP** (ECMP, firewall traversal, hardware offload)
2. **Why NVGRE failed** (GRE blocking, poor ECMP, single-vendor)
3. **What Geneve adds** (TLVs, extensibility, OAM)
4. **The overhead comparison** (VXLAN 50B, NVGRE ~42B, STT ~58B, Geneve 50B+)
5. **Why UDP source port matters** (entropy for ECMP)

## The Future: VXLAN → Geneve Transition?

The industry is slowly moving toward Geneve for new deployments:
- AWS: Geneve internally
- OVN (Open Virtual Network): Geneve default
- Linux kernel: Geneve support mature
- Hardware: Broadcom, Mellanox adding Geneve offload

But VXLAN isn't going anywhere soon:
- Millions of ports deployed
- EVPN-VXLAN is the enterprise standard
- Cisco, Arista, Juniper all VXLAN-first
- CCIE DC tests VXLAN, not Geneve (as of current blueprint)

**Prediction:** VXLAN and Geneve will coexist for 5-10 years. New greenfield might choose Geneve; existing VXLAN fabrics won't be ripped out.

## Key Takeaways

- VXLAN won because of UDP transport, hardware support, and multi-vendor ecosystem.
- NVGRE died because GRE is firewall-unfriendly and had poor ECMP.
- STT died because TCP-like headers confused middleboxes.
- Geneve is the IETF's extensible successor (TLVs, OAM) but hasn't displaced VXLAN yet.
- For CCIE DC: know VXLAN deeply, understand alternatives at a comparison level.
- The 50-byte overhead and UDP 4789 are VXLAN's fingerprints — know them cold.

## What's Next

Chapter 15 covers multi-site VXLAN and Data Center Interconnect — extending the fabric across geographies.
