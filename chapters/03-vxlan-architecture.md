# Chapter 3: VXLAN Architecture — VTEPs, VNIs & the Big Picture

## The VXLAN Ecosystem

RFC 7348 defines VXLAN's architecture with a small set of components. Let's meet them all:

```
┌─────────────────────────────────────────────────────────────────┐
│                        VXLAN DOMAIN                              │
│                                                                  │
│  ┌─────────┐    ┌─────────────────────────┐    ┌─────────┐    │
│  │  VTEP   │    │    IP TRANSPORT          │    │  VTEP   │    │
│  │         │    │    (Underlay Network)    │    │         │    │
│  │ ┌─────┐ │    │                          │    │ ┌─────┐ │    │
│  │ │VNI 5│ │════╪══════════════════════════╪════│ │VNI 5│ │    │
│  │ │VNI 7│ │    │   OSPF / IS-IS / BGP    │    │ │VNI 7│ │    │
│  │ │VNI 9│ │    │                          │    │ │VNI 9│ │    │
│  │ └─────┘ │    └─────────────────────────┘    │ └─────┘ │    │
│  └────┬────┘                                    └────┬────┘    │
│       │                                              │          │
│  ┌────┴────┐                                    ┌────┴────┐    │
│  │Servers/ │                                    │Servers/ │    │
│  │  VMs    │                                    │  VMs    │    │
│  └─────────┘                                    └─────────┘    │
└─────────────────────────────────────────────────────────────────┘
```

## Component Deep Dive

### 1. VTEP (VXLAN Tunnel Endpoint)

The VTEP is the workhorse. Every VXLAN packet is either created or destroyed at a VTEP.

**Responsibilities:**
- Encapsulate local frames into VXLAN packets (egress)
- Decapsulate incoming VXLAN packets into local frames (ingress)
- Maintain a mapping of VNI → local ports/interfaces
- Maintain a forwarding table of remote VTEP IPs per VNI
- Participate in the control plane (learn remote MACs, VTEPs)

**Where VTEPs live:**
| Platform | VTEP Implementation |
|----------|-------------------|
| Cisco Nexus 9000 | Hardware ASIC (line-rate) |
| VMware ESXi | vSphere Distributed Switch (software/hardware offload) |
| Linux KVM | Open vSwitch, Linux kernel VXLAN module |
| Arista EOS | Hardware ASIC |
| Cumulus/NVIDIA | Hardware ASIC or software |
| Cisco ACI (APIC) | Managed across all leaf VTEPs |

### 2. VNI (VXLAN Network Identifier)

The VNI is a **24-bit field** that identifies a Layer 2 segment in the overlay.

```
VNI range: 0 to 16,777,215 (2^24 - 1)
Usable:    1 to 16,777,215 (VNI 0 is reserved in some implementations)
```

**Key properties:**
- VNI is locally significant to the VTEP (like a VLAN ID is to a switch)
- The same VNI on two different VTEPs = the same L2 segment
- A VTEP can host multiple VNIs simultaneously
- VNI ↔ VLAN mapping is configured per VTEP (often 1:1, but not required)

**VNI vs VLAN analogy:**

| Concept | Traditional | VXLAN |
|---------|------------|-------|
| Segment ID | VLAN ID (12-bit, 4094) | VNI (24-bit, 16M) |
| Scope | Single switch / STP domain | Across entire IP fabric |
| Mapping | Port → VLAN | Port/Interface → VNI |
| Broadcast domain | Per-VLAN | Per-VNI |
| Isolation | VLAN ACLs | VNI + policy |

### 3. VXLAN Tunnel

A "tunnel" between two VTEPs is not a pre-established circuit. It's a **logical association**:

- VTEP-A knows: "To reach MAC X in VNI 5, send a VXLAN packet to VTEP-B's IP"
- VTEP-B knows: "To reach MAC Y in VNI 5, send a VXLAN packet to VTEP-A's IP"

There's no tunnel setup, no handshake, no state in the underlay. The "tunnel" exists only in the VTEPs' forwarding tables.

```
VTEP-A Forwarding Table (VNI 5):
┌──────────────────┬──────────────────┬─────────────┐
│ Inner Dest MAC   │ Remote VTEP IP   │ Action      │
├──────────────────┼──────────────────┼─────────────┤
│ 00:11:22:33:44:55│ 10.0.0.2        │ Encapsulate │
│ 00:AA:BB:CC:DD:EE│ 10.0.0.3        │ Encapsulate │
│ (local MACs)     │ (local ports)    │ Bridge      │
└──────────────────┴──────────────────┴─────────────┘
```

### 4. The Underlay (IP Transport Network)

The underlay's only job: **deliver IP packets between VTEP addresses**.

Requirements:
- IP reachability between all VTEP loopback/source IPs
- Sufficient MTU (inner frame + 50 bytes overhead)
- Low latency, low jitter (especially for storage traffic)
- ECMP support for load balancing
- No requirement for multicast (if using EVPN or head-end replication)

