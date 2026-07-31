# Chapter 11: VXLAN on Cisco Nexus & ACI

## Two Worlds: NX-OS and ACI

Cisco implements VXLAN in two fundamentally different ways:

| Aspect | Nexus 9000 (NX-OS) | ACI (APIC) |
|--------|--------------------|-----------| 
| VXLAN mode | RFC 7348 (standard) | Cisco proprietary extensions |
| Control plane | EVPN (BGP) or multicast | COOP + IS-IS (spine) |
| Configuration | CLI (manual) | APIC GUI/API (declarative) |
| VNI assignment | Manual | Automatic (APIC) |
| Target audience | Network engineers | Cloud/DevOps + NetEng |
| Exam focus | 300-630 DCACI | 300-630 DCACI |

Both are on the CCIE DC exam. Let's cover both.

## Part 1: Nexus 9000 Standalone NX-OS

### Platform Overview

The Nexus 9000 series is Cisco's data center switch line. In standalone NX-OS mode, you configure VXLAN manually (as shown in Chapters 9-10).

**Key platforms:**
- N9K-C93180YC-FX / FX2 / FX3 — 48x25G + 6x100G (ToR leaf)
- N9K-C9364C — 64x100G (spine or high-density leaf)
- N9K-C9508 — Modular chassis (core/spine)

**ASIC capabilities:**
- Hardware VXLAN encap/decap at line rate
- 100K+ MAC entries in hardware
- 16K+ VNI support
- ARP suppression in hardware
- ECMP across 64+ paths

### Complete NX-OS VXLAN Configuration Template

```
! === FEATURES ===
feature ospf
feature bgp
feature interface-vlan
feature vn-segment-vlan-based
feature nv overlay
feature fabric forwarding

! === FABRIC FORWARDING (Anycast GW) ===
fabric forwarding anycast-gateway-mac 0000.1111.1111

! === UNDERLAY ===
interface loopback0
  description ROUTER-ID / VTEP-SOURCE
  ip address 10.0.0.1/32
  ip router ospf UNDERLAY area 0.0.0.0

interface Ethernet1/49
  description TO-SPINE-1
  no switchport
  ip address 10.1.49.1/31
  ip router ospf UNDERLAY area 0.0.0.0
  mtu 9216

router ospf UNDERLAY
  router-id 10.0.0.1

! === VLAN / VNI MAPPING ===
vlan 100
  vn-segment 100
vlan 200
  vn-segment 200
vlan 10000
  vn-segment 10000

! === SVIs ===
interface Vlan100
  no shutdown
  vrf member Tenant-A
  ip address 10.1.1.1/24
  fabric forwarding mode anycast-gw

interface Vlan200
  no shutdown
  vrf member Tenant-A
  ip address 10.2.1.1/24
  fabric forwarding mode anycast-gw

! === VRF ===
vrf context Tenant-A
  vni 10000
  address-family ipv4 unicast
    route-target both auto
    route-target both auto evpn

! === NVE (VTEP) ===
interface nve1
  no shutdown
  source-interface loopback0
  host-reachability protocol bgp
  member vni 100
    suppress-arp
    ingress-replication protocol bgp
  member vni 200
    suppress-arp
    ingress-replication protocol bgp
  member vni 10000 associate-vrf

! === BGP ===
router bgp 65000
  router-id 10.0.0.1
  address-family l2vpn evpn
    retain route-target all
  neighbor 10.0.0.10
    remote-as 65000
    update-source loopback0
    address-family l2vpn evpn
      send-community both

! === ACCESS PORTS ===
interface Ethernet1/1
  switchport mode access
  switchport access vlan 100
  spanning-tree port type edge
```

### NX-OS Verification Toolkit

```bash
! Underlay health
show ip route ospf
show ip ospf neighbors
ping 10.0.0.2 source-interface loopback0

! VTEP status
show nve vni
show nve peers
show interface nve1

! EVPN control plane
show bgp l2vpn evpn summary
show bgp l2vpn evpn route-type 2
show bgp l2vpn evpn route-type 3
show bgp l2vpn evpn route-type 5

! Forwarding tables
show mac address-table vlan 100
show ip arp suppression-cache
show ip arp vrf Tenant-A
show ip route vrf Tenant-A

! Statistics and counters
show interface nve1 counters
show system internal nve stats
show hardware internal access-list manager stats

! Troubleshooting
show logging | grep -i "nve\|vni\|evpn"
debug bgp l2vpn evpn updates    ← USE WITH CAUTION
```

## Part 2: Cisco ACI

### ACI Architecture Overview

ACI (Application Centric Infrastructure) is Cisco's SDN solution for data centers. It uses VXLAN internally but abstracts it behind a policy model.

```
┌─────────────────────────────────────────────────┐
│                  APIC Cluster                     │
│         (Policy Controller, 3+ nodes)            │
└──────────────────────┬──────────────────────────┘
                       │ (Southbound: COOP, IS-IS)
┌──────────────────────┴──────────────────────────┐
│              ACI Fabric                          │
│                                                  │
│   [Spine-1]  [Spine-2]  [Spine-3]              │
│       │           │           │                  │
│   [Leaf-1]  [Leaf-2]  [Leaf-3]  [Leaf-4]      │
│       │           │                              │
│    Servers     Servers                           │
└─────────────────────────────────────────────────┘
```

### ACI vs NX-OS: Key Differences

| Concept | NX-OS | ACI |
|---------|-------|-----|
| VLAN | Configured manually | Encap (auto-assigned) |
| VNI | Configured manually | Auto-assigned by APIC |
| VRF | `vrf context` | Tenant → VRF |
| Subnet | SVI with IP | Bridge Domain → Subnet |
| L2 segment | VLAN + VNI | Bridge Domain (BD) |
| L3 segment | VRF + SVI | VRF + BD + Subnet |
| Gateway | SVI IP | BD Subnet (scope) |
| Policy | ACLs, route-maps | Contracts, Filters |
| VXLAN encap | Standard RFC 7348 | Cisco VXLAN (8-byte VNID) |

### ACI VXLAN Specifics

ACI uses a **modified VXLAN header**:
- 8-byte VNID (instead of 24-bit VNI) — allows for more metadata
- Additional policy bits in the VXLAN header
- Not interoperable with standard VXLAN (without VXLAN flood-and-learn or external connectivity)

ACI VNID types:
- **BD VNID**: Identifies a Bridge Domain (L2 segment)
- **VRF VNID**: Identifies a VRF (L3 instance)
- **EPG VNID**: Identifies an Endpoint Group (policy group)

### ACI Control Plane: COOP

ACI doesn't use BGP EVPN internally. It uses **COOP (Council of Oracles Protocol)**:

- Spine switches run the COOP database
- Leaf switches register endpoints (MAC + IP + location) with spines
- Spines act as a distributed endpoint database
- Leaf queries spine for unknown destinations

```
Leaf-1 learns VM-A (MAC + IP):
  → Registers with Spine COOP database:
    "MAC 00:AA:AA:AA:AA:AA, IP 10.1.1.5, on Leaf-1, VTEP 10.0.0.1"

Leaf-2 needs to reach VM-A:
  → Queries Spine COOP: "Where is 10.1.1.5?"
  → Spine replies: "Leaf-1, VTEP 10.0.0.1, BD-VNID 15001"
  → Leaf-2 sends VXLAN to Leaf-1
```

### ACI Policy Model (Brief)

```
Tenant
  └── VRF (L3 instance)
       └── Bridge Domain (L2 segment + subnet)
            └── Subnet (gateway IP)
  └── Application Profile
       └── EPG (Endpoint Group)
            └── Static Path (port binding)
            └── Contracts (policy)
```

For the CCIE DC exam, you need to understand:
- How ACI maps to VXLAN concepts
- The APIC GUI workflow for creating tenants/BDs/EPGs
- ACI external connectivity (L3Out, L2Out)
- ACI multi-site (covered in Chapter 15)

### ACI Verification

```bash
! On leaf (NX-OS style CLI available)
show endpoint
show ip arp vrf <vrf-name>
show vxlan vni
show bgp evpn summary    ! (for external EVPN)

! Via APIC REST API
GET /api/node/class/fvCEp.json          ← All endpoints
GET /api/node/class/fvBD.json           ← All bridge domains
GET /api/node/class/vzBrCP.json         ← All contracts
```

## Part 3: Choosing Between NX-OS and ACI

| Factor | Choose NX-OS | Choose ACI |
|--------|-------------|-----------|
| Team skills | Traditional CLI | API/GUI comfortable |
| Scale | <100 leaves | 100+ leaves |
| Automation | Ansible/Terraform | APIC API / Terraform |
| Multi-tenancy | Manual VRFs | Native tenant model |
| Microsegmentation | Manual ACLs | Contracts (native) |
| Interop | Standard VXLAN/EVPN | Cisco ecosystem |
| Budget | Lower (no APIC) | Higher (APIC licenses) |

## Key Takeaways

- Nexus 9000 NX-OS uses standard VXLAN + EVPN (what Chapters 1-10 taught you).
- ACI uses VXLAN internally but with proprietary extensions and COOP control plane.
- ACI abstracts VXLAN behind a policy model (Tenant/BD/EPG/Contract).
- Both are on the CCIE DC exam — NX-OS for config/troubleshoot, ACI for design/policy.
- The verification methodology is the same: underlay → control plane → forwarding → traffic.

## What's Next

Chapter 12 covers VXLAN design and scalability — how to architect a fabric for 10, 100, or 1000+ VTEPs.
