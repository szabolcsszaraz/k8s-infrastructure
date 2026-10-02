# Kubernetes Infrastructure

## Overview

This repository contains the infrastructure configuration, monitoring configuration, ingress resources, Ansible inventory and maintenance playbooks for the Kubernetes cluster.

The cluster consists of:

- 1 control-plane node
- 2 worker nodes
- 1 external NFS storage server

The environment is designed so that application workloads run on the worker nodes, while the control-plane node remains cordoned.

---

## Servers

| Host | Role | Internal IP |
|---|---|---|
| master | Kubernetes control-plane / etcd | 10.10.147.130 |
| worker1 | Kubernetes worker | 10.10.147.131 |
| worker2 | Kubernetes worker | 10.10.147.132 |
| nfs | NFS storage server | 10.10.147.133 |

Important:

The Kubernetes node object for the master currently reports:

```text
194.160.208.131
```

as its `Internal-IP`.

This is caused by the master having multiple network interfaces.

Do not change the Kubernetes node IP configuration without first checking the current networking setup.

---

## Kubernetes

Current cluster versions:

- Kubernetes: `v1.36.4`
- containerd: `2.3.5`
- Calico: `3.31.7`
- CoreDNS: `1.14.2`
- metrics-server: `0.8.1`
- NodeLocal DNS: `1.25.0`

The master node is intentionally cordoned:

```text
Ready,SchedulingDisabled
```

Application workloads should run on the worker nodes.

---

## Applications

Application namespace:

```text
app
```

Main applications:

- `svk-fe`
- `svk-be`

Both applications currently run with:

```text
2 replicas
```

The application Services use:

```text
ClusterIP
```

External access is handled through `ingress-nginx`.

---

## Application Hosts

Frontend:

```text
svk-ukf.sk
```

Backend API:

```text
api.svk-ukf.sk
```

Grafana:

```text
grafana.svk-ukf.sk
```

---

## Ingress

Ingress controller:

```text
ingress-nginx
```

Current ingress controller Service type:

```text
NodePort
```

Current ports:

```text
HTTP:  32097
HTTPS: 31950
```

The ingress manifests are stored in:

```text
platform/ingress-nginx/
```

Application ingress resources:

```text
platform/ingress-nginx/apps/
├── grafana-ingress.yaml
├── svk-be-ingress.yaml
└── svk-fe-ingress.yaml
```

Ingress installation and configuration can be applied with:

```bash
ansible-playbook \
  -i inventory/hosts.yml \
  playbooks/ingress-nginx.yml \
  --ask-vault-pass
```

The backend has an additional NetworkPolicy allowing traffic from the ingress controller.

The manifest is stored in:

```text
platform/ingress-nginx/network-policies/allow-ingress-to-svk-be.yaml
```

---

## Public Access

Final public WAN access is not configured yet.

After the servers are moved to their final location, the network administrator should configure:

- public WAN IP
- DNS records
- TCP port 80 forwarding
- TCP port 443 forwarding
- TLS certificates

The intended traffic flow is:

```text
Internet
   |
   v
Public IP
   |
   v
TCP 80 / 443
   |
   v
ingress-nginx
   |
   +--> svk-fe
   |
   +--> svk-be
```

Kubernetes API port `6443` should not be exposed publicly unless explicitly required and protected.

---

## Grafana

Grafana runs in the:

```text
monitoring
```

namespace.

Hostname:

```text
grafana.svk-ukf.sk
```

Grafana is intended for internal administration only.

It should not be exposed publicly during final WAN configuration.

Current access is through the ingress controller.

Example internal access:

```text
http://grafana.svk-ukf.sk:32097
```

If DNS is not available internally, a temporary `/etc/hosts` entry can be used.

Example:

```text
10.10.147.131 grafana.svk-ukf.sk
```

---

## Monitoring

The monitoring stack contains:

- Prometheus
- Grafana
- Alertmanager
- Loki
- Grafana Alloy
- kube-state-metrics
- node-exporter

