## RHOSO18 OpenStack on OpenShift Deployment Guide

This document provides a step-by-step guide for deploying Red Hat OpenStack
Services on OpenShift (RHOSO18). It captures the exact sequence of actions,
configurations, and validation steps required for a repeatable deployment.

Helm is used only to generate Kubernetes manifests from
`helm/rhoso18-openstack/values.yaml`. Apply generated manifests with `oc apply`.
Do not use `helm install` or `helm upgrade`.

---

#### 1. Apply Required Operator Manifests

```bash
oc apply -f prerequisites/operators/cert-manager-install.yaml
oc apply -f prerequisites/operators/cluster-observability-install.yaml
oc apply -f prerequisites/operators/metallb-install.yaml
oc apply -f prerequisites/operators/metallb-instance.yaml
oc apply -f prerequisites/operators/nmstate-install.yaml
oc apply -f prerequisites/operators/nmstate-instance.yaml
oc apply -f prerequisites/storage/lvms-install.yaml
```

Install the OpenStack Operator from the OpenShift OperatorHub. The operator is
ready when its ClusterServiceVersion shows `Phase: Succeeded`.

##### Verify:
```bash
oc get csv --all-namespaces | grep -E 'cert-manager|nmstate|metallb|observability|lvms|openstack'
oc get pods -A | grep -E 'cert-manager|nmstate|metallb|observability'
```

---

#### 2. Secrets and Service Setup

```bash
oc new-project openstack
oc get namespace openstack -ojsonpath='{.metadata.labels}' | jq
oc label ns openstack security.openshift.io/scc.podSecurityLabelSync=false --overwrite
oc label ns openstack pod-security.kubernetes.io/enforce=privileged --overwrite
oc project openstack
```

Copy the example secret files, populate them with real credentials, and apply.
Never commit files that contain real passwords or switch credentials.

```bash
cp prerequisites/secrets/osp-secret.yaml.example osp-secret.yaml
# Edit osp-secret.yaml — replace every <base64-encoded-password> with a real value:
#   echo -n 'YourPassword' | base64
oc apply -f osp-secret.yaml -n openstack
oc describe secret osp-secret -n openstack

cp prerequisites/secrets/neutron-switch-config.yaml.example neutron-switch-config.yaml
# Edit neutron-switch-config.yaml — fill in switch IPs, credentials, and port names
oc apply -f neutron-switch-config.yaml -n openstack
oc describe secret neutron-switch-config -n openstack
```

---

#### 3. Configure and Generate Deployment YAML

Edit the Helm values file to confirm NICs, VLANs, IPs, storage device paths,
and VIPs for this environment. Replace `<persistent-disk-identifier>` in
`storage.deviceClasses[*].devicePath` with real `/dev/disk/by-path/` paths
before generating.

```bash
vi helm/rhoso18-openstack/values.yaml
```

Generate all deployment manifests:

```bash
helm template rhoso18-openstack helm/rhoso18-openstack/ \
  --values helm/rhoso18-openstack/values.yaml \
  --output-dir generated/
```

The `generated/` directory is created automatically and is excluded from Git.

---

#### 4. Node Network Configuration Policies (NNCP)

```bash
oc apply -f generated/rhoso18-openstack/templates/networking/nncp.yaml
oc get nncp -w
```

Wait until all three NNCPs show `Available` before proceeding.

```bash
oc get nnce
```

---

#### 5. Apply MetalLB, NAD & IP Pools

```bash
oc apply -f generated/rhoso18-openstack/templates/networking/net-attach-def.yaml
oc get net-attach-def -n openstack

oc apply -f generated/rhoso18-openstack/templates/networking/ipaddresspools.yaml
oc get ipaddresspool -n metallb-system
oc describe ipaddresspool -n metallb-system

oc apply -f generated/rhoso18-openstack/templates/networking/l2advertisement.yaml
oc get l2advertisement -n metallb-system
```

###### Enable Global IP Forwarding (OVNKubernetes)

```bash
oc get network.operator cluster --output=jsonpath='{.spec.defaultNetwork.type}'

oc patch network.operator cluster \
  -p '{"spec":{"defaultNetwork":{"ovnKubernetesConfig":{"gatewayConfig":{"ipForwarding":"Global"}}}}}' \
  --type=merge
```

---

#### 6. Configure LVM Storage

Apply node labels so the LVMCluster nodeSelector matches the correct disks on
each node:

```bash
oc apply -f prerequisites/storage/node_labels.yaml
oc get nodes --show-labels
```

Apply the LVMCluster:

```bash
oc apply -f generated/rhoso18-openstack/templates/storage/lvmcluster.yaml
oc get lvmcluster -n openshift-storage
oc get pods -n openshift-storage -o wide | grep vg-manager
```

Confirm the StorageClass exists before applying the control plane:

```bash
oc get storageclass lvms-local-storage-oac
```

---

#### 7. Deploy OpenStack Control Plane

```bash
oc apply -f generated/rhoso18-openstack/templates/control-plane/openstack-control-plane.yaml -n openstack
oc get openstackcontrolplanes -n openstack

oc apply -f generated/rhoso18-openstack/templates/control-plane/dnsmasq-dns-ironic.yaml
```

Verify the deployment from inside the openstackclient pod:

```bash
oc get pods -n openstack | grep -i openstackclient
oc rsh -n openstack openstackclient

# Inside pod
openstack endpoint list
openstack token issue
```

---

#### 8. (Post-Install) Create Ironic Provisioning Network

```bash
oc rsh -n openstack openstackclient

# Create a shared provider network for Ironic provisioning and cleaning traffic
openstack network create \
  --provider-network-type <network_type> \
  --provider-segment <vlan_id> \
  --provider-physical-network <provider_physical_network> \
  --share \
  <network_name>

# Create the Ironic provisioning subnet with DHCP enabled for PXE boot
openstack subnet create \
  --network <network_name> \
  --subnet-range <network_cidr> \
  --ip-version 4 \
  --gateway <gateway_ip> \
  --allocation-pool start=<start_ip>,end=<end_ip> \
  --dhcp \
  --dns-nameserver <dns_ip> \
  <subnet_name>

# Create a router and attach the provisioning subnet
openstack router create <router_name>
openstack router add subnet <router_name> <subnet_name>

# Retrieve the network UUID for the next step
openstack network show <network_name> -f value -c id
```

Update `openstack.ironic.networkUUID` in `helm/rhoso18-openstack/values.yaml`
with the UUID from the last command. Regenerate and reapply the control plane:

```bash
helm template rhoso18-openstack helm/rhoso18-openstack/ \
  --values helm/rhoso18-openstack/values.yaml \
  --output-dir generated/

oc apply -f generated/rhoso18-openstack/templates/control-plane/openstack-control-plane.yaml -n openstack
```

---

#### References

- [Planning your deployment](https://docs.redhat.com/en/documentation/red_hat_openstack_services_on_openshift/18.0/pdf/planning_your_deployment/Red_Hat_OpenStack_Services_on_OpenShift-18.0-Planning_your_deployment-en-US.pdf)
- [Deploying Red Hat OpenStack Services on OpenShift](https://docs.redhat.com/en/documentation/red_hat_openstack_services_on_openshift/18.0/html/deploying_red_hat_openstack_services_on_openshift/index)
- [Configuring the Bare Metal Provisioning Service](https://docs.redhat.com/en/documentation/red_hat_openstack_services_on_openshift/18.0/html/configuring_the_bare_metal_provisioning_service/index)
- [Configuring Networking Services](https://docs.redhat.com/en/documentation/red_hat_openstack_services_on_openshift/18.0/html/configuring_networking_services/index)
