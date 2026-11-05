# rhoso18-openstack Helm Chart

Generates Kubernetes manifests for a RHOSO18 deployment from a single
`values.yaml`. Use `helm template` to render manifests; apply them with
`oc apply`. Do not use `helm install` or `helm upgrade`.

---

## Generate Manifests

Edit environment-specific parameters, then run from the `openstack-on-openshift/` directory:

```bash
vi helm/rhoso18-openstack/values.yaml

helm template rhoso18-openstack helm/rhoso18-openstack/ \
  --values helm/rhoso18-openstack/values.yaml \
  --output-dir generated/
```

The `generated/` directory is created automatically and is excluded from Git.
Regenerate any time `values.yaml` changes.

Generated files:

```
generated/rhoso18-openstack/templates/
├── networking/
│   ├── nncp.yaml
│   ├── net-attach-def.yaml
│   ├── ipaddresspools.yaml
│   └── l2advertisement.yaml
├── storage/
│   └── lvmcluster.yaml
└── control-plane/
    ├── openstack-control-plane.yaml
    └── dnsmasq-dns-ironic.yaml
```

---

## Another Environment

```bash
cp helm/rhoso18-openstack/values.yaml helm/rhoso18-openstack/values-<env>.yaml
# Edit values-<env>.yaml

helm template rhoso18-openstack helm/rhoso18-openstack/ \
  --values helm/rhoso18-openstack/values-<env>.yaml \
  --output-dir generated/
```

---

See the [root README](../../README.md) for the full deployment procedure.
