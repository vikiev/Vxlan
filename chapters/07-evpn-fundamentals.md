# Chapter 7: EVPN Fundamentals — The Modern Control Plane

## What EVPN Actually Is

EVPN (Ethernet VPN) is a **control plane technology** that uses BGP to distribute Layer 2 and Layer 3 reachability information. It was originally defined in RFC 7432 for MPLS networks, then adapted for VXLAN in RFC 8365.

The core insight: **treat MAC addresses like IP prefixes**. Instead of flooding frames to learn MACs (data plane learning), advertise them via BGP (control plane learning). Just as BGP tells you "prefix 10.0.0.0/24 is reachable via router X," EVPN tells you "MAC 00:AA:AA:AA:AA:AA is reachable via VTEP Y."

## Why "VPN" in the Name?

EVPN provides **virtual private network** semantics:
- Each EVPN instance (EVI) is an isolated L2 domain (like a VRF for L2)
- Multiple tenants share the same physical infrastructure
- Complete isolation between EVIs
- In VXLAN, each VNI maps to an EVI

## EVPN Components

### 1. EVPN Instance (EVI)

An EVI is a single EVPN broadcast domain. In VXLAN terms: **one EVI = one VNI**.

```
EVI 100 ←→ VNI 100 ←→ VLAN 100 (locally)
EVI 200 ←→ VNI 200 ←→ VLAN 200 (locally)
```

Each EVI has its own:
- Route Distinguisher (RD) — makes routes globally unique
- Route Targets (RT) — controls import/export between EVIs
- MAC/IP table
- BUM replication list

### 2. Ethernet Tag (ET)

The Ethernet Tag identifies a broadcast domain within an EVPN instance. In most VXLAN deployments, the ET is the **VLAN ID** or **VNI**.

For a simple VXLAN fabric: ET = VNI. One EVI per VNI, one tag per EVI.

### 3. Route Distinguisher (RD)

The RD makes an EVPN route globally unique. Two different tenants can use the same MAC address (00:11:22:33:44:55) — the RD distinguishes them.

```
Format: <admin>:<local>
Example: 10.0.0.1:100  (router-id:EVI)
         65000:100      (ASN:EVI)
```

An EVPN route is identified by: **RD + Ethernet Tag + MAC + IP**

### 4. Route Target (RT)

RTs control which EVIs import/export routes. They enable:
- **L2 isolation**: EVI 100 only imports routes with RT 65000:100
- **L3 interconnection**: A VRF imports from multiple EVIs for inter-VNI routing

```
EVI 100:
  export RT: 65000:100
  import RT: 65000:100

EVI 200:
  export RT: 65000:200
  import RT: 65000:200

VRF "Tenant-A":
  import RT: 65000:100, 65000:200   ← Can route between both
```

### 5. EVPN PE (Provider Edge)

In VXLAN, the **VTEP is the EVPN PE**. It:
- Participates in BGP EVPN address family
- Advertises local MACs/IPs
- Learns remote MACs/IPs
- Handles BUM replication
- Performs routing between EVIs (if configured)

## EVPN Address Families

EVPN uses BGP with specific address families:

```
router bgp 65000
  address-family l2vpn evpn        ← EVPN routes (MAC/IP, multicast, etc.)
  neighbor 10.0.0.2
    address-family l2vpn evpn
```

The NLRI (Network Layer Reachability Information) for EVPN is defined per route type (covered in Chapter 8).

## EVPN vs. Traditional L2: A Paradigm Shift

| Aspect | Traditional L2 (STP + flood) | EVPN |
|--------|------------------------------|------|
| MAC learning | Data plane (flood & learn) | Control plane (BGP advertise) |
| Loop prevention | STP (blocks links) | No L2 loops (routed underlay) |
| Multi-path | One active path (STP) | All paths active (ECMP) |
| MAC scalability | Every switch learns all MACs | Only relevant MACs per VTEP |
| ARP | Broadcast flood | Suppressed (MAC/IP routes) |
| Failure detection | STP timers (30-50s) | BFD + BGP (sub-second) |
| Multi-homing | STP (active/standby) | All-active (EVPN MH) |
| Convergence | STP reconvergence | BGP withdraw + FIB update |

## EVPN Services Model

EVPN supports multiple service types:

