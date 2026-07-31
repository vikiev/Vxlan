# Chapter 12: VXLAN Design & Scalability Considerations

## Design Is Where Exams Are Won

Any candidate can configure a 3-leaf VXLAN fabric from a guide. The CCIE lab and the DCCOR exam test whether you can **design** a fabric that scales, survives failures, and meets business requirements. This chapter is about the decisions that separate a working lab from a production architecture.

## Fabric Sizing: The Numbers Game

### Underlay Scale

| Component | Small | Medium | Large |
|-----------|-------|--------|-------|
| Leaves | 4-16 | 16-64 | 64-256 |
| Spines | 2 | 4 | 8-16 |
| Links per leaf | 2 uplinks | 4 uplinks | 8+ uplinks |
| ECMP paths | 2 | 4 | 8-16 |
| Underlay IGP | OSPF single area | OSPF or IS-IS | IS-IS or eBGP |

### Overlay Scale

| Component | Small | Medium | Large |
|-----------|-------|--------|-------|
| VNIs per leaf | 10-50 | 50-500 | 500-5000 |
| Total VNIs | 50 | 500 | 5000+ |
| MACs per leaf | 1K | 10K | 100K |
| EVPN routes (total) | 5K | 100K | 1M+ |
| BGP sessions per leaf | 2 (to RRs) | 2-4 | 4 |
| Route Reflectors | 2 (spines) | 2-4 | 4-8 (dedicated) |

## Spine-Leaf Design Decisions

### How Many Spines?

The number of spines determines:
- **Oversubscription ratio**: Total leaf uplink bandwidth ÷ total leaf downlink bandwidth
- **Redundancy**: Surviving N-1 spines without congestion
- **ECMP spread**: More spines = better hash distribution

```
Example: 32 leaves, each with 48x25G downlinks + 6x100G uplinks

Downlink per leaf: 48 × 25G = 1200G
Uplink per leaf:   6 × 100G = 600G
Oversubscription:  1200/600 = 2:1

With 6 spines: each leaf has 6 × 100G uplinks (one per spine)
Lose 1 spine: 5 × 100G = 500G uplink → 2.4:1 oversubscription (acceptable)
```

### Spine as Route Reflector vs. Dedicated RR

| Approach | Pros | Cons |
|----------|------|------|
| Spines as RRs | Fewer devices, simpler | Spine CPU/memory for BGP + forwarding |
| Dedicated RRs | Spines focus on forwarding | More devices, more failure domains |
| Hybrid (spine + RR) | Best of both | More complex |

**Recommendation:** Up to ~64 leaves, spines as RRs is fine. Beyond that, consider dedicated RRs.

### Leaf Pair Design

For redundancy, leaves are often deployed in **pairs** (vPC or EVPN multi-homing):

```
        [Spine-1]  [Spine-2]
           │╲  ╲╱╱    │
           │ ╲╱╲╱     │
        [Leaf-1]──[Leaf-2]    ← Pair (vPC or EVPN MH)
           │         │
        ┌──┴─────────┴──┐
        │   Server Rack  │
        └────────────────┘
```

## EVPN Multi-Homing (All-Active)

### The Problem

A server with dual NICs connected to two leaves. Traditional solution: vPC (Cisco proprietary) or MC-LAG.

### EVPN Multi-Homing Solution

EVPN defines **Ethernet Segments (ES)** for multi-homing:

```
Ethernet Segment:
  ESI: 00:01:00:00:00:00:00:00:00:01
  Members: Leaf-1 (Eth1/1), Leaf-2 (Eth1/1)
  Mode: All-Active
```

**All-Active**: Both links forward simultaneously. ECMP across both leaves.

**Single-Active**: One link active, one standby. Like STP but faster.

### EVPN MH Route Types Used

- **Type 1 (Per-ES A-D)**: Advertises the ES, enables aliasing
- **Type 1 (Per-EVI A-D)**: Split-horizon filtering, mass withdrawal
- **Type 4 (ES Route)**: DF election among ES members

### Split-Horizon Problem

When Leaf-1 receives BUM from the server (via ES), it must NOT send it back to Leaf-2 (also on the same ES). Otherwise: loop.

**Solution:** Type 1 Per-EVI A-D routes carry the ESI. When a VTEP receives BUM with a specific ESI label, it knows not to forward it back to that ES.

### DF Election

