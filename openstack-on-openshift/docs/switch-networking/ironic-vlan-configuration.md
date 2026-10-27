# Switch Networking — Step 02: Ironic VLAN 83 Configuration

## Purpose

Configure VLAN 83 for RHOSO18 Ironic bare-metal provisioning across the OpenShift node interfaces, Cumulus switch, Dell switch, and inter-switch link. Use the local interface and management values appropriate to the deployment.

## Network requirements

| Interface / VLAN | Purpose |
|---|---|
| `eno1` | OpenShift primary networking |
| `eno2` | RHOSO control plane and service networks |
| `eno17` | Dedicated Ironic provisioning connection |
| `eno18` | Reserved / unmanaged |
| VLAN 83 | Ironic provisioning (tagged across trunk links) |
| VLAN 914 | Existing untagged network on OpenShift-facing switch ports |

## 1. Identify OpenShift Ironic interfaces and switch ports

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

## 2. Allow tagged VLAN 83 on the Cumulus OpenShift-facing ports

Confirm VLAN 83 is present on the bridge, and add it if needed:

```bash
sudo nv show bridge domain br_default vlan
sudo nv set bridge domain br_default vlan 83
```

Configure the identified ports (the commands below use generic placeholders):

```bash
sudo nv set interface <eno17-port-node-0> bridge domain br_default vlan 83
sudo nv set interface <eno17-port-node-1> bridge domain br_default vlan 83
sudo nv set interface <eno17-port-node-2> bridge domain br_default vlan 83
sudo nv config diff
sudo nv config apply
```

Retain the existing VLAN 914 PVID/untagged membership on these ports; VLAN 83 is tagged.

Verify each port:

```bash
bridge vlan show dev <eno17-port-node-0>
bridge vlan show dev <eno17-port-node-1>
bridge vlan show dev <eno17-port-node-2>
```

Expected membership: VLAN 83 tagged and VLAN 914 PVID/egress untagged.

## 3. Identify the inter-switch uplink

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

## 4. Configure VLAN 83 on the Dell uplink

On the Dell switch, inspect the existing VLAN:

```text
show running-config interface vlan 83
```

Tag the uplink in VLAN 83:

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

Keep existing VLAN memberships required by the environment.

## 5. Configure Neutron GenericSwitch trunk definitions

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
