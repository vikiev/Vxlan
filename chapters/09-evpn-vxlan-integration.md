# Chapter 9: EVPN-VXLAN Integration — Tying It All Together

## The Full Picture

By now you understand VXLAN (encapsulation) and EVPN (control plane) separately. This chapter shows how they integrate into a single, working fabric. We'll build a complete configuration from scratch.

## Reference Topology

```
                    ┌─────────────┐
                    │  Spine-1    │  (Route Reflector)
                    │  10.0.0.10  │
                    └──┬──────┬───┘
                       │      │
              ┌────────┘      └────────┐
              │                         │
        ┌─────┴─────┐           ┌──────┴────┐
        │  Leaf-1    │           │  Leaf-2   │
        │  10.0.0.1  │           │  10.0.0.2 │
        │  VTEP      │           │  VTEP     │
        └─────┬──────┘           └─────┬─────┘
              │                         │
        ┌─────┴─────┐           ┌──────┴────┐
        │  VM-A      │           │  VM-B     │
        │ 10.1.1.5   │           │ 10.1.1.6  │
        │ VNI 100    │           │ VNI 100   │
        └────────────┘           └───────────┘
```

## Step 1: Underlay Configuration

The underlay provides IP reachability between VTEP loopbacks.

### Leaf-1 Underlay

```
feature ospf
feature interface-vlan
feature vn-segment-vlan-based
feature nv overlay
feature vn-segment-vlan-based

! Loopback for VTEP source and BGP peering
interface loopback0
  ip address 10.0.0.1/32
  ip router ospf UNDERLAY area 0.0.0.0

! Uplink to Spine-1
interface Ethernet1/49
  no switchport
  ip address 10.1.49.1/31
  ip router ospf UNDERLAY area 0.0.0.0
  mtu 9216

router ospf UNDERLAY
  router-id 10.0.0.1
```

### Spine-1 Underlay

```
feature ospf

interface loopback0
  ip address 10.0.0.10/32
  ip router ospf UNDERLAY area 0.0.0.0

interface Ethernet1/1
  no switchport
  ip address 10.1.49.0/31
  ip router ospf UNDERLAY area 0.0.0.0
  mtu 9216

interface Ethernet1/2
  no switchport
  ip address 10.1.50.0/31
  ip router ospf UNDERLAY area 0.0.0.0
  mtu 9216

router ospf UNDERLAY
  router-id 10.0.0.10
```

### Verify Underlay

```
Leaf-1# ping 10.0.0.2 source-interface loopback0
Leaf-1# ping 10.0.0.10 source-interface loopback0
Leaf-1# show ip route 10.0.0.2
```

**If underlay ping fails, nothing else will work. Fix this first.**

## Step 2: VLAN and VNI Configuration

Map local VLANs to VNIs.

### Leaf-1

```
! Create VLAN and assign VNI
vlan 100
  vn-segment 100

! Create SVI for the VLAN (gateway)
interface Vlan100
  no shutdown
  ip address 10.1.1.1/24
  fabric forwarding mode anycast-gw

! Access port for VM-A
interface Ethernet1/1
  switchport mode access
  switchport access vlan 100
  spanning-tree port type edge
```

### Leaf-2

```
vlan 100
  vn-segment 100

interface Vlan100
  no shutdown
  ip address 10.1.1.1/24
  fabric forwarding mode anycast-gw

interface Ethernet1/1
  switchport mode access
  switchport access vlan 100
  spanning-tree port type edge
```

**Note:** Both leaves use the same gateway IP (10.1.1.1) — this is the **Anycast Gateway** (covered in Chapter 10). VMs always ARPs for the same MAC regardless of which leaf they're on.

## Step 3: NVE (VXLAN) Interface

The NVE (Network Virtualization Edge) interface is the VTEP.

### Leaf-1

```
interface nve1
  no shutdown
  source-interface loopback0
  host-reachability protocol bgp
  
  member vni 100
    suppress-arp
    ingress-replication protocol bgp
```

### Leaf-2

```
interface nve1
  no shutdown
  source-interface loopback0
  host-reachability protocol bgp
  
  member vni 100
    suppress-arp
    ingress-replication protocol bgp
```

Key commands explained:
- `source-interface loopback0`: VTEP source IP = loopback IP
- `host-reachability protocol bgp`: Use BGP EVPN for MAC/IP learning (not flood & learn)
- `suppress-arp`: Enable ARP suppression (answer ARP from MAC/IP routes)
- `ingress-replication protocol bgp`: BUM peer list from BGP Type 3 routes

## Step 4: BGP EVPN Configuration

### Leaf-1

```
router bgp 65000
  router-id 10.0.0.1
  
  address-family l2vpn evpn
    retain route-target all
  
  neighbor 10.0.0.10
    remote-as 65000
    update-source loopback0
    address-family l2vpn evpn
      send-community both
```

### Leaf-2

```
router bgp 65000
  router-id 10.0.0.2
  
  address-family l2vpn evpn
    retain route-target all
  
  neighbor 10.0.0.10
    remote-as 65000
    update-source loopback0
    address-family l2vpn evpn
      send-community both
```

### Spine-1 (Route Reflector)

```
router bgp 65000
  router-id 10.0.0.10
  
  address-family l2vpn evpn
    retain route-target all
  
  neighbor 10.0.0.1
    remote-as 65000
    update-source loopback0
    address-family l2vpn evpn
      send-community both
      route-reflector-client
  
  neighbor 10.0.0.2
    remote-as 65000
    update-source loopback0
    address-family l2vpn evpn
      send-community both
      route-reflector-client
```

## Step 5: EVPN Instance (VRF-less L2VNI)

