# VXLAN Mastery - The Complete CCIE Data Center Guide

> From "What is a VLAN?" to designing multi-site EVPN-VXLAN fabrics — everything a CCIE DC candidate needs, in one place.

## Who This Is For

- CCIE Data Center candidates (350-601 DCCOR + 300-630 DCACI)
- Network engineers transitioning from traditional DC to fabric architectures
- Anyone who wants to *truly understand* VXLAN, not just configure it

## How to Use This Book

Each chapter builds on the previous one. The material flows from **why** VXLAN exists, through **how** it works at the packet level, into **control plane** deep dives, and finally into **design, operations, and exam strategy**.

- Chapters 1-2: Set the stage. Read them even if you think you know overlays.
- Chapters 3-6: The core mechanics. This is where most candidates have gaps.
- Chapters 7-10: EVPN and routing. The heart of modern VXLAN fabrics.
- Chapters 11-15: Platform, design, security. Where theory meets production.
- Chapters 16-18: Automation, troubleshooting, and exam day.

## Table of Contents

| Chapter | Title | Difficulty |
|---------|-------|-----------|
| [01](chapters/01-the-data-center-revolution.md) | The Data Center Revolution - Why VXLAN Exists | Foundation |
| [02](chapters/02-network-virtualization-overlays.md) | Network Virtualization & Overlay Fundamentals | Foundation |
| [03](chapters/03-vxlan-architecture.md) | VXLAN Architecture - VTEPs, VNIs & the Big Picture | Core |
| [04](chapters/04-vxlan-packet-format.md) | VXLAN Packet Format - Deep Dive into Encapsulation | Core |
| [05](chapters/05-vxlan-data-plane.md) | VXLAN Data Plane - Unicast, Multicast & BUM Traffic | Core |
| [06](chapters/06-vxlan-control-plane.md) | VXLAN Control Plane - Multicast to Head-End Replication | Core |
| [07](chapters/07-evpn-fundamentals.md) | EVPN Fundamentals - The Modern Control Plane | Intermediate |
| [08](chapters/08-bgp-evpn-route-types.md) | BGP EVPN Route Types & Operations | Intermediate |
| [09](chapters/09-evpn-vxlan-integration.md) | EVPN-VXLAN Integration - Tying It All Together | Intermediate |
| [10](chapters/10-vxlan-routing.md) | VXLAN Routing - IRB, Anycast Gateway & Inter-VNI | Intermediate |
| [11](chapters/11-vxlan-cisco-nexus-aci.md) | VXLAN on Cisco Nexus & ACI | Platform |
| [12](chapters/12-vxlan-design-scalability.md) | VXLAN Design & Scalability Considerations | Advanced |
| [13](chapters/13-vxlan-security.md) | VXLAN Security & Hardening | Advanced |
| [14](chapters/14-vxlan-vs-alternatives.md) | VXLAN vs NVGRE vs Geneve vs STT | Advanced |
| [15](chapters/15-multisite-dci.md) | Multi-Site VXLAN & Data Center Interconnect | Advanced |
| [16](chapters/16-vxlan-automation.md) | VXLAN Automation & Programmability | Advanced |
| [17](chapters/17-vxlan-troubleshooting.md) | VXLAN Troubleshooting Mastery | Expert |
| [18](chapters/18-ccie-lab-exam-prep.md) | CCIE DC Lab Scenarios & Exam Preparation | Expert |

## Prerequisites

- Solid understanding of IP routing (OSPF, BGP basics)
- Familiarity with Ethernet switching (VLANs, STP, LACP)
- Basic BGP knowledge (peering, route types, attributes)
- Comfort with CLI (Cisco NX-OS preferred but not required)

## Key Standards & RFCs

| RFC | Topic |
|-----|-------|
| [RFC 7348](https://datatracker.ietf.org/doc/html/rfc7348) | VXLAN: A Framework for Overlaying Virtualized Layer 2 Networks over Layer 3 Networks |
| [RFC 7432](https://datatracker.ietf.org/doc/html/rfc7432) | BGP MPLS-Based Ethernet VPN |
| [RFC 8365](https://datatracker.ietf.org/doc/html/rfc8365) | A Network Virtualization Overlay Solution Using Ethernet VPN (EVPN) |
| [RFC 9135](https://datatracker.ietf.org/doc/html/rfc9135) | An Integrated Routing and Bridging Solution for EVPN |
| [draft-ietf-bess-evpn-inter-subnet-forwarding](https://datatracker.ietf.org/doc/html/draft-ietf-bess-evpn-inter-subnet-forwarding) | Integrated Routing and Bridging in EVPN |

## License

This material is provided for educational purposes. Feel free to use, share, and build upon it.

---

*Built for engineers who want to understand the fabric, not just configure it.*
