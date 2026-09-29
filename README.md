# Kubernetes Infrastructure

Ansible-based infrastructure management for a three-node Kubernetes cluster and a dedicated NFS storage server.

## Infrastructure

| Node | Role |
|---|---|
| master | Kubernetes control plane, etcd |
| worker1 | Kubernetes worker |
| worker2 | Kubernetes worker |
| nfs | Shared persistent storage |

Kubernetes version: `v1.34.3`

The master is intentionally cordoned (`Ready,SchedulingDisabled`). Application workloads run on the two workers.

## Kubespray
he Kubernetes cluster was provisioned using Kubespray.

Kubespray version:
`v2.26.0-180-gb8541962f`

Source location:
`/home/kubernetes1/Downloads/kubespray`

Cluster inventory:
`inventory/mycluster/inventory.ini`

The cluster-specific group variables currently match the
Kubespray sample inventory.

The original inventory is backed up separately using Ansible Vault
and is not stored in Git.

After running Kubespray, reapply the custom DNS configuration:

```bash
ansible-playbook -i inventory/hosts.yml \
  playbooks/dns-configuration.yml --ask-vault-pass

ansible-playbook -i inventory/hosts.yml \
  playbooks/nodelocaldns-configuration.yml --ask-vault-pass

## Ansible Playbooks

The `playbooks/` directory contains:

- OS maintenance and update playbooks.
- Infrastructure audit and kubectl update playbooks.
- Worker-only DNS scheduling configuration.
- NodeLocal DNS upstream configuration.

## DNS Configuration

CoreDNS and DNS-autoscaler are restricted to worker nodes.

The DNS-autoscaler is not pinned to a specific worker.

NodeLocal DNS forwards cluster DNS queries to CoreDNS and uses `8.8.8.8` and `8.8.4.4` for external queries.

The live NodeLocal DNS configuration differs from the Kubespray-generated manifest. After a Kubespray configuration run, reapply the custom DNS playbooks.

## Applying Custom DNS Configuration

Run from the Ansible project directory:

```bash
ansible-playbook -i inventory/hosts.yml \
  playbooks/dns-configuration.yml \
  --ask-vault-pass

ansible-playbook -i inventory/hosts.yml \
  playbooks/nodelocaldns-configuration.yml \
  --ask-vault-pass
```

Both playbooks are designed to be idempotent.

## Security

Do not commit passwords, private keys, kubeconfigs, Kubernetes Secret values, etcd snapshots or kubeadm certificate credentials.

Sensitive Ansible variables are stored outside Git.

## Version Control

Infrastructure changes are committed locally. Remote publishing and automated deployment will be introduced later as part of the CI/CD pipeline.
