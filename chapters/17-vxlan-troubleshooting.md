# Chapter 17: VXLAN Troubleshooting Mastery

## The Troubleshooting Mindset

VXLAN troubleshooting is **layered**. The most common mistake is jumping to the overlay when the problem is in the underlay. Always work bottom-up:

```
Step 1: Physical / Underlay connectivity
Step 2: Underlay routing (IGP, reachability)
Step 3: VTEP / NVE status
Step 4: BGP EVPN sessions
Step 5: EVPN route exchange
Step 6: MAC / ARP tables
Step 7: Data plane forwarding
Step 8: Application / end-host
```

## The Systematic Approach

### Step 1: Underlay Reachability

**Question:** Can VTEPs reach each other at the IP level?

```bash
! From Leaf-1, ping Leaf-2's loopback (VTEP source)
ping 10.0.0.2 source-interface loopback0

! If ping fails:
show ip route 10.0.0.2
show ip ospf neighbors
show interface Ethernet1/49    ← Uplink to spine
show ip ospf interface Ethernet1/49
```

**Common failures:**
- OSPF adjacency not formed (MTU mismatch, area mismatch, auth failure)
- Loopback not advertised (passive interface, wrong area)
- Physical link down
- MTU too small (ping works small, fails large)

**MTU test:**
```bash
ping 10.0.0.2 source-interface loopback0 size 9000 df-bit
! If this fails but small ping works → MTU issue
```

### Step 2: NVE Interface Status

**Question:** Is the VTEP interface up and configured correctly?

```bash
show interface nve1
show nve vni
show nve vni 100 detail
```

**Expected output:**
```
Interface: nve1, State: up, source-interface: loopback0
  VNI: 100, State: up, Multicast-group: 0.0.0.0
  Ingress-replication: BGP, Suppress-ARP: enabled
```

**Common failures:**
- NVE interface administratively down (`shutdown`)
- Source interface wrong (not loopback, or loopback not in IGP)
- VNI not added to NVE (`member vni 100` missing)
- `host-reachability protocol bgp` missing → falls back to flood & learn

### Step 3: BGP EVPN Sessions

**Question:** Are BGP EVPN sessions established?

```bash
show bgp l2vpn evpn summary
```

**Expected:**
```
Neighbor        V    AS   MsgRcvd   MsgSent   State/PfxRcd
10.0.0.10       4 65000       542       538   12
```

**Common failures:**

| Symptom | Cause | Fix |
|---------|-------|-----|
| State: Active | TCP can't connect | Check underlay reachability, port 179 |
| State: Idle | Admin down or policy | `no shutdown` under neighbor |
| State: Idle (Admin) | Neighbor shut | `no shutdown` |
| PfxRcd: 0 | No routes exchanged | Check address-family, send-community |
| Session flapping | MTU, keepalive timeout | Check underlay stability |

**Debug:**
```bash
show bgp l2vpn evpn neighbors 10.0.0.10
show bgp l2vpn evpn neighbors 10.0.0.10 received-routes
debug bgp l2vpn evpn events    ← CAREFUL in production
```

### Step 4: EVPN Route Exchange

**Question:** Are Type 2 and Type 3 routes being received?

```bash
! Check for MAC/IP routes from remote VTEP
show bgp l2vpn evpn route-type 2

! Check for BUM peer routes
show bgp l2vpn evpn route-type 3

! Check routes for specific VNI
show bgp l2vpn evpn vni 100
```

**Common failures:**
- Routes present but not installed → RT mismatch
- No routes at all → `send-community both` missing on neighbor
- Routes filtered → route-map or prefix-list blocking
- RR not reflecting → `route-reflector-client` missing, or `retain route-target all` missing

**RT verification:**
```bash
! Check what RT a route carries
show bgp l2vpn evpn route-type 2 10.0.0.2:100 0 48 00bb.bbbb.bbbb 32 10.1.1.6 detail
! Look for: Extended Community: RT:65000:100

! Check what RTs are imported locally
show bgp l2vpn evpn vni 100
! Look for: Import RT: 65000:100
```

### Step 5: MAC and ARP Tables

**Question:** Does the VTEP know where remote MACs are?

```bash
show mac address-table vlan 100
show ip arp suppression-cache
show ip arp vrf Tenant-A
```

**Expected:**
```
VLAN  MAC Address       Type     Ports
100   00aa.aaaa.aaaa    static   Eth1/1          ← Local
100   00bb.bbbb.bbbb    dynamic  nve1(10.0.0.2)  ← Remote (from EVPN)
```

**Common failures:**
- Remote MAC not in table → EVPN route not received/installed
- MAC shows as "flood" → control plane not working (flood & learn fallback)
- ARP suppression cache empty → `suppress-arp` not configured
- MAC flapping → duplicate MAC, loop, or VM migrating rapidly

### Step 6: Data Plane Verification

**Question:** Are packets actually being encapsulated and forwarded?

```bash
! NVE counters
show interface nve1 counters

! Check for drops
show interface nve1 counters errors
show system internal nve stats

! Platform-specific
show hardware internal access-list manager stats
show platform internal dme vxlan stats
```

