# Chapter 15: Multi-Site VXLAN & Data Center Interconnect

## Why Multi-Site?

Single data center designs are increasingly rare. Businesses need:

- **Disaster recovery**: Workload failover to a secondary site
- **Active-active**: Applications spanning multiple DCs for latency/resilience
- **Cloud bursting**: Overflow to public cloud during peak
- **Mergers/acquisitions**: Connecting disparate DC networks
- **Geo-distribution**: Users served from nearest DC

VXLAN fabrics must extend beyond a single site. This chapter covers how.

## Multi-Site Design Options

### Option 1: L2 Stretch (Same VNI Across Sites)

Extend the same VXLAN segment (VNI) across multiple data centers.

```
┌──── Site A ────┐          ┌──── Site B ────┐
│ [Spine] [Spine]│          │ [Spine] [Spine]│
│   │       │    │          │   │       │    │
│ [Leaf-1][Leaf-2]│◄══════►│ [Leaf-3][Leaf-4]│
│   │              │  DCI   │              │ │
│ VM-A (VNI 100)  │  Link   │ VM-B (VNI 100)│
└─────────────────┘          └─────────────────┘
```

**VM-A and VM-B are in the same L2 segment**, despite being in different buildings.

**Use cases:**
- vMotion / Live Migration across sites
- Applications requiring L2 adjacency
- Active-active clusters (database, etc.)

**Challenges:**
- BUM traffic crosses the DCI (bandwidth waste)
- Failure domain spans sites (broadcast storm can cross)
- MAC mobility detection (VM moved or duplicate MAC?)
- ARP flooding across sites (without suppression)

### Option 2: L3 Interconnect (Different VNIs, Routed Between)

Each site has its own VNIs. Sites connect via L3 routing.

```
┌──── Site A ────┐          ┌──── Site B ────┐
│ VNI 100: 10.1.1.0/24│    │ VNI 200: 10.2.1.0/24│
│ VNI 101: 10.1.2.0/24│    │ VNI 201: 10.2.2.0/24│
│         │         │      │         │         │
│    [Border Leaf]  │◄════►│  [Border Leaf]  │
│         │         │ L3   │         │         │
└─────────────────┘ DCI  └─────────────────┘
```

**Use cases:**
- DR with planned failover (not live migration)
- Microservices communicating over L3
- Sites with independent addressing

**Advantages:**
- Failure domains are site-local
- No BUM across DCI
- Simpler troubleshooting
- Standard IP routing between sites

### Option 3: Hybrid (Selective L2 Stretch + L3)

Stretch only the VNIs that need L2 adjacency. Route everything else.

```
Stretched: VNI 100 (database cluster needs L2)
Routed:    VNI 200, 300, 400 (web, app, management)
```

This is the most common production design.

## Cisco Multi-Site Solutions

### Nexus 9000: EVPN Multi-Site

Using standard EVPN with border leaves:

```
┌──── Site A ────┐     ┌──── Site B ────┐
│                │     │                │
│ [Spine/RR]    │     │ [Spine/RR]    │
│   │    │      │     │   │    │      │
│ [Leaf][Border]│◄═══►│[Border][Leaf] │
│         │     │ DCI │  │            │
│         │     │     │  │            │
│    [Leaf]     │     │ [Leaf]        │
└────────────────┘     └────────────────┘
```

**Border Leaf responsibilities:**
- Participates in both site's EVPN domain
- Re-originates routes between sites (route leakage)
- Handles inter-site BUM replication
- May use different ASN per site (eBGP between sites)

**Configuration concept:**
```
! Border Leaf - Site A side
router bgp 65001
  neighbor 10.0.0.1    ← Site A spine/RR
    address-family l2vpn evpn

! Border Leaf - Site B side (or DCI peer)
  neighbor 172.16.1.2  ← Site B border
    remote-as 65002
    address-family l2vpn evpn

! Route re-origination between sites
  address-family l2vpn evpn
    redistribute internal
```

### ACI Multi-Site (MSO)

ACI Multi-Site uses **Multi-Site Orchestrator (MSO)** to manage fabrics across sites:

```
┌─────────────────────────┐
│   Multi-Site Orchestrator│
│   (MSO / NDO)           │
└────┬──────────┬─────────┘
     │          │
┌────┴────┐ ┌──┴──────┐
│ APIC-A  │ │ APIC-B  │
│ (Site A)│ │ (Site B)│
└────┬────┘ └────┬────┘
     │            │
[Fabric A]   [Fabric B]
```

