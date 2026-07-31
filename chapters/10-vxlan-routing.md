# Chapter 10: VXLAN Routing — IRB, Anycast Gateway & Inter-VNI

## The Routing Problem in VXLAN

So far, we've discussed VXLAN as a Layer 2 technology — extending broadcast domains across an IP fabric. But real networks need **routing between subnets**. In VXLAN terms: routing between VNIs.

The question: **Where does the packet get routed?**

There are three models:

| Model | Where Routing Happens | Pros | Cons |
|-------|----------------------|------|------|
| Centralized | Dedicated gateway VTEP | Simple policy | Single point, hairpinning |
| Distributed (IRB) | Every VTEP (ingress) | Optimal paths | Config on all VTEPs |
| Asymmetric IRB | Ingress VTEP | Simple | VNI translation |
| Symmetric IRB | Ingress VTEP | Scalable | Slightly more complex |

## Anycast Gateway: The Foundation

Before routing between VNIs, let's solve routing **within** a VNI. Every VM needs a default gateway. In a VXLAN fabric, the gateway is the VTEP (leaf switch).

### The Problem with Traditional Gateways

```
VM-A on Leaf-1: gateway = 10.1.1.1 (Leaf-1's SVI MAC: 00:11:11:11:11:11)
VM-B on Leaf-2: gateway = 10.1.1.1 (Leaf-2's SVI MAC: 00:22:22:22:22:22)
```

If VM-A migrates to Leaf-2 (vMotion), it still ARPs for 00:11:11:11:11:11. Leaf-2 doesn't own that MAC. Traffic breaks.

### The Anycast Gateway Solution

**All VTEPs in the same VNI share the same gateway IP AND the same gateway MAC.**

```
Leaf-1 SVI: IP 10.1.1.1, MAC 00:00:11:11:11:11 (virtual)
Leaf-2 SVI: IP 10.1.1.1, MAC 00:00:11:11:11:11 (virtual)
```

Now VM-A can migrate anywhere. It ARPs for 10.1.1.1, gets MAC 00:00:11:11:11:11, and the local VTEP answers. No traffic traverses the fabric just to reach the gateway.

### Configuration (NX-OS)

```
! Define the virtual MAC (same on all leaves)
fabric forwarding anycast-gateway-mac 0000.1111.1111

! Enable on the SVI
interface Vlan100
  ip address 10.1.1.1/24
  fabric forwarding mode anycast-gw
```

### How Anycast GW Works

```
VM-A sends ARP: "Who has 10.1.1.1?"
  → Leaf-1 sees ARP for its own SVI IP
  → Replies with anycast MAC 00:00:11:11:11:11
  → VM-A uses this MAC as gateway

VM-A sends packet to external destination:
  → Dst MAC: 00:00:11:11:11:11 (anycast GW)
  → Leaf-1 receives, recognizes its own anycast MAC
  → Routes the packet (L3 lookup)
```

## Integrated Routing and Bridging (IRB)

IRB means a device does **both L2 bridging and L3 routing**. In VXLAN, every VTEP with an SVI is an IRB device — it bridges within a VNI and routes between VNIs.

### Asymmetric IRB

In asymmetric IRB, routing happens at the **ingress VTEP**, and the packet is sent to the destination VNI directly.

```
VM-A (VNI 100, 10.1.1.5) → VM-C (VNI 200, 10.2.1.5)

1. VM-A sends to gateway MAC (anycast)
2. Leaf-1 receives on VNI 100
3. Leaf-1 routes: 10.2.1.5 → VNI 200, next-hop VTEP = Leaf-2
4. Leaf-1 encapsulates with VNI 200 (destination VNI)
5. Leaf-2 receives on VNI 200, bridges to VM-C
```

**Problem:** Leaf-1 must have VNI 200 configured (to encapsulate with it), even if it has no local hosts in VNI 200. At scale, every VTEP needs every VNI. Doesn't scale.

### Symmetric IRB (Preferred)

In symmetric IRB, routing happens at the **ingress VTEP**, but the packet is sent with a **transit VNI** (or the source VNI), and the **egress VTEP does the final routing**.

Actually, let me be more precise. In the Cisco/NX-OS implementation of symmetric IRB:

```
VM-A (VNI 100, 10.1.1.5) → VM-C (VNI 200, 10.2.1.5)

1. VM-A sends to gateway MAC (anycast)
2. Leaf-1 receives on VNI 100
3. Leaf-1 routes: 10.2.1.5 → next-hop is Leaf-2 (from EVPN Type 5 or Type 2)
4. Leaf-1 encapsulates with VNI 200 (the destination's VNI)
   BUT: Leaf-1 uses the destination VTEP's Router-MAC as inner dst MAC
5. Leaf-2 receives, decapsulates VNI 200
6. Leaf-2 sees inner dst MAC = Leaf-2's Router-MAC → routes locally
7. Leaf-2 bridges to VM-C on VNI 200
```

Wait — let me clarify the actual NX-OS behavior:

**Symmetric IRB in NX-OS:**
- Ingress VTEP routes the packet
- Looks up destination IP in the VRF
- Finds the remote MAC (from EVPN Type 2 MAC/IP route)
- Encapsulates with the **destination VNI**
- Uses the **remote host's MAC** as inner destination
- Egress VTEP just bridges (no routing needed on egress)

This means: **only the ingress VTEP needs the VRF and routing table.** The egress VTEP just delivers the frame. This is truly symmetric because either VTEP can be ingress or egress.

### Configuration for Inter-VNI Routing

```
! Create VRF for the tenant
vrf context Tenant-A
  vni 10000
  address-family ipv4 unicast
    route-target both auto
    route-target both auto evpn

! VLAN/VNI for subnet 1
vlan 100
  vn-segment 100

! VLAN/VNI for subnet 2
vlan 200
  vn-segment 200

! SVI for VNI 100
interface Vlan100
  no shutdown
  vrf member Tenant-A
  ip address 10.1.1.1/24
  fabric forwarding mode anycast-gw

! SVI for VNI 200
interface Vlan200
  no shutdown
  vrf member Tenant-A
  ip address 10.2.1.1/24
  fabric forwarding mode anycast-gw

! L3 VNI (for inter-VNI transit)
vlan 10000
  vn-segment 10000

interface nve1
  member vni 10000 associate-vrf

! NVE for L2 VNIs
interface nve1
  member vni 100
    suppress-arp
    ingress-replication protocol bgp
  member vni 200
    suppress-arp
    ingress-replication protocol bgp
```

### The L3 VNI (Transit VNI)

For inter-VNI routing, a **dedicated L3 VNI** is used:

```
VRF Tenant-A → VNI 10000 (L3 VNI)
```

- The L3 VNI identifies the VRF in the overlay
- Inter-VNI routed packets are encapsulated with the L3 VNI
- Remote VTEPs use the L3 VNI to identify which VRF to route into
- The L3 VNI is NOT a broadcast domain — it's a routing identifier

```
interface nve1
  member vni 10000 associate-vrf    ← Associates L3 VNI with VRF
```

## Inter-VNI Routing: Packet Flow

```
VM-A (10.1.1.5, VNI 100, Leaf-1) → VM-C (10.2.1.5, VNI 200, Leaf-2)

1. VM-A sends packet: dst IP 10.2.1.5, dst MAC = anycast GW
2. Leaf-1 receives on Vlan100 (VNI 100)
3. Leaf-1 routes in VRF Tenant-A:
   - Lookup 10.2.1.5 → EVPN Type 2 route from Leaf-2
   - Next-hop: 10.0.0.2 (Leaf-2 VTEP)
   - Inner dst MAC: 00:CC:CC:CC:CC:CC (VM-C's MAC, from Type 2)
4. Leaf-1 encapsulates:
   - Outer dst IP: 10.0.0.2
   - VNI: 10000 (L3 VNI for VRF Tenant-A)
   - Inner frame: src MAC = Leaf-1 Router-MAC, dst MAC = VM-C MAC
5. Leaf-2 receives, decapsulates VNI 10000
6. Leaf-2 associates VNI 10000 → VRF Tenant-A
7. Leaf-2 routes: 10.2.1.5 → local, Vlan200
8. Leaf-2 bridges to VM-C on Eth1/2
```

## EVPN Type 5 Routes for Inter-VNI

When a VTEP has a VRF with connected subnets, it advertises them as **Type 5 (IP Prefix)** routes:

```
Leaf-1 advertises:
  Type 5: RD 10.0.0.1:10000, prefix 10.1.1.0/24, GW 10.0.0.1, VNI 10000

Leaf-2 advertises:
  Type 5: RD 10.0.0.2:10000, prefix 10.2.1.0/24, GW 10.0.0.2, VNI 10000
```

Remote VTEPs install these in their VRF routing table:
```
Leaf-1 VRF Tenant-A routing table:
  10.1.1.0/24 → directly connected, Vlan100
  10.2.1.0/24 → via 10.0.0.2 (EVPN), VNI 10000
```

## Verification

```bash
# VRF configuration
show vrf Tenant-A

# VRF routing table
show ip route vrf Tenant-A

# L3 VNI status
show nve vni 10000

# EVPN Type 5 routes
show bgp l2vpn evpn route-type 5

# VRF neighbors (inter-VNI)
show ip arp vrf Tenant-A

# Anycast gateway MAC
show fabric forwarding anycast-gw
```

## Design Considerations

| Decision | Recommendation | Rationale |
|----------|---------------|-----------|
| IRB type | Symmetric | Scales better, less config per VTEP |
| Gateway | Anycast | VM mobility, local routing |
| L3 VNI | One per VRF | Clean separation, easy troubleshooting |
| Default route | Type 5 from border | Controlled egress |
| Inter-VNI policy | VRF route-targets | Flexible, standard BGP |

## Key Takeaways

- Anycast Gateway gives all VTEPs the same GW IP+MAC → seamless VM mobility.
- Symmetric IRB routes at ingress, delivers at egress → scales to thousands of VNIs.
- The L3 VNI (transit VNI) identifies the VRF in the overlay.
- EVPN Type 5 routes distribute inter-VNI prefixes.
- Every VTEP that needs to route must have the VRF + SVIs + L3 VNI configured.
- For CCIE DC: inter-VNI routing configuration is a guaranteed lab topic.

## What's Next

Chapter 11 explores how all of this maps to Cisco's platforms — Nexus 9000 standalone NX-OS and ACI fabric.