### VLAN-Based Service
- One EVI per VLAN/VNI
- Each EVI is a separate broadcast domain
- Most common in VXLAN deployments

```
VLAN 100 → EVI 100 → VNI 100 (one segment)
VLAN 200 → EVI 200 → VNI 200 (another segment)
```

### VLAN-Bundle Service
- Multiple VLANs/VNIs in a single EVI
- They share the same broadcast domain
- Less common in VXLAN

### VLAN-Aware Bundle Service
- Multiple VLANs/VNIs in one EVI but with separate broadcast domains per tag
- Uses Ethernet Tag to differentiate
- Useful for QinQ scenarios

For CCIE DC, **VLAN-Based is what you'll configure 95% of the time**.

## EVPN and BGP: The Relationship

EVPN doesn't replace BGP — it **extends** it. EVPN is a BGP address family, just like IPv4 unicast or IPv6 multicast.

```
router bgp 65000
  router-id 10.0.0.1
  
  ! Standard IPv4 unicast (for underlay or L3VPN)
  address-family ipv4 unicast
  
  ! EVPN (for VXLAN overlay)
  address-family l2vpn evpn
  
  neighbor 10.0.0.2
    remote-as 65000
    update-source loopback0
    address-family l2vpn evpn
      send-community both
```

This means:
- You already know BGP? You know 70% of EVPN.
- Peering, session establishment, route reflectors, policies — all standard BGP.
- What's new: the NLRI format (route types) and the semantics (MAC/IP routes).

## EVPN in the VXLAN Stack

```
┌─────────────────────────────────────────────┐
│          Tenant VMs / Servers                │
├─────────────────────────────────────────────┤
│          VTEP (EVPN PE)                      │
│  ┌─────────────────────────────────────┐    │
│  │  EVPN Control Plane (BGP)           │    │
│  │  - Advertises local MAC/IP          │    │
│  │  - Learns remote MAC/IP             │    │
│  │  - Builds BUM peer list             │    │
│  └─────────────────────────────────────┘    │
│  ┌─────────────────────────────────────┐    │
│  │  VXLAN Data Plane                   │    │
│  │  - Encap/Decap                      │    │
│  │  - Forwarding per VNI MAC table     │    │
│  └─────────────────────────────────────┘    │
├─────────────────────────────────────────────┤
│          IP Underlay (OSPF/IS-IS/BGP)        │
├─────────────────────────────────────────────┤
│          Physical Network                    │
└─────────────────────────────────────────────┘
```

## Route Reflectors: Scaling EVPN

In a full-mesh BGP design, N VTEPs require N×(N-1)/2 sessions. At 100 VTEPs, that's 4,950 sessions. Unmanageable.

**Solution: Route Reflectors (RRs)**

```
        ┌──────────┐
        │  RR-1    │  (Spine-1 acting as RR)
        └────┬─────┘
       ╱     │      ╲
     ╱       │        ╲
[Leaf-1] [Leaf-2] [Leaf-3] ... [Leaf-100]
```

- Each leaf peers only with the RRs (2 sessions per leaf for redundancy)
- RRs reflect EVPN routes to all clients
- 100 leaves × 2 RRs = 200 sessions (vs 4,950)
- RRs don't need to be in the data path

**NX-OS RR configuration (on spine):**
```
router bgp 65000
  address-family l2vpn evpn
    retain route-target all
  
  neighbor 10.0.0.10    ← Leaf-1
    remote-as 65000
    address-family l2vpn evpn
      route-reflector-client
  
  neighbor 10.0.0.11    ← Leaf-2
    remote-as 65000
    address-family l2vpn evpn
      route-reflector-client
```

## Key Takeaways

- EVPN treats MAC addresses as BGP routes — proactive, scalable, fast.
- Each VNI maps to an EVPN Instance (EVI) with its own RD and RTs.
- EVPN is a BGP address family (`l2vpn evpn`) — standard BGP mechanics apply.
- Route Reflectors are essential for scaling beyond ~20 VTEPs.
- EVPN eliminates flood & learn, enables ARP suppression, and supports all-active multi-homing.
- For CCIE DC: EVPN is not optional. It IS the VXLAN control plane.

## What's Next

Chapter 8 dissects each EVPN route type in detail — the NLRI format, what each carries, and how they interact. This is the deepest technical chapter in the book.