Monitoring namespace:

```text
monitoring
```

### Prometheus

Prometheus uses local storage on:

```text
worker1
```

PersistentVolume:

```text
prometheus-local-worker1
```

Storage size:

```text
30Gi
```

Storage configuration:

```text
monitoring/prometheus-local-storage.yaml
```

Current retention:

```text
15d
```

Retention size:

```text
25GB
```

### Loki

Loki runs in SingleBinary mode.

Loki uses local storage on:

```text
worker2
```

PersistentVolume:

```text
loki-local-worker2
```

Storage size:

```text
30Gi
```

Storage configuration:

```text
monitoring/loki-local-storage.yaml
```

Current log retention:

```text
168h
```

Loki is available internally through:

```text
loki-gateway.monitoring.svc.cluster.local
```

### Grafana Alloy

Grafana Alloy runs as a DaemonSet.

It collects Kubernetes pod logs from:

```text
/var/log/pods/*/*/*.log
```

and forwards them to Loki.

Alloy also runs on the control-plane node because it has a toleration for the control-plane taint.

Configuration:

```text
monitoring/alloy-values.yaml
```

---

## Grafana Dashboard

A custom Grafana dashboard is stored in:

```text
monitoring/dashboards/cluster-overview.json
```

The dashboard contains:

- Nodes Ready
- Running Pods
- Pod Restarts
- Cluster CPU
- Cluster Memory
- PVC Usage
- CPU Usage by Node
- Memory Usage by Node
- SVK Frontend Replicas
- SVK Backend Replicas
- Application CPU
- Application Memory
- Recent Application Errors

The dashboard can be installed or updated with:

```bash
ansible-playbook \
  -i inventory/hosts.yml \
  playbooks/grafana-dashboards.yml \
  --ask-vault-pass
```

---

## Storage

Application and database persistent storage uses the external NFS server.

NFS server:

```text
10.10.147.133
```

NFS export:

```text
/srv/nfs/k8s
```

Main StorageClass:

```text
nfs-rwx
```

The application upload PVC currently uses:

```text
50Gi RWX
```

Monitoring storage is separate:

```text
Prometheus -> local disk on worker1
Loki       -> local disk on worker2
```

---

## Databases

The cluster contains:

- MariaDB
- MongoDB

Persistent database data is stored using Kubernetes PersistentVolumes backed by NFS.

MariaDB currently uses multiple PVCs.

MongoDB currently uses multiple PVCs and runs as a replica set.

Database data should be verified before maintenance or physical server relocation.

---

## Ansible

The infrastructure repository contains the Ansible inventory and maintenance playbooks.

Inventory:

```text
inventory/hosts.yml
```

Global variables:

```text
inventory/group_vars/all/vars.yml
```

Encrypted variables:

```text
inventory/group_vars/all/vault.yml
```

The Vault password is provided separately and is not stored in the repository.

---

## Ansible Vault

Sensitive values are stored in:

```text
inventory/group_vars/all/vault.yml
```

The file is encrypted using Ansible Vault.

To inspect it:

```bash
ansible-vault view \
  inventory/group_vars/all/vault.yml \
  --ask-vault-pass
```

To change the Vault password:

```bash
ansible-vault rekey \
  inventory/group_vars/all/vault.yml
```

The Vault password should be transferred separately from the repository.

Do not commit a plaintext `.vault_pass` file.

---

## Ansible Connectivity Test

Basic test:

```bash
ansible all \
  -i inventory/hosts.yml \
  -m ping
```

Privileged test:

```bash
ansible all \
  -i inventory/hosts.yml \
  -b \
  -m command \
  -a "whoami" \
  --ask-vault-pass
```

Expected privileged result:

```text
root
```

---

## Available Playbooks

Current maintenance and configuration playbooks:

```text
playbooks/
├── audit.yml
├── dns-configuration.yml
├── grafana-dashboards.yml
├── ingress-nginx.yml
├── master-maintenance.yml
├── nfs-maintenance.yml
├── nodelocaldns-configuration.yml
├── os-update.yml
├── worker1-maintenance.yml
└── worker2-maintenance.yml
```

---

## Audit Playbook

General infrastructure audit:

```bash
ansible-playbook \
  -i inventory/hosts.yml \
  playbooks/audit.yml \
  --ask-vault-pass
```

The audit checks:

- package updates
- held packages
- root filesystem
- OS information
- kernel
- RAM
- NFS RAID status

---

## OS Update Preflight

To check available system updates without applying them:

```bash
ansible-playbook \
  -i inventory/hosts.yml \
  playbooks/os-update.yml \
  --ask-vault-pass
```

This playbook performs checks only.

---

## Worker Maintenance

Worker maintenance playbooks:

```text
playbooks/worker1-maintenance.yml
playbooks/worker2-maintenance.yml
```

The playbooks require the worker to be cordoned before an OS upgrade.

Example:

```bash
kubectl cordon worker1
```

Then:

```bash
ansible-playbook \
  -i inventory/hosts.yml \
  playbooks/worker1-maintenance.yml \
  --ask-vault-pass
```

OS upgrades are disabled by default.

They must be explicitly confirmed through the playbook variables.

After maintenance:

```bash
kubectl uncordon worker1
```

The same procedure applies to `worker2`.

---

## Master Maintenance

Master maintenance is handled by:

```text
playbooks/master-maintenance.yml
```

The playbook checks:

- Debian version
- master cordon status
- kubelet
- containerd
- etcd
- etcd health
- etcd snapshots
- IP forwarding
- gateway NAT
- persistent firewall configuration
- dpkg consistency
- disk space
- APT upgrade simulation

The master should remain cordoned during maintenance.

The playbook checks for an etcd snapshot in:

```text
/var/backups/kubernetes
```

At least one valid etcd snapshot should exist before maintenance.

---

## NFS Maintenance

NFS maintenance is handled by:

```text
playbooks/nfs-maintenance.yml
```

The playbook checks:

- Ubuntu version
- RAID status
- `/dev/md0`
- filesystem type
- NFS service
- NFS exports
- package consistency
- disk space
- upgrade simulation

The expected Kubernetes NFS export is:

```text
/srv/nfs/k8s
```

---

## DNS

Cluster DNS uses CoreDNS and NodeLocal DNS.

Custom DNS configuration exists because of the cluster network setup.

NodeLocal DNS external upstreams:

```text
8.8.8.8
8.8.4.4
```

The configuration can be reconciled with:

```bash
ansible-playbook \
  -i inventory/hosts.yml \
  playbooks/nodelocaldns-configuration.yml \
  --ask-vault-pass
```

Worker-only DNS scheduling can be reconciled with:

```bash
ansible-playbook \
  -i inventory/hosts.yml \
  playbooks/dns-configuration.yml \
  --ask-vault-pass
```

Do not modify the DNS configuration without first checking the current network setup.

---

## Kubespray

The Kubernetes cluster was installed and upgraded using Kubespray.

Kubespray is stored separately from this infrastructure repository.

Location on the master:

```text
/home/kubernetes1/Downloads/kubespray
```

Python virtual environment:

```text
/home/kubernetes1/Downloads/kubespray-venv
```

Current Kubespray state:

```text
Base version:    v2.32.0
Local branch:    local-v2.32.0
Current commit:  758bbb6ca
Base tag commit: 9751ea6f6
```

Local patch:

```text
fix resolvconf None paths with Ansible 2.19
```

Runtime environment:

```text
Ansible Core: 2.19.13
Python:       3.11.2
```

To activate the Kubespray environment:

```bash
source ~/Downloads/kubespray-venv/bin/activate
cd ~/Downloads/kubespray
```

The local Kubespray patch should be preserved.

Do not replace the current Kubespray checkout with a fresh upstream version without first preserving:

