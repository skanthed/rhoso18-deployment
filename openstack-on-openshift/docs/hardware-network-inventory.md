## RHOSO18 — Hardware and Network Configuration

### VLANs and Networks

| VLAN | Network / Purpose | CIDR | Switch-Port Configuration |
| --- | --- | --- | --- |
| 914 | Existing network | — | Untagged on RHOSO service and Ironic host-facing ports, except where removed for Ironic provisioning |
| 20 | RHOSO Internal API | 172.17.0.0/24 | Tagged on OpenShift `eno2` ports (Dell SW2) |
| 22 | RHOSO Tenant | 172.19.0.0/24 | Tagged on OpenShift `eno2` ports (Dell SW2) |
| 83 | Ironic provisioning | 172.20.1.0/24 | Tagged on OpenShift `eno17` ports (Cumulus SW1) and the switch interconnect |


### Jump Host

| Node IP | Role | Dell Switch Ports |
| --- | --- | --- |
| 10.5.1.112 | Jump host | Te1/25, Te1/26 |


### OpenShift Master Nodes

| Node IP | OpenShift Node | Dell Switch Ports | Cumulus Switch Ports |
| --- | --- | --- | --- |
| 10.5.1.12 | master-0 | Te1/21, Te1/22 | swp13s1, swp13s3 |
| 10.5.1.14 | master-1 | Te1/19, Te1/20 | swp12s1, swp12s3 |
| 10.5.1.17 | master-2 | Te1/1, Te1/2 | swp11s0, swp11s2 |


### Ironic Tested/Provisionied Node

| Node IP | Dell Switch Ports | Cumulus Switch Port |
| --- | --- | --- |
| 10.5.1.16 | Te1/17, Te1/18 | swp11s1, Down |


### Additional Nodes

| Node IP | Status | Dell Switch Ports | Cumulus Switch Ports |
| --- | --- | --- | --- |
| 10.5.1.15 | Available | Te1/3, Te1/4 | swp12s0, swp12s2 |
| 10.5.1.111 | Not working | Te1/23, Te1/24 | — |