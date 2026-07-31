# Chapter 16: VXLAN Automation & Programmability

## Why Automation Is Non-Negotiable

A 3-leaf VXLAN fabric is manageable by hand. A 100-leaf fabric with 500 VNIs is not. Every VNI addition requires configuration on:

- Every VTEP hosting that VNI (VLAN, VNI, NVE member)
- Every VTEP's SVI (if routing)
- The VRF (if inter-VNI)
- BGP (route targets)
- Access ports (VLAN assignment)

Multiply by 500 VNIs × 50 leaves = 25,000 configuration changes. Manual is not an option.

## Automation Landscape for VXLAN

| Tool | Approach | Best For |
|------|----------|----------|
| NX-OS REST API | Programmatic CLI | Custom scripts, integration |
| NETCONF/YANG | Structured config | Model-driven automation |
| Ansible | Agentless playbooks | Day-1 provisioning |
| Terraform | Declarative IaC | Infrastructure as code |
| Cisco NDFC (DCNM) | GUI + API | Full lifecycle management |
| ACI APIC API | Policy model | ACI-specific automation |
| Python + nxos modules | Scripts | Custom workflows |

## NX-OS REST API

The Nexus 9000 exposes a REST API for programmatic configuration:

### Enabling the API

```
feature nxapi
nxapi http port 80
nxapi https port 443
```

### Example: Create a VNI via REST

```python
import requests
import json

switch = "https://10.0.0.1/ins"
headers = {"content-type": "application/json"}
auth = ("admin", "password")

payload = {
    "ins_api": {
        "version": "1.0",
        "type": "cli_conf",
        "chunk": "0",
        "sid": "1",
        "input": "vlan 300 ; vn-segment 300",
        "output_format": "json"
    }
}

response = requests.post(
    switch,
    data=json.dumps(payload),
    headers=headers,
    auth=auth,
    verify=False
)
```

### Example: Query VNI Status

```python
payload = {
    "ins_api": {
        "version": "1.0",
        "type": "cli_show",
        "chunk": "0",
        "sid": "1",
        "input": "show nve vni 100 detail",
        "output_format": "json"
    }
}

response = requests.post(switch, data=json.dumps(payload), headers=headers, auth=auth, verify=False)
vni_data = response.json()["ins_api"]["outputs"]["output"]["body"]
print(f"VNI State: {vni_data['TABLE_nvi']['ROW_nvi']['vni_state']}")
```

## Ansible for VXLAN

### Playbook: Add a VNI Across All Leaves

```yaml
---
- name: Deploy VNI 300 to all leaves
  hosts: leaves
  gather_facts: no
  connection: network_cli
  
  vars:
    vni_id: 300
    vlan_id: 300
    subnet: "10.3.1.1/24"
    vrf: "Tenant-A"
    
  tasks:
    - name: Create VLAN with VNI segment
      cisco.nxos.nxos_vlan:
        vlan_id: "{{ vlan_id }}"
        mapped_vni: "{{ vni_id }}"
        state: present

    - name: Configure SVI
      cisco.nxos.nxos_l3_interface:
        name: "Vlan{{ vlan_id }}"
        vrf: "{{ vrf }}"
        ipv4: "{{ subnet }}"

    - name: Add VNI to NVE
      cisco.nxos.nxos_vxlan_vtep_vni:
        interface: nve1
        vni: "{{ vni_id }}"
        suppress_arp: true
        ingress_replication: bgp

    - name: Save configuration
      cisco.nxos.nxos_save:
```

### Inventory Structure

```ini
[spines]
spine-1 ansible_host=10.0.0.10
spine-2 ansible_host=10.0.0.11

[leaves]
leaf-1 ansible_host=10.0.0.1
leaf-2 ansible_host=10.0.0.2
leaf-3 ansible_host=10.0.0.3

[leaves:vars]
ansible_network_os=cisco.nxos.nxos
ansible_connection=network_cli
ansible_user=admin
ansible_ssh_pass=password
```

## Terraform for VXLAN (Cisco NDFC Provider)

```hcl
terraform {
  required_providers {
    ndfc = {
      source  = "CiscoDevNet/ndfc"
      version = ">= 1.0"
    }
  }
}

provider "ndfc" {
  username = "admin"
  password = "password"
  url      = "https://ndfc-controller:8443"
}

resource "ndfc_vxlan_fabric" "fabric_a" {
  fabric_name = "VXLAN-Fabric-A"
  
  fabric_settings {
    overlay_mode = "cli"
    deployment_type = "Multi-Site"
    routing_protocol = "OSPF"
    anycast_gateway_mac = "0000.1111.1111"
  }
}

resource "ndfc_network" "vni_100" {
  network_name = "VNI-100-Web"
  fabric_name  = ndfc_vxlan_fabric.fabric_a.fabric_name
  
  network_id   = 100
  vlan_id      = 100
  gateway_ip   = "10.1.1.1/24"
  vrf_name     = "Tenant-A"
}
```

