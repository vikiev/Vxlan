# Chapter 13: VXLAN Security & Hardening

## The Security Model of VXLAN

Let's be honest: **VXLAN was not designed with security as a primary goal.** RFC 7348 explicitly states that VXLAN provides no encryption, no authentication, and no integrity checking of the inner frame. The overlay is only as secure as the underlay.

This doesn't mean VXLAN is insecure — it means security must be **designed in**, not assumed.

## Threat Model

### What VXLAN Protects Against (By Design)

| Threat | Protection |
|--------|-----------|
| Tenant A seeing Tenant B's traffic | VNI isolation (separate broadcast domains) |
| MAC spoofing across VNIs | VNI boundary prevents cross-segment L2 |
| Broadcast storms crossing segments | VNI scoping limits blast radius |

### What VXLAN Does NOT Protect Against

| Threat | Risk |
|--------|------|
| Underlay packet sniffing | Inner frame visible in cleartext |
| VTEP impersonation | No authentication of VXLAN packets |
| VNI spoofing | Attacker can craft packets with any VNI |
| Underlay MITM | Outer headers can be modified in transit |
| Rogue VTEP injection | Any device with underlay access can join |
| Control plane attacks | BGP session hijacking, route injection |

## Layer 1: Underlay Security

The underlay is the foundation. If it's compromised, the overlay is compromised.

### Control Plane Protection

```
! Protect BGP sessions
router bgp 65000
  neighbor 10.0.0.10
    password 7 <encrypted-password>
    ttl-security hops 2
    
! Protect OSPF
router ospf UNDERLAY
  authentication message-digest
  area 0.0.0.0 authentication message-digest

! Control Plane Policing (CoPP)
copp profile strict
```

### Data Plane Protection

```
! ACL on underlay interfaces — only allow VXLAN from known VTEPs
ip access-list PROTECT-VXLAN
  permit udp host 10.0.0.1 host 10.0.0.2 eq 4789
  permit udp host 10.0.0.2 host 10.0.0.1 eq 4789
  deny udp any any eq 4789
  permit ip any any

interface Ethernet1/49
  ip access-group PROTECT-VXLAN in
```

### Underlay Hardening Checklist

- [ ] Use loopback IPs as VTEP source (not physical interface IPs)
- [ ] Enable IGP authentication (OSPF MD5/SHA, IS-IS auth)
- [ ] Enable BGP authentication (MD5/AO)
- [ ] Apply CoPP to protect control plane
- [ ] Restrict management plane access (SSH only, ACLs)
- [ ] Disable unused protocols (CDP on underlay links, etc.)
- [ ] Use dedicated management VRF
- [ ] Enable NTP with authentication
- [ ] Log all BGP session changes

## Layer 2: Overlay Security

### VNI Isolation

VNI isolation is the primary tenant boundary. Ensure:

```
! Each VNI maps to exactly one VLAN
vlan 100
  vn-segment 100

! VRFs enforce L3 isolation
vrf context Tenant-A
  vni 10000

vrf context Tenant-B
  vni 20000

! Route targets prevent cross-tenant route leakage
! (auto RT uses VNI, so different VNIs = different RTs)
```

### ARP/ND Suppression and Security

```
interface nve1
  member vni 100
    suppress-arp           ← Prevents ARP spoofing across VTEPs
    
! DHCP snooping (prevents rogue DHCP)
ip dhcp snooping
vlan 100
  ip dhcp snooping

! IP Source Guard (prevents IP spoofing)
interface Ethernet1/1
  ip verify source dhcp-snooping-vlan
```

### Unknown Unicast Drop

```
interface nve1
  member vni 100
    unknown-unicast-drop    ← Don't flood unknown MACs
```

This prevents MAC scanning and limits information leakage.

### MAC Limit and Storm Control

```
! Limit MACs per port (prevent MAC flooding attacks)
interface Ethernet1/1
  switchport port-security maximum 5
  switchport port-security violation restrict

! Storm control
interface Ethernet1/1
  storm-control broadcast level 1.0
  storm-control multicast level 5.0
  storm-control action drop
```

## Layer 3: Control Plane Security (EVPN/BGP)

### BGP Session Protection

```
router bgp 65000
  neighbor 10.0.0.10
    password 7 <key>
    ttl-security hops 2        ← GTSM: only accept from adjacent TTL
    timers 3 9                 ← Fast detection of peer failure
    
  address-family l2vpn evpn
    maximum-prefix 50000 80    ← Alert at 80% of max
```

### Route Filtering

```
! Only accept routes for VNIs this leaf hosts
route-map FILTER-EVPN-IN deny 10
  match vni 999                ← Reject routes for non-existent VNIs
route-map FILTER-EVPN-IN permit 20

router bgp 65000
  neighbor 10.0.0.10
    address-family l2vpn evpn
      route-map FILTER-EVPN-IN in
```

### Route Reflector Security

```
! On RR: prevent route leakage between unrelated tenants
router bgp 65000
  address-family l2vpn evpn
    ! Do NOT use 'retain route-target all' unless needed
    ! Use explicit RT filtering per client
```

## Layer 4: Microsegmentation

Microsegmentation applies security policy at the workload level, regardless of IP/MAC/VLAN.

### In NX-OS (SGT-based)

```
! Security Group Tags
cts role-based sgt-map 00:AA:AA:AA:AA:AA sgt 10
cts role-based sgt-map 00:BB:BB:BB:BB:BB sgt 20

! Policy between groups
cts role-based enforcement
cts role-based policy
  from sgt 10 to sgt 20
    deny ip
  from sgt 20 to sgt 10
    permit tcp eq 443
```

### In ACI (Contract-based)

```
! EPGs define groups
! Contracts define allowed communication
! Default deny between EPGs (whitelist model)

EPG "Web-Servers" → Contract "Allow-HTTPS" → EPG "App-Servers"
EPG "App-Servers" → Contract "Allow-DB" → EPG "DB-Servers"
! All other traffic: denied by default
```

## Layer 5: Encryption

### VXLAN Has No Built-in Encryption

If the underlay traverses untrusted networks (DCI over Internet, shared infrastructure):

| Solution | Layer | Overhead | Complexity |
|----------|-------|----------|-----------|
| IPsec on underlay | Network | High | Medium |
| MACsec on links | Link | Low | Low (hardware) |
| TLS in application | Transport | Medium | Low |
| VXLAN over IPsec tunnel | Network | High | Medium |

### MACsec (802.1AE)

For point-to-point underlay links:

```
interface Ethernet1/49
  macsec mka policy MKA-POLICY
  macsec network-policy NET-POLICY
```

MACsec encrypts at Layer 2, line-rate, with minimal latency. Ideal for spine-leaf links.

## Security Verification

```bash
! Check for unauthorized VTEPs
show nve peers
! Expected: only known VTEP IPs. Any unknown = potential rogue.

! Check for MAC moves (potential spoofing)
show logging | grep "MAC.*move"

! Verify ARP suppression is working
show ip arp suppression-cache
! All remote entries should have '#' flag (COOP/EVPN learned)

! Check BGP session security
show bgp l2vpn evpn summary
! All sessions should be Established, no unexpected peers

! Verify CoPP is active
show copp status

! Check port security violations
show port-security
```

## CCIE DC Security Topics

For the exam, focus on:
1. VNI/VRF isolation (correct RT configuration)
2. ARP suppression and its security implications
3. BGP authentication and GTSM
4. CoPP for control plane protection
5. ACI contracts (default deny, whitelist model)
6. Identifying misconfigurations that break isolation

## Key Takeaways

- VXLAN provides isolation, not encryption. Security is layered.
- Underlay security (IGP auth, BGP auth, CoPP) is the foundation.
- ARP suppression + unknown unicast drop eliminates most L2 attacks.
- Route target filtering prevents cross-tenant route leakage.
- Microsegmentation (SGT/Contracts) adds workload-level policy.
- Encryption (MACsec/IPsec) is needed only for untrusted underlay paths.
- Monitor for rogue VTEPs, MAC moves, and unexpected BGP peers.

## What's Next

Chapter 14 compares VXLAN with alternative overlay technologies — NVGRE, Geneve, and STT.