**Key counters:**
- `encap-success`: Packets successfully encapsulated
- `decap-success`: Packets successfully decapsulated
- `encap-fail`: Encapsulation failures (MTU? Buffer?)
- `decap-fail`: Decapsulation failures (bad VNI? bad header?)
- `unknown-vni`: Received VXLAN with unrecognized VNI

### Step 7: Packet Capture

When all else fails, capture packets:

```bash
! NX-OS embedded packet capture (ERSPAN)
monitor session 1 type erspan-source
  source interface Ethernet1/1 both
  destination
    erspan-id 1
    ip address 10.99.99.1
    origin ip address 10.0.0.1

! Or use Wireshark on a SPAN port
! Filter: udp.port == 4789
```

**What to look for in captures:**
- Outer src/dst IP correct? (VTEP loopbacks)
- UDP dst port 4789?
- VNI correct in VXLAN header?
- I-flag set?
- Inner frame intact?
- UDP source port varying? (ECMP)

## Common Troubleshooting Scenarios

### Scenario 1: "VMs in same VNI can't ping"

```
Diagnostic path:
1. ping 10.0.0.2 source lo0        → Underlay OK?
2. show nve vni 100                 → VNI up on both VTEPs?
3. show bgp l2vpn evpn summary      → BGP session up?
4. show bgp l2vpn evpn route-type 2 → MAC routes exchanged?
5. show mac address-table vlan 100  → Remote MAC learned?
6. show ip arp suppression-cache    → ARP entry present?
7. Packet capture on ingress VTEP   → Is encap happening?
8. Packet capture on egress VTEP    → Is decap happening?
```

### Scenario 2: "Inter-VNI routing not working"

```
Diagnostic path:
1. show vrf Tenant-A                → VRF exists? VNI assigned?
2. show ip route vrf Tenant-A       → Remote subnet in routing table?
3. show bgp l2vpn evpn route-type 5 → Type 5 routes received?
4. show nve vni 10000               → L3 VNI up?
5. show ip arp vrf Tenant-A         → Remote host ARP entry?
6. Verify SVI is up: show interface vlan 100
7. Verify VRF membership: show interface vlan 100 | include vrf
```

### Scenario 3: "BUM traffic is excessive"

```
Diagnostic path:
1. show interface nve1 counters     → BUM counters high?
2. show ip arp suppression-cache    → Is suppression working?
3. show nve vni 100 detail          → suppress-arp enabled?
4. show bgp l2vpn evpn route-type 3 → How many BUM peers?
5. Check for ARP storms: show logging | grep ARP
6. Verify unknown-unicast-drop is configured
7. Check for duplicate IPs (causes ARP chaos)
```

### Scenario 4: "ECMP load balancing is uneven"

```
Diagnostic path:
1. Packet capture → Is UDP source port varying?
2. show policy-map interface Ethernet1/49 → Hash fields correct?
3. show ip cef 10.0.0.2 → How many ECMP paths?
4. Are there enough flows? (few elephant flows = bad distribution)
5. Check underlay hash configuration:
   show system internal switch-profile hash-info
```

### Scenario 5: "VXLAN works for small packets, fails for large"

```
This is ALWAYS an MTU issue.

1. ping 10.0.0.2 size 1472 df-bit   → Fails? MTU too small
2. show interface Ethernet1/49 | include MTU
3. Check EVERY hop: spine interfaces too
4. Remember: 1500 + 50 (VXLAN) = 1550 minimum
5. With inner VLAN tag: 1554
6. Fix: set MTU 9216 on ALL underlay interfaces
```

## Troubleshooting Tools Summary

| Tool | Use Case |
|------|----------|
| `ping` / `traceroute` | Underlay reachability |
| `show nve vni/peers` | VTEP and VNI status |
| `show bgp l2vpn evpn` | Control plane routes |
| `show mac address-table` | L2 forwarding table |
| `show ip arp suppression-cache` | ARP state |
| `show interface nve1 counters` | Data plane stats |
| ERSPAN / packet capture | Deep packet inspection |
| `show logging` | Events, errors, MAC moves |
| `debug bgp l2vpn evpn` | Route exchange (use cautiously) |
| `show system internal nve stats` | Internal VTEP statistics |

## Key Takeaways

- Always troubleshoot bottom-up: underlay → VTEP → BGP → EVPN → MAC → traffic.
- 80% of VXLAN issues are underlay problems (MTU, reachability, IGP).
- BGP EVPN issues are usually: missing `send-community both`, wrong RT, or RR misconfiguration.
- ARP suppression and unknown unicast drop are your friends — verify they're enabled.
- Packet captures are the ultimate truth — when in doubt, capture.
- MTU issues are the #1 data plane problem. Always test with large packets + DF bit.
- For the CCIE lab: practice this systematic approach until it's muscle memory.

## What's Next

Chapter 18 brings it all together with CCIE DC lab scenarios and exam preparation strategy.