## Cisco NDFC (Nexus Dashboard Fabric Controller)

NDFC (formerly DCNM) is Cisco's fabric management platform:

**Capabilities:**
- Day-0: Design fabric topology (spine-leaf, pod)
- Day-1: Provision VRFs, networks (VNIs), policies
- Day-2: Monitor, troubleshoot, compliance
- Day-N: Upgrade, expand, modify

**VXLAN-specific features:**
- Auto-generate underlay config (OSPF/BGP)
- Auto-generate NVE/BGP EVPN config
- VNI lifecycle management (create, modify, delete across all VTEPs)
- Topology visualization with VNI overlay
- Health monitoring per VNI

**API workflow:**
```
POST /appcenter/cisco/ndfc/api/v1/lan-fabric/rest/control/fabrics
  → Create fabric

POST /appcenter/cisco/ndfc/api/v1/lan-fabric/rest/control/networks
  → Create network (VNI)

POST /appcenter/cisco/ndfc/api/v1/lan-fabric/rest/control/networks/deploy
  → Deploy to switches
```

## ACI Automation (APIC API)

ACI is API-first. Everything in the GUI is a REST call:

### Create a Bridge Domain (VXLAN segment)

```python
import requests

apic = "https://apic1"
auth = {"aaaUser": {"attributes": {"name": "admin", "pwd": "password"}}}

# Login
session = requests.post(f"{apic}/api/aaaLogin.json", json=auth, verify=False)
token = session.json()["imdata"][0]["aaaLogin"]["attributes"]["token"]
headers = {"APIC-cookie": token}

# Create Bridge Domain
bd = {
    "fvBD": {
        "attributes": {
            "name": "Web-BD",
            "arpFlood": "yes",
            "unkMacUcastAct": "proxy"
        },
        "children": [
            {
                "fvSubnet": {
                    "attributes": {
                        "ip": "10.1.1.1/24",
                        "scope": "public,shared"
                    }
                }
            }
        ]
    }
}

tenant = "Production"
url = f"{apic}/api/mo/uni/tn-{tenant}/BD-Web-BD.json"
requests.post(url, json=bd, headers=headers, verify=False)
```

### ACI Terraform Provider

```hcl
resource "aci_bridge_domain" "web_bd" {
  tenant_name          = aci_tenant.production.name
  name                 = "Web-BD"
  arp_flood            = "yes"
  unk_mac_ucast_act    = "proxy"
  
  subnet {
    ip    = "10.1.1.1/24"
    scope = ["public", "shared"]
  }
}
```

## Model-Driven Programmability (NETCONF/YANG)

NX-OS supports NETCONF with YANG models:

```xml
<!-- Create a VNI via NETCONF -->
<edit-config>
  <target><running/></target>
  <config>
    <System xmlns="http://cisco.com/ns/yang/cisco-nx-os-device">
      <bd-items>
        <bd-items>
          <BD-list>
            <fabEncap>vlan-300</fabEncap>
            <name>vlan300</name>
            <accEncap>vxlan-300</accEncap>
          </BD-list>
        </bd-items>
      </bd-items>
    </System>
  </config>
</edit-config>
```

## Day-2 Automation: Compliance and Drift Detection

```python
def check_vni_compliance(switch_ip, expected_vnis):
    """Verify all expected VNIs exist and are correctly configured."""
    actual_vnis = get_vni_config(switch_ip)
    
    drift = []
    for vni in expected_vnis:
        if vni["id"] not in actual_vnis:
            drift.append(f"MISSING: VNI {vni['id']}")
        elif actual_vnis[vni["id"]]["suppress_arp"] != True:
            drift.append(f"MISCONFIGURED: VNI {vni['id']} ARP suppression disabled")
    
    return drift
```

## Key Takeaways

- VXLAN at scale requires automation. Manual configuration doesn't work beyond ~10 VNIs.
- NX-OS REST API and NETCONF provide programmatic access to all VXLAN config.
- Ansible is the most common tool for Day-1 VXLAN provisioning.
- NDFC/DCNM provides full lifecycle management with a GUI + API.
- ACI is API-first — everything is a REST call.
- Day-2 automation (compliance, drift detection) is as important as Day-1.
- For CCIE DC: understand the automation tools conceptually; CLI config is still tested in the lab.

## What's Next

Chapter 17 is the troubleshooting mastery chapter — systematic approaches to finding and fixing VXLAN issues.