```text
branch: local-v2.32.0
commit: 758bbb6ca
```

The currently installed Kubernetes version is:

```text
v1.36.4
```

---

## Repository Structure

```text
k8s-infrastructure/
├── README.md
├── inventory/
│   ├── group_vars/
│   │   └── all/
│   │       ├── vars.yml
│   │       └── vault.yml
│   └── hosts.yml
├── monitoring/
│   ├── alloy-values.yaml
│   ├── dashboards/
│   │   └── cluster-overview.json
│   ├── loki-local-storage.yaml
│   ├── loki-values.yaml
│   ├── prometheus-local-storage.yaml
│   └── values.yaml
├── platform/
│   └── ingress-nginx/
│       ├── apps/
│       │   ├── grafana-ingress.yaml
│       │   ├── svk-be-ingress.yaml
│       │   └── svk-fe-ingress.yaml
│       ├── ingress-nginx.yaml
│       └── network-policies/
│           └── allow-ingress-to-svk-be.yaml
└── playbooks/
    ├── audit.yml
    ├── dns-configuration.yml
    ├── grafana-dashboards.yml
    ├── ingress-nginx.yml
    ├── master-maintenance.yml
    ├── nfs-maintenance.yml
    ├── nodelocaldns-configuration.yml
    ├── os-update.yml
    ├── worker1-maintenance.yml
    └── worker2-maintenance.yml
```

---

## Secrets

Sensitive information must not be committed in plaintext.

Ansible passwords are stored in:

```text
inventory/group_vars/all/vault.yml
```

The file is encrypted.

Kubernetes Secrets should also not be stored in plaintext in the repository.

Passwords, tokens and other credentials should be transferred separately.

---

## Server Relocation

Before physical relocation:

1. verify node health
2. verify pod health
3. verify persistent volumes
4. verify database health
5. create a recent etcd snapshot
6. perform a controlled shutdown

Recommended shutdown order:

```text
1. worker1
2. worker2
3. master
4. NFS server
```

Recommended startup order:

```text
1. NFS server
2. master
3. worker1
4. worker2
```

After startup, verify:

```bash
kubectl get nodes -o wide
```

and:

```bash
kubectl get pods -A
```

All nodes should return to:

```text
Ready
```

and all expected workloads should return to:

```text
Running
```

---

## Network Changes After Relocation

The most important risk after relocation is network configuration.

The current internal addressing is:

```text
master   10.10.147.130
worker1  10.10.147.131
worker2  10.10.147.132
nfs      10.10.147.133
```

If possible, preserve these internal addresses.

Changing the internal network may require changes to:

- host network configuration
- Kubernetes node networking
- NFS access
- DNS
- gateway/NAT configuration
- ingress routing
- firewall configuration

The master currently also uses the interface:

```text
eno1
```

for gateway NAT configuration.

This should be checked if the physical networking changes.

---

## Remaining Tasks

The following tasks are intentionally left for the final deployment location or future administrator:

- configure final WAN/public IP
- configure final DNS records
- configure public TCP 80/443 forwarding
- configure HTTPS/TLS certificates with cert-manager
- verify certificate renewal
- verify ingress after DNS migration
- keep Grafana internal-only
- define production backup policy
- define database backup policy
- define NFS backup policy
- configure operational alerts if required
- optionally add more system-level Loki dashboards and alerts

---

## Final Health Check

Useful commands after maintenance or relocation:

```bash
kubectl get nodes -o wide
```

```bash
kubectl get pods -A
```

```bash
kubectl get ingress -A
```

```bash
kubectl get pvc -A
```

```bash
kubectl -n ingress-nginx get pods,svc
```

```bash
kubectl -n monitoring get pods
```

Check for non-running pods:

```bash
kubectl get pods -A | grep -v -E 'Running|Completed' || true
```

Check application resources:

```bash
kubectl -n app get deploy,pods,svc,ingress
```

---