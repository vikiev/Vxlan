# Chapter 8: BGP EVPN Route Types & Operations

## The Route Types at a Glance

EVPN defines several BGP route types, each carrying specific information. For CCIE DC, you must know Types 1-5 cold.

| Type | Name | Purpose | Critical? |
|------|------|---------|-----------|
| 1 | Ethernet Auto-Discovery (A-D) | Multi-homing, split-horizon | Yes (MH) |
| 2 | MAC/IP Advertisement | MAC and IP reachability | **Critical** |
| 3 | Inclusive Multicast Ethernet Tag | BUM replication list | **Critical** |
| 4 | Ethernet Segment Route | Multi-homing ES discovery | Yes (MH) |
| 5 | IP Prefix Advertisement | Inter-subnet / L3 routes | Yes (routing) |

## Type 2: MAC/IP Advertisement Route

This is the **workhorse** of EVPN. It's how VTEPs learn remote MACs and IPs.

### NLRI Format

```
┌────────────────────────────────────────────────────────────┐
│ Route Distinguisher (8 bytes)                               │
├────────────────────────────────────────────────────────────┤
│ Ethernet Tag ID (4 bytes)                                   │
├────────────────────────────────────────────────────────────┤
│ MAC Address Length (1 byte) — always 48                     │
├────────────────────────────────────────────────────────────┤
│ MAC Address (6 bytes)                                       │
├────────────────────────────────────────────────────────────┤
│ IP Address Length (1 byte) — 0, 32, or 128                 │
├────────────────────────────────────────────────────────────┤
│ IP Address (0, 4, or 16 bytes)                              │
├────────────────────────────────────────────────────────────┤
│ MPLS Label (3 bytes) — repurposed as VNI in VXLAN           │
└────────────────────────────────────────────────────────────┘
```

### What It Carries

A Type 2 route says: *"I have this MAC (and optionally this IP) in this broadcast domain, reachable via my VTEP."*

**Example:**
```
Route Distinguisher: 10.0.0.1:100
Ethernet Tag: 0 (or VNI)
MAC: 00:AA:AA:AA:AA:AA
IP: 10.1.1.5
VNI: 100
Next Hop: 10.0.0.1 (advertising VTEP)
```

### Two Flavors of Type 2

1. **MAC-only**: MAC address advertised without IP
   - Learned from data plane (frame received with unknown src)
   - Or from static configuration
   
2. **MAC+IP**: MAC and IP advertised together
   - Learned from ARP/ND snooping on the VTEP
   - Enables ARP suppression
   - The IP field is what makes ARP suppression possible

### How Type 2 Routes Are Generated

```
VM-A sends ARP: "Who has 10.1.1.6? Tell 10.1.1.5 (MAC: 00:AA:AA:AA:AA:AA)"

Leaf-1 (VTEP) snoops this ARP:
  → Learns: MAC 00:AA:AA:AA:AA:AA ↔ IP 10.1.1.5
  → Generates Type 2 route:
      RD: 10.0.0.1:100
      MAC: 00:AA:AA:AA:AA:AA
      IP: 10.1.1.5
      VNI: 100
  → Advertises via BGP EVPN to all peers (or RR)
```

### Viewing Type 2 Routes (NX-OS)

```
Nexus9K# show bgp l2vpn evpn route-type 2

   Network            Next Hop            Metric     Path
Route Distinguisher: 10.0.0.1:100
*> [2]:[0]:[48]:[00aa.aaaa.aaaa]:[32]:[10.1.1.5]
                      10.0.0.1                         0 65000 i
*> [2]:[0]:[48]:[00bb.bbbb.bbbb]:[32]:[10.1.1.6]
                      10.0.0.2                         0 65000 i

Route Distinguisher: 10.0.0.2:100
*> [2]:[0]:[48]:[00cc.cccc.cccc]:[32]:[10.1.1.7]
                      10.0.0.2                         0 65000 i
```

Reading the notation: `[2]:[ET]:[MAC-len]:[MAC]:[IP-len]:[IP]`

## Type 3: Inclusive Multicast Ethernet Tag Route

Type 3 builds the **BUM replication list**. It says: *"I participate in this broadcast domain. Send BUM traffic to my VTEP IP."*

### NLRI Format

```
┌────────────────────────────────────────────────────────────┐
│ Route Distinguisher (8 bytes)                               │
├────────────────────────────────────────────────────────────┤
│ Ethernet Tag ID (4 bytes)                                   │
├────────────────────────────────────────────────────────────┤
│ IP Address Length (1 byte)                                  │
├────────────────────────────────────────────────────────────┤
│ Originating Router's IP Address (4 or 16 bytes)             │
└────────────────────────────────────────────────────────────┘
```

### How It Works

```
Leaf-1 joins VNI 100:
  → Generates Type 3 route:
      RD: 10.0.0.1:100
      ET: 0 (or VNI)
      Originating IP: 10.0.0.1
  → Advertises via BGP

Leaf-2 joins VNI 100:
  → Generates Type 3 route:
      RD: 10.0.0.2:100
      Originating IP: 10.0.0.2

Leaf-1 receives Leaf-2's Type 3:
  → Adds 10.0.0.2 to VNI 100's BUM peer list
  → BUM traffic for VNI 100 will be sent to 10.0.0.2
```

### Viewing Type 3 Routes

```
Nexus9K# show bgp l2vpn evpn route-type 3

Route Distinguisher: 10.0.0.1:100
*> [3]:[0]:[32]:[10.0.0.1]
                      10.0.0.1                         0 65000 i

Route Distinguisher: 10.0.0.2:100
*> [3]:[0]:[32]:[10.0.0.2]
                      10.0.0.2                         0 65000 i
```

### Type 3 and BUM Delivery

The Type 3 routes define **where BUM goes**:
- **Head-end replication**: Ingress VTEP sends unicast copy to each IP in the Type 3 list
- **P-multicast**: Type 3 may carry a P-multicast group address (via PMSI attribute)

## Type 1: Ethernet Auto-Discovery (A-D) Route

Type 1 is primarily for **multi-homing** (Chapter 12). It has two sub-types:

### Per-ES A-D Route
- Advertised per Ethernet Segment
- Carries the Ethernet Segment Identifier (ESI)
- Used for ES discovery and aliasing

### Per-EVI A-D Route
- Advertised per EVI (per VNI)
- Carries the ESI + Ethernet Tag
- Used for split-horizon filtering and mass MAC withdrawal

### NLRI Format (Per-EVI)

```
┌────────────────────────────────────────────────────────────┐
│ Route Distinguisher (8 bytes)                               │
├────────────────────────────────────────────────────────────┤
│ Ethernet Segment Identifier (10 bytes)                      │
├────────────────────────────────────────────────────────────┤
│ Ethernet Tag ID (4 bytes)                                   │
├────────────────────────────────────────────────────────────┤
│ MPLS Label / VNI (3 bytes)                                  │
└────────────────────────────────────────────────────────────┘
```

**For single-homed VXLAN (most common):** Type 1 routes are still generated but with ESI = all zeros. They're mostly irrelevant.

**For multi-homed VXLAN:** Type 1 is critical for all-active forwarding and split-horizon.

## Type 4: Ethernet Segment Route

Type 4 is used exclusively for **multi-homing**:
- Discovers which PEs share an Ethernet Segment
- Carries the ESI and the originating PE's IP
- Used for Designated Forwarder (DF) election

```
NLRI:
  RD (8 bytes)
  ESI (10 bytes)
  IP Address Length (1 byte)
  Originating Router IP (4/16 bytes)
```

If you're not doing multi-homing, you won't see Type 4 routes.

## Type 5: IP Prefix Advertisement Route

Type 5 carries **IP prefix routes** in EVPN. It's used for:
- Inter-VNI routing (advertising subnets between VNIs)
- External route injection into the EVPN domain
- Default route advertisement

### NLRI Format

```
┌────────────────────────────────────────────────────────────┐
│ Route Distinguisher (8 bytes)                               │
├────────────────────────────────────────────────────────────┤
│ Ethernet Tag ID (4 bytes)                                   │
├────────────────────────────────────────────────────────────┤
│ IP Prefix Length (1 byte)                                   │
├────────────────────────────────────────────────────────────┤
│ IP Prefix (4 or 16 bytes)                                   │
├────────────────────────────────────────────────────────────┤
│ GW IP Address (4 or 16 bytes)                               │
├────────────────────────────────────────────────────────────┤
│ MPLS Label / VNI (3 bytes)                                  │
└────────────────────────────────────────────────────────────┘
```

### Example

```
Type 5 route:
  RD: 10.0.0.1:100
  Prefix: 10.1.1.0/24
  GW: 10.0.0.1
  VNI: 100

Meaning: "To reach subnet 10.1.1.0/24, send to VTEP 10.0.0.1, VNI 100"
```

### Type 5 vs Type 2

| Aspect | Type 2 | Type 5 |
|--------|--------|--------|
| Granularity | Host (/32 MAC+IP) | Prefix (/24, /16, etc.) |
| Use case | L2 forwarding (MAC lookup) | L3 routing (IP lookup) |
| Generated by | ARP/ND snooping | VRF route redistribution |
| Forwarding | MAC table | FIB / VRF routing table |

## BGP Extended Communities in EVPN

EVPN routes carry critical extended communities:

### Route Target (RT)
```
RT: 65000:100    ← Import/export control for EVI 100
```

### Encapsulation Extended Community
```
Encap: VXLAN      ← Tells receiver this is VXLAN (not MPLS)
```

### Default Gateway
```
Default-GW: yes   ← This MAC/IP is the default gateway for the subnet
```

### Router's MAC
```
Router-MAC: 00:11:22:33:44:55   ← The VTEP's MAC for routing
```

### Viewing Extended Communities

```
Nexus9K# show bgp l2vpn evpn route-type 2 10.0.0.1:100 0 48 00aa.aaaa.aaaa 32 10.1.1.5 detail

    Extended Community: RT:65000:100
    Extended Community: Encap:VXLAN
    Extended Community: Router MAC:0011.2233.4455
    Extended Community: Default-gateway
```

## Route Type Interaction: A Complete Example

Let's trace what happens when VM-A (Leaf-1) wants to talk to VM-B (Leaf-2):

```
Time 0: Both VTEPs boot, join VNI 100
  Leaf-1 → Type 3: "I'm in VNI 100, IP 10.0.0.1"
  Leaf-2 → Type 3: "I'm in VNI 100, IP 10.0.0.2"
  Result: Both know each other for BUM

Time 1: VM-A sends ARP "Who has 10.1.1.6?"
  Leaf-1 snoops: MAC-A = 00:AA:AA:AA:AA:AA, IP = 10.1.1.5
  Leaf-1 → Type 2: "MAC 00:AA:AA:AA:AA:AA + IP 10.1.1.5 in VNI 100"
  Leaf-1 also floods ARP to Leaf-2 (BUM via Type 3 peer list)

Time 2: VM-B responds ARP "I'm 10.1.1.6"
  Leaf-2 snoops: MAC-B = 00:BB:BB:BB:BB:BB, IP = 10.1.1.6
  Leaf-2 → Type 2: "MAC 00:BB:BB:BB:BB:BB + IP 10.1.1.6 in VNI 100"

Time 3: VM-A sends data to VM-B
  Leaf-1 looks up MAC-B → learned from Type 2 → remote VTEP 10.0.0.2
  Encapsulates, sends unicast to Leaf-2
  No flooding. Pure unicast.

Time 4: VM-C (also on Leaf-1) ARPs for 10.1.1.6
  Leaf-1 checks: Do I have a MAC/IP route for 10.1.1.6?
  Yes! Type 2 from Leaf-2 says: 10.1.1.6 → MAC 00:BB:BB:BB:BB:BB
  Leaf-1 replies to VM-C: "10.1.1.6 is at 00:BB:BB:BB:BB:BB"
  → ARP SUPPRESSED. No broadcast sent.
```

## Verification Commands

```bash
# All EVPN routes
show bgp l2vpn evpn

# Specific route type
show bgp l2vpn evpn route-type 2
show bgp l2vpn evpn route-type 3
show bgp l2vpn evpn route-type 5

# Routes for a specific VNI
show bgp l2vpn evpn vni 100

# Detailed route with extended communities
show bgp l2vpn evpn route-type 2 <rd> <et> <mac-len> <mac> <ip-len> <ip> detail

# BGP EVPN neighbors
show bgp l2vpn evpn summary

# EVPN routes in RIB (installed routes)
show bgp l2vpn evpn route-type 2 | begin "Route Distinguisher: 10.0.0.1"
```

## Key Takeaways

- Type 2 (MAC/IP) is the most important route — it builds the MAC forwarding table.
- Type 3 (Inclusive Multicast) builds the BUM peer list.
- Type 5 (IP Prefix) enables inter-VNI and external routing.
- Types 1 and 4 are for multi-homing (important but less common in basic deployments).
- Extended communities carry RT, encapsulation type, and gateway information.
- ARP suppression works because Type 2 routes carry MAC+IP bindings.

## What's Next

Chapter 9 ties EVPN and VXLAN together in a complete configuration — from underlay IGP to BGP EVPN peering to VNI creation. You'll see the full picture working as one system.