On NX-OS, the EVPN instance is auto-created from the VLAN/VNI config. But you can customize:

```
evpn
  vni 100 l2
    rd auto
    route-target import auto
    route-target export auto
```

`rd auto` generates: `<router-id>:<vni>` → `10.0.0.1:100`
`route-target auto` generates: `<asn>:<vni>` → `65000:100`

## Step 6: Verification

### BGP EVPN Session

```
Leaf-1# show bgp l2vpn evpn summary

BGP summary information for VRF default, address family L2VPN EVPN
BGP router identifier 10.0.0.1, local AS number 65000

Neighbor        V    AS   MsgRcvd   MsgSent   TblVer  InQ OutQ  Up/Down  State/PfxRcd
10.0.0.10       4 65000       142       138       45    0    0 01:23:45  4
```

### EVPN Routes

```
Leaf-1# show bgp l2vpn evpn route-type 2

   Network            Next Hop            Metric     Path
Route Distinguisher: 10.0.0.1:100    (L2VNI 100)
*> [2]:[0]:[48]:[00aa.aaaa.aaaa]:[32]:[10.1.1.5]
                      10.0.0.1                         0 65000 i

Route Distinguisher: 10.0.0.2:100    (L2VNI 100)
*> [2]:[0]:[48]:[00bb.bbbb.bbbb]:[32]:[10.1.1.6]
                      10.0.0.2                         0 65000 i
```

### VNI Status

```
Leaf-1# show nve vni 100

Interface VNI      Multicast-group   State Mode Type [BD/VRF]      Flags
--------- -------- ----------------- ----- ---- ------------------ -----
nve1      100      0.0.0.0           Up    CP   L2 [100]
```

### NVE Peers

```
Leaf-1# show nve peers

Interface Peer-IP                                   State LearnType Uptime   Router-Mac
--------- ----------------------------------------  ----- --------- -------- -----------------
nve1      10.0.0.2                                  Up    CP        01:23:45 0011.2233.4455
```

### MAC Table

```
Leaf-1# show mac address-table vlan 100

        VLAN     MAC Address      Type      age     Secure NTFY   Ports
---------+-----------------+--------+---------+------+----+------------------
   100     00aa.aaaa.aaaa   static   -         F      F    Eth1/1
   100     00bb.bbbb.bbbb   dynamic  -         F      F    nve1(10.0.0.2)
```

### ARP Suppression Cache

```
Leaf-1# show ip arp suppression-cache

Flags: * - Adjacencies synced via CFSoE
       # - Adjacencies learned via COOP

IP Address      MAC Address     Age    Flags   Interface     Physical-Interface   VLAN
--------------- --------------- ------ ------- ------------- -------------------- ----
10.1.1.5        00aa.aaaa.aaaa  -      *       Vlan100       Eth1/1               100
10.1.1.6        00bb.bbbb.bbbb  -      #       Vlan100       nve1                 100
```

## Complete Data Flow (With EVPN)

```
VM-A (10.1.1.5) pings VM-B (10.1.1.6):

1. VM-A checks ARP cache → no entry for 10.1.1.6
2. VM-A sends ARP request (broadcast)
3. Leaf-1 receives ARP on Eth1/1 (VLAN 100 → VNI 100)
4. Leaf-1 snoops ARP: learns MAC-A + IP 10.1.1.5
5. Leaf-1 generates Type 2 route (MAC-A + 10.1.1.5) → BGP → RR → Leaf-2
6. Leaf-1 checks: Do I have MAC/IP route for 10.1.1.6?
   → YES (from Leaf-2's Type 2: 10.1.1.6 → 00bb.bbbb.bbbb)
7. Leaf-1 replies to VM-A: "10.1.1.6 is at 00:BB:BB:BB:BB:BB"
   → ARP SUPPRESSED. No broadcast leaves Leaf-1.
8. VM-A sends ICMP to 00:BB:BB:BB:BB:BB
9. Leaf-1 looks up MAC-B → remote, VTEP 10.0.0.2
10. Leaf-1 encapsulates: outer dst=10.0.0.2, VNI=100, UDP sport=hash
11. Spine-1 routes outer packet to Leaf-2
12. Leaf-2 decapsulates, delivers to VM-B
13. VM-B replies (reverse path, same logic)
```

## Common Configuration Mistakes

| Mistake | Symptom | Fix |
|---------|---------|-----|
| Missing `host-reachability protocol bgp` | Flood & learn, no ARP suppression | Add command under nve1 |
| Missing `send-community both` on BGP neighbor | Routes received but no RT → not imported | Add to neighbor config |
| Underlay MTU too small | Large packets dropped | Set MTU 9216 on all underlay interfaces |
| Missing `suppress-arp` | ARP storms, high BUM | Add under member vni |
| Loopback not in IGP | BGP session won't establish | Add loopback to OSPF |
| Wrong ASN on neighbor | BGP session stuck in Active | Verify remote-as matches |
| Missing `retain route-target all` on RR | Routes filtered at RR | Add under address-family |

## Key Takeaways

- EVPN-VXLAN integration requires: underlay IGP + VLAN/VNI + NVE + BGP EVPN.
- The configuration order matters: underlay first, then overlay.
- `host-reachability protocol bgp` is what connects NVE to EVPN.
- ARP suppression eliminates most broadcast traffic.
- Route reflectors scale the control plane; leaves only peer with spines.
- Verification is layered: underlay ping → BGP session → EVPN routes → MAC table → traffic.

## What's Next

Chapter 10 covers VXLAN routing — how traffic moves between VNIs (inter-subnet routing), the Anycast Gateway, and Integrated Routing and Bridging (IRB).