The underlay is typically:
- A Spine-Leaf CLOS topology
- Running OSPF or IS-IS (or eBGP)
- Using loopback addresses as VTEP source IPs
- Configured with jumbo frames (MTU 9216 recommended)

## VXLAN Operation: Step by Step

Let's trace a complete packet flow:

**Scenario:** VM-A (on Leaf-1, VNI 100) sends a frame to VM-B (on Leaf-2, VNI 100).

### Step 1: VM-A sends a frame
```
Inner Frame:
  Dst MAC: 00:BB:BB:BB:BB:BB (VM-B)
  Src MAC: 00:AA:AA:AA:AA:AA (VM-A)
  EtherType: 0x0800 (IPv4)
  Payload: IP packet (10.1.1.5 → 10.1.1.6)
```

### Step 2: Leaf-1 (VTEP) receives the frame
- Looks up the ingress port → determines VNI 100
- Looks up Dst MAC 00:BB:BB:BB:BB:BB in VNI 100's MAC table
- Finds: remote VTEP IP = 10.0.0.2 (Leaf-2)

### Step 3: Leaf-1 encapsulates
```
Outer Ethernet:
  Dst MAC: Next-hop spine's MAC (from ARP/adjacency)
  Src MAC: Leaf-1's MAC

Outer IP:
  Src: 10.0.0.1 (Leaf-1 loopback / VTEP source)
  Dst: 10.0.0.2 (Leaf-2 loopback / VTEP source)
  Protocol: UDP (17)

UDP:
  Src Port: Hash of inner frame (for ECMP) — e.g., 49152
  Dst Port: 4789 (IANA-assigned VXLAN port)

VXLAN Header:
  Flags: I-bit set (VNI is valid)
  VNI: 100

Inner Frame: (original, unchanged)
  Dst MAC: 00:BB:BB:BB:BB:BB
  Src MAC: 00:AA:AA:AA:AA:AA
  ...
```

### Step 4: Spine forwards
- Spine sees outer IP: dst 10.0.0.2
- Routes it (OSPF/IS-IS) toward Leaf-2
- Never looks inside. Doesn't know VNI 100 exists.

### Step 5: Leaf-2 (VTEP) decapsulates
- Receives UDP packet on port 4789
- Strips outer headers
- Reads VNI 100 → knows which L2 segment
- Looks up inner Dst MAC → finds local port
- Forwards original frame to VM-B

### Step 6: VM-B receives
- Gets a normal Ethernet frame
- Has no idea it traversed an IP network
- Responds normally (reverse path)

## VTEP Source IP: Why Loopbacks Matter

Each VTEP uses a **loopback interface IP** as its VXLAN source address:

```
interface loopback0
  ip address 10.0.0.1/32

interface nve1
  source-interface loopback0
```

Why loopbacks?
- Always up (not tied to a physical port)
- Advertised by the IGP → reachable from anywhere
- Stable identifier for the VTEP
- Survives link failures (as long as *some* path exists)

## Multi-VNI VTEPs

A single VTEP typically hosts many VNIs:

```
interface nve1
  member vni 100
    ingress-replication protocol bgp
  member vni 200
    ingress-replication protocol bgp
  member vni 300
    ingress-replication protocol bgp
```

Each VNI maps to a VLAN (or SVI) locally:

```
vlan 100
  vn-segment 100

interface Vlan100
  ip address 10.1.1.1/24
```

The VTEP maintains separate MAC tables, ARP tables, and forwarding decisions per VNI.

## Hardware vs. Software VTEPs

This distinction matters for performance and for the exam:

| Aspect | Hardware VTEP (Nexus 9000) | Software VTEP (Hypervisor) |
|--------|---------------------------|---------------------------|
| Encap/Decap | ASIC, line-rate | CPU or NIC offload |
| Scale | 100K+ MAC entries | Limited by host resources |
| Latency | ~microseconds | Higher (software path) |
| Features | Full VXLAN + routing | Usually bridge-only |
| Use case | Top-of-rack leaf | Server-attached (less common now) |

In modern designs, the **ToR (Top-of-Rack) switch is the VTEP**. The hypervisor vSwitch is a simple access switch. This is called a **"hardware VTEP"** or **"switch-based VXLAN"** design.

## Key Takeaways

- VTEPs are the only devices that know about VXLAN. Everything between them is plain IP.
- VNIs replace VLANs as the segment identifier (24-bit vs 12-bit).
- Tunnels are logical — just forwarding table entries, not circuits.
- The underlay must provide IP reachability and adequate MTU.
- Loopback IPs serve as stable VTEP identifiers.
- Hardware VTEPs (ASICs) are essential for production performance.

## What's Next

Chapter 4 dissects the VXLAN packet format bit by bit. You'll know exactly what every byte means — critical for troubleshooting and for the lab exam.
