# Harvester

Ansible playbooks for operating Harvester itself and for provisioning VMs on
Harvester as Ansible-managed Kubernetes/Rancher nodes.

## Playbooks

| Playbook | Purpose |
|----------|---------|
| `upgrade-harvester.yml` | Upgrade a running Harvester cluster and restart VMs afterward. |
| `create-vms-playbook.yml` | Provision VMs via the `tofu/harvester/modules/vm` OpenTofu module and generate a static Ansible inventory from the output. |
| `harvester-rancher-playbook.yml` | End-to-end: provision Harvester VMs → install RKE2 → deploy Rancher, chaining unmodified existing playbooks. |
| `import-downstream-playbook.yml` | Register an already-provisioned RKE2/K3s cluster as a downstream cluster in an existing Rancher instance. |
| `harvester-downstream-playbook.yml` | End-to-end: provision a new set of Harvester VMs → install RKE2 → import that cluster as downstream into an existing Rancher instance. |

## Provisioning VMs and installing Rancher

`harvester-rancher-playbook.yml` reuses three existing pieces of automation
end-to-end, without modifying any of them:

```text
create-vms-playbook.yml                     (this directory)
  -> tofu/harvester/modules/vm              tofu apply (VMs, SSH key, cloud-init)
  -> scripts/generate_inventory.py          writes ansible/rke2/default/inventory/inventory.yml
                                             (same contract/location used for AWS/GCP)

ansible/rke2/default/rke2-playbook.yml      installs and forms the RKE2 cluster
                                             (kube_api_host/fqdn come from the
                                             inventory's `all.vars`, same as any
                                             other provider — no changes needed)

ansible/rancher/default-ha/rancher-playbook.yml   deploys Rancher via Helm
                                             (kubeconfig_file/fqdn defaults
                                             already match what the RKE2 step
                                             produces)
```

### Prerequisites

In addition to the [general ansible prereqs](../README.md):

* Harvester's kubeconfig downloaded as `tofu/harvester/local.yaml` (see
  [tofu/harvester/modules/vm/README.md](../../tofu/harvester/modules/vm/README.md)).
* A `terraform.tfvars` file for `tofu/harvester/modules/vm` (nodes, ssh_key, ...).
* `vars.yaml` files for each stage, copied from their `.example`:
  * `ansible/harvester/vars.yaml` (this directory — `distro`, optional path overrides)
  * `ansible/rke2/default/vars.yaml` (RKE2 install settings)
  * `ansible/rancher/default-ha/vars.yaml` (Rancher chart/version settings)

### Running the full pipeline

```bash
cp ansible/harvester/vars.yaml.example ansible/harvester/vars.yaml
# edit ansible/harvester/vars.yaml, ansible/rke2/default/vars.yaml,
# ansible/rancher/default-ha/vars.yaml as needed

ansible-playbook ansible/harvester/harvester-rancher-playbook.yml
```

### Running steps individually

```bash
# 1. Provision VMs + generate inventory only
ansible-playbook ansible/harvester/create-vms-playbook.yml

# 2. Install RKE2 (reuses the same playbook as every other provider)
ansible-playbook -i ansible/rke2/default/inventory/inventory.yml \
  ansible/rke2/default/rke2-playbook.yml

# 3. Deploy Rancher (reuses the same playbook as every other provider)
ansible-playbook -i ansible/rke2/default/inventory/inventory.yml \
  ansible/rancher/default-ha/rancher-playbook.yml
```

### K3s instead of RKE2

Set `distro: k3s` in `ansible/harvester/vars.yaml` before running
`create-vms-playbook.yml` (this writes the inventory to
`ansible/k3s/default/inventory` instead), then run
`ansible/k3s/default/k3s-playbook.yml` and
`ansible/rancher/default-ha/rancher-playbook.yml` (with `KUBECONFIG_FILE`
pointed at the k3s playbook's kubeconfig, since its default assumes RKE2)
the same way. `harvester-rancher-playbook.yml` itself only chains the RKE2 path.

## Upgrading Harvester

See [upgrade-harvester.yml](./upgrade-harvester.yml) and
[group_vars/all.yml.example](./group_vars/all.yml.example) for the required
variables (`harvester_host`, `harvester_user`, `harvester_password`,
`target_version`).

## Importing a Harvester cluster as downstream in Rancher

`import-downstream-playbook.yml` reuses the exact same Rancher-API import
logic already used by
[`ansible/rke2/airgap/playbooks/deploy/add-downstream-cluster.yml`](../rke2/airgap/playbooks/deploy/add-downstream-cluster.yml)
(Rancher login, the `tofu/rancher/import` module, registration token, manifest
apply, readiness checks) — unmodified. The only difference is how the target
cluster's kubeconfig is reached: the airgap playbook fetches it through a
bastion host, while this one uses the kubeconfig already written locally by
`ansible/rke2/default/rke2-playbook.yml`, since Harvester VMs are directly
reachable from the Ansible control host.

This creates **two clusters**: one running Rancher (cluster A, built with
`harvester-rancher-playbook.yml`) and a separate one to import as downstream
into it (cluster B). Importing a cluster into the Rancher instance running on
itself is unnecessary — Rancher already manages its own hosting cluster as the
built-in `local` cluster.

```bash
# 1. Stand up cluster A + Rancher (see previous section). Note the fqdn it
#    prints — that's the Rancher instance downstream clusters import into.
ansible-playbook ansible/harvester/harvester-rancher-playbook.yml

# 2. Point terraform.tfvars (or harvester_tfvars_file in vars.yaml) at a
#    *different* set of nodes for cluster B, then provision + install RKE2:
ansible-playbook ansible/harvester/create-vms-playbook.yml
ansible-playbook -i ansible/rke2/default/inventory/inventory.yml \
  ansible/rke2/default/rke2-playbook.yml

# 3. Import cluster B as downstream into cluster A's Rancher:
ansible-playbook ansible/harvester/import-downstream-playbook.yml \
  -e "rancher_hostname=<cluster A fqdn>"
```

Or run steps 2-3 in one go with the orchestrator (requires `rancher_hostname`
to already be set, since it targets a pre-existing Rancher instance):

```bash
ansible-playbook ansible/harvester/harvester-downstream-playbook.yml \
  -e "rancher_hostname=<cluster A fqdn>"
```

See `vars.yaml.example` for all the variables this step accepts
(`rancher_hostname`, `rancher_password`, `cluster_name`,
`harvester_kubeconfig_file`, ...).