**Key concepts:**
- **Stretched BD**: Same Bridge Domain across sites (L2 stretch)
- **Stretched VRF**: Same VRF across sites (L3 continuity)
- **Local BD**: Site-specific Bridge Domain
- **Inter-site L3Out**: Routed connection between sites

**ACI Multi-Site VXLAN specifics:**
- Uses a **Multi-Site VNID** for inter-site traffic
- Border leaves perform VXLAN-to-VXLAN translation
- COOP database is site-local; inter-site uses BGP EVPN

## DCI Transport Options

The link between sites (DCI) can be:

| Transport | Latency | Bandwidth | Cost | VXLAN Support |
|-----------|---------|-----------|------|--------------|
| Dark fiber | <1ms/km | 100G+ | High | Native (just extend underlay) |
| DWDM | <1ms/km | 100G+ | High | Native |
| MPLS L3VPN | 5-50ms | 1-100G | Medium | VRF-aware, needs border |
| IP WAN | 10-100ms | Variable | Low | VXLAN over IP (encrypted) |
| Internet + IPsec | 20-200ms | Variable | Low | Encrypted tunnel |

### DCI Design Considerations

**Latency:**
- L2 stretch: Keep RTT < 10ms (ARP, MAC learning timers)
- L3 interconnect: More tolerant (up to 100ms+)
- vMotion: VMware recommends < 10ms RTT for L2 stretch

**Bandwidth:**
- BUM traffic estimation: (VNIs stretched) × (broadcast rate per VNI)
- Unicast: Application traffic patterns
- Replication factor: Head-end rep multiplies BUM by N

**MTU:**
- DCI links must support VXLAN overhead (50 bytes)
- If DCI adds its own encap (IPsec, MPLS), add more
- End-to-end MTU verification is critical

**Resilience:**
- Dual DCI paths (diverse routes)
- BFD for fast failure detection
- Graceful degradation (site-local if DCI fails)

## BUM Traffic Across Sites

This is the hardest multi-site problem:

### With Underlay Multicast DCI
- Extend PIM across DCI
- Multicast groups span sites
- Efficient but operationally complex

### With Head-End Replication
- Border leaf replicates BUM to remote site VTEPs
- Bandwidth waste on DCI link
- Simpler but doesn't scale

### With EVPN + Selective Forwarding
- Only stretch VNIs that need it
- ARP suppression reduces BUM dramatically
- Unknown unicast drop prevents flooding
- **Best practice for most deployments**

## MAC Mobility and Failure Detection

When a VM migrates between sites:

```
VM-A moves from Site A to Site B:
1. VM-A sends frame on Site B leaf
2. Site B leaf advertises Type 2: MAC-A now at Site B VTEP
3. Site A leaf receives updated Type 2 (newer sequence number)
4. Site A leaf updates MAC table: MAC-A → remote (Site B VTEP)
5. Traffic to MAC-A now flows to Site B
```

**MAC move detection:**
```
! NX-OS MAC move limiting (prevent flapping)
system mac address-table move-detect
  threshold 5
  interval 10
```

If a MAC moves more than 5 times in 10 seconds → flagged as potential loop/attack.

## Verification (Multi-Site)

```bash
! Verify inter-site EVPN routes
show bgp l2vpn evpn route-type 2 vni 100
! Should show routes from both sites

! Verify border leaf peers
show bgp l2vpn evpn summary
! Should show intra-site RR + inter-site border peers

! Verify MAC mobility
show mac address-table vlan 100
! Remote MACs should show correct remote VTEP (possibly in other site)

! Verify DCI underlay
ping 172.16.1.2 source-interface loopback0
traceroute 172.16.1.2

! Check BUM replication
show nve peers vni 100
! Should list VTEPs from both sites
```

## Key Takeaways

- Multi-site VXLAN can be L2 stretch, L3 interconnect, or hybrid.
- L2 stretch enables vMotion but extends failure domains.
- Border leaves (or ACI MSO) handle inter-site route exchange.
- BUM across DCI is the key scaling challenge — use ARP suppression + selective stretch.
- DCI transport choice (dark fiber vs MPLS vs IP) drives design constraints.
- MAC mobility detection prevents flapping and identifies misconfigurations.
- For CCIE DC: understand both NX-OS EVPN multi-site and ACI Multi-Site concepts.

## What's Next

Chapter 16 covers VXLAN automation — APIs, Ansible, Terraform, and programmatic fabric management.
