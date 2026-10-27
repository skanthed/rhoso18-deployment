# Switch Networking — VLAN Configuration

## Purpose

Configure the required VLANs on the physical switches to support RHOSO18 control plane services, tenant networking, and Ironic bare-metal provisioning.

## Required VLANs

| VLAN | Purpose |
|---|---|
| 20 | RHOSO Internal API Network |
| 22 | RHOSO Tenant Network |
| 83 | Ironic Bare-Metal Provisioning |
| 914 | Existing Untagged Network |

## 1. Identify Physical Switch Ports

Before configuring VLANs, identify the switch ports connected to each OpenShift node's `eno2` interface.

1. Retrieve the MAC address of `eno2` on each OpenShift node.
2. Identify the corresponding switch port using the switch MAC address table or LLDP neighbors.
3. Record the mapping between the OpenShift nodes and their physical switch ports.

## 2. Configure VLANs 20 and 22 on dell switch

Configure VLAN 20 (Internal API) and VLAN 22 (Tenant) as tagged VLANs on the identified switch ports, while preserving VLAN 914 as untagged.

### Step 1: Enter configuration mode

```text
configure
```

### Step 2: Remove the existing untagged VLAN temporarily

```text
interface vlan 914
 no untagged TenGigabitEthernet <port>
 exit
```

### Step 3: Enable hybrid mode on the switch port

```text
interface TenGigabitEthernet <port>
 no switchport
 portmode hybrid
 switchport
 no shutdown
 exit
```

### Step 4: Restore VLAN 914 as untagged

```text
interface vlan 914
 untagged TenGigabitEthernet <port>
 exit
```

Repeat Steps 2–4 for each identified switch port that requires hybrid mode.

### Step 5: Configure VLAN 20 as tagged

```text
interface vlan 20
 description RHOSO-InternalAPI
 tagged TenGigabitEthernet <port-1>
 tagged TenGigabitEthernet <port-2>
 tagged TenGigabitEthernet <port-3>
 no shutdown
 exit
```

### Step 6: Configure VLAN 22 as tagged

```text
interface vlan 22
 description RHOSO-Tenant
 tagged TenGigabitEthernet <port-1>
 tagged TenGigabitEthernet <port-2>
 tagged TenGigabitEthernet <port-3>
 no shutdown
 exit
```

### Step 7: Save the configuration

```text
end
write
```

## 3. Expected VLAN Configuration

Each identified switch port should support:

| VLAN | Mode | Purpose |
|---|---|---|
| 914 | Untagged | Existing Network |
| 20 | Tagged | RHOSO Internal API |
| 22 | Tagged | RHOSO Tenant Network |

This configuration allows RHOSO Internal API and Tenant traffic to share the physical `eno2` interface while maintaining separate Layer 2 networks.