For BUM traffic from the network toward the multi-homed server, one leaf must be the **Designated Forwarder (DF)**:

```
ES members: Leaf-1, Leaf-2
DF election (based on BGP best path): Leaf-1 wins
  → Leaf-1 forwards BUM to server
  → Leaf-2 does NOT forward BUM to server (non-DF)
```

For unicast: both forward (all-active). For BUM: only DF forwards.

## Scalability Bottlenecks and Solutions

### 1. BGP Route Scale

**Problem:** 100K+ EVPN routes overwhelm VTEP TCAM/RAM.

**Solutions:**
- Route reflectors (reduce sessions, not routes)
- Route filtering (only import routes for local VNIs)
- `retain route-target all` only on RRs, not on leaves
- Aggregate Type 5 routes where possible

### 2. BUM Traffic Scale

**Problem:** Head-end replication with 100+ VTEPs per VNI.

**Solutions:**
- P-multicast for BUM (underlay multicast)
- ARP suppression (eliminate most BUM)
- Unknown unicast drop (eliminate flooding)
- Limit VNI span (don't stretch every VNI everywhere)

### 3. MAC Table Scale

**Problem:** ASIC TCAM limited to 100K-200K MACs.

**Solutions:**
- EVPN (only learn relevant MACs, not all)
- MAC aging (remove stale entries)
- Limit VNI-to-leaf mapping (not every VNI on every leaf)
- Hardware platforms with larger TCAM

### 4. Convergence Time

**Problem:** Link/node failure → how fast does traffic reconverge?

**Solutions:**
- BFD on underlay links (sub-second detection)
- BGP PIC (Prefix Independent Convergence) for fast FIB update
- ECMP (lose one path, others continue immediately)
- EVPN fast withdraw (BGP UPDATE, not timer-based)

```
! BFD on underlay
interface Ethernet1/49
  bfd interval 50 min_rx 50 multiplier 3

! BGP PIC
router bgp 65000
  address-family l2vpn evpn
    bestpath as-path multipath-relax
    maximum-paths 8
```

## Design Patterns

### Pattern 1: Single Pod (Up to 32 Leaves)

```
    [Spine-1] [Spine-2] [Spine-3] [Spine-4]
       │  │      │  │      │  │      │  │
    [L1][L2] [L3][L4] [L5][L6] ... [L31][L32]
```
- 4 spines, 32 leaves
- Spines are RRs
- Single OSPF area
- All-active EVPN MH for server pairs

### Pattern 2: Multi-Pod (32-256 Leaves)

```
    ┌─── Pod 1 ───┐     ┌─── Pod 2 ───┐
    │ [Sp][Sp]    │     │ [Sp][Sp]    │
    │ [L][L][L]  │     │ [L][L][L]  │
    └──────┬──────┘     └──────┬──────┘
           │                    │
       [Super-Spine-1]  [Super-Spine-2]
```
- Each pod is independent (own spines/RRs)
- Super-spines connect pods
- Inter-pod traffic via super-spines
- Separate OSPF areas or eBGP between pods

### Pattern 3: Stretched Fabric (DCI)

Covered in Chapter 15. Key design points:
- Same VNI across sites (L2 stretch)
- Or different VNIs with L3 interconnect
- Border leaves for inter-site routing
- RT filtering to control route propagation

## Capacity Planning Formulas

```
Uplink bandwidth per leaf = (Number of spines) × (Spine link speed)
Downlink bandwidth per leaf = (Number of ports) × (Port speed)
Oversubscription = Downlink ÷ Uplink

ECMP paths = Number of spines (in single-pod)
Maximum leaf-to-leaf latency = 2 × (leaf-to-spine latency) + spine processing

BGP sessions per leaf = Number of RRs (typically 2-4)
Total EVPN routes ≈ (Total MACs) × 2 + (Total VNIs) + (Total subnets)
```

## Key Takeaways

- Design for failure: N-1 spine, N-1 leaf, BFD, ECMP.
- EVPN multi-homing replaces vPC/MC-LAG for all-active server access.
- Scale bottlenecks are: BGP routes, BUM replication, MAC tables, convergence.
- Use spines as RRs for small fabrics; dedicated RRs for large.
- Multi-pod designs use super-spines and separate routing domains.
- Always calculate oversubscription and ECMP distribution before deploying.

## What's Next

Chapter 13 covers VXLAN security — threats, hardening, and microsegmentation.
