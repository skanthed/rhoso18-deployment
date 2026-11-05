# Switch Networking — VLAN Configuration

## Overview

This document covers all physical switch configuration required for an RHOSO18 deployment:

- **Part 1** — VLANs 20 and 22 on the Dell switch (OpenShift node `eno2` interfaces)
- **Part 2** — VLAN 83 on the Cumulus switch (OpenShift node `eno17` interfaces), including the inter-switch uplink to the Dell switch and Neutron GenericSwitch configuration

| VLAN | Purpose | Interface | Switch |
|---|---|---|---|
| 20 | RHOSO Internal API Network | `eno2` | Dell |
| 22 | RHOSO Tenant Network | `eno2` | Dell |
| 83 | Ironic Bare-Metal Provisioning | `eno17` | Cumulus → Dell uplink |
| 914 | Pre-configured management network | `eno2`, `eno17` | Both |

VLAN 914 is a pre-configured management network. Preserve its existing VLAN membership when configuring RHOSO services and Ironic provisioning.

---

## Part 1: RHOSO Services — VLANs 20 and 22 (Dell Switch, `eno2`)

### 1.1 Identify Physical Switch Ports

Before configuring VLANs, identify the switch ports connected to each OpenShift node's `eno2` interface.

1. Retrieve the MAC address of `eno2` on each OpenShift node.
2. Identify the corresponding switch port using the switch MAC address table or LLDP neighbors.
3. Record the mapping between the OpenShift nodes and their physical switch ports.

### 1.2 Configure VLANs 20 and 22 on the Dell Switch

Configure VLAN 20 (Internal API) and VLAN 22 (Tenant) as tagged VLANs on the identified switch ports.

#### Step 1: Enter configuration mode

```text
configure
```

#### Step 2: Remove the existing untagged VLAN 914 temporarily

```text
interface vlan 914
 no untagged TenGigabitEthernet <port>
 exit
```

#### Step 3: Enable hybrid mode on the switch port

```text
interface TenGigabitEthernet <port>
 no switchport
 portmode hybrid
 switchport
 no shutdown
 exit
```

#### Step 4: Restore VLAN 914 as untagged

```text
interface vlan 914
 untagged TenGigabitEthernet <port>
 exit
```

Repeat Steps 2–4 for each identified switch port that requires hybrid mode.

#### Step 5: Configure VLAN 20 as tagged

```text
interface vlan 20
 description RHOSO-InternalAPI
 tagged TenGigabitEthernet <port-1>
 tagged TenGigabitEthernet <port-2>
 tagged TenGigabitEthernet <port-3>
 no shutdown
 exit
```

#### Step 6: Configure VLAN 22 as tagged

```text
interface vlan 22
 description RHOSO-Tenant
 tagged TenGigabitEthernet <port-1>
 tagged TenGigabitEthernet <port-2>
 tagged TenGigabitEthernet <port-3>
 no shutdown
 exit
```

#### Step 7: Save the configuration

```text
end
write
```

#### Step 8: Verify VLAN membership

```text
show interfaces switchport TenGigabitEthernet <port>
```

Expected output:

```text
802.1QTagged: Hybrid

Vlan membership:
U       914
T       20,22

Native VlanId: 914
```

### 1.3 Expected VLAN Configuration (Dell Switch, `eno2` Ports)

Each identified switch port should show:

| VLAN | Mode | Purpose |
|---|---|---|
| 914 | Untagged | Management Network |
| 20 | Tagged | RHOSO Internal API |
| 22 | Tagged | RHOSO Tenant Network |

This configuration allows RHOSO Internal API and Tenant traffic to share the physical `eno2` interface while maintaining separate Layer 2 networks.

---

## Part 2: Ironic Provisioning — VLAN 83 (Cumulus Switch, `eno17`)

### 2.1 Identify OpenShift Ironic Interfaces and Switch Ports

From an OpenShift-admin terminal, identify the `eno17` MAC on each node:

```bash
oc get nodes
oc debug node/<node-name> -- chroot /host cat /sys/class/net/eno17/address
```

On the Cumulus switch, map each MAC address to its connected port:

```bash
sudo bridge fdb show | grep -i '<eno17-mac-address>'
sudo lldpctl
bridge vlan show dev <eno17-switch-port>
```

Repeat for all three OpenShift nodes and record the mapping locally.

### 2.2 Allow Tagged VLAN 83 on the Cumulus OpenShift-Facing Ports

Confirm VLAN 83 is present on the bridge, and add it if needed:

```bash
sudo nv show bridge domain br_default vlan
sudo nv set bridge domain br_default vlan 83
```

**Configure VLAN 83 as tagged on the identified ports** (commands below use generic placeholders):

```bash
sudo nv set interface <eno17-port-node-0> bridge domain br_default vlan 83
sudo nv set interface <eno17-port-node-1> bridge domain br_default vlan 83
sudo nv set interface <eno17-port-node-2> bridge domain br_default vlan 83
sudo nv config diff
sudo nv config apply
```

**Configure VLAN 914 as untagged (PVID) on the identified ports:**

```bash
sudo nv set interface <eno17-port-node-0> bridge domain br_default untagged 914
sudo nv set interface <eno17-port-node-1> bridge domain br_default untagged 914
sudo nv set interface <eno17-port-node-2> bridge domain br_default untagged 914
sudo nv config diff
sudo nv config apply
```

**Verify each port:**

```bash
bridge vlan show dev <eno17-port-node-0>
bridge vlan show dev <eno17-port-node-1>
bridge vlan show dev <eno17-port-node-2>
```

Expected output per port:

```text
83
914 PVID Egress Untagged
```


### 2.3 Identify the Inter-Switch Uplink

On Cumulus:

```bash
sudo lldpcli show neighbors
```

On Dell:

```text
show lldp neighbors
show running-config interface tengigabitethernet <uplink-port>
```

Identify both ends of the physical Cumulus–Dell inter-switch link.

### 2.4 Configure VLAN 83 on the Dell Uplink

On the Dell switch, inspect the existing VLAN configuration:

```text
show running-config interface vlan 83
```

Tag the uplink port in VLAN 83:

```text
configure
interface vlan 83
 tagged TenGigabitEthernet <uplink-port>
 no shutdown
 exit
end
write
```

Verify:

```text
show running-config interface vlan 83
show vlan
```

Keep all existing VLAN memberships required by the environment.

### 2.5 Configure Neutron GenericSwitch Trunk Definitions

Within the Neutron GenericSwitch configuration, identify the inter-switch interfaces as trunk ports. Keep switch management addresses, usernames, passwords, and secrets out of public documentation.

```ini
[genericswitch:cumulus-switch]
device_type = netmiko_cumulus_linux
ngs_manage_vlans = True
ngs_save_configuration = True
ngs_trunk_ports = <cumulus-uplink>

[genericswitch:dell-switch]
device_type = netmiko_dell_force10
ngs_manage_vlans = True
ngs_save_configuration = True
ngs_trunk_ports = <dell-uplink>
```

Use the existing secret-backed GenericSwitch configuration to provide management connections and authentication.
