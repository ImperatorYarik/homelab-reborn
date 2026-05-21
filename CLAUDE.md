# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

This is a homelab infrastructure-as-code repository. The local development environment is managed by Vagrant (two Ubuntu 24.04 VirtualBox VMs), and the cluster is provisioned via Ansible. The target stack is a k3s Kubernetes cluster with ArgoCD for GitOps.

## Development Environment

**Requirements:** Vagrant 2.4.9, VirtualBox 7.2

All Vagrant commands run from `dev/`:

```bash
vagrant up                          # Start both VMs
vagrant ssh k3s-master              # SSH into master (or k3s-worker-1)
vagrant halt                        # Stop VMs
vagrant destroy                     # Delete VMs
```

VMs use a private network:
- `k3s-master` — `192.168.56.101` (4 GB RAM, 2 vCPU)
- `k3s-worker-1` — `192.168.56.102` (4 GB RAM, 2 vCPU)

After `vagrant up`, manually add your SSH public key to each VM's `~/.ssh/authorized_keys` so Ansible can connect as `vagrant` without a password.

## Ansible

All Ansible commands run from `Ansible/`. The `ansible.cfg` there sets the inventory and roles path, so no extra flags are needed.

```bash
# Verify connectivity
ansible all -m ping

# Provision the full cluster
ansible-playbook playbooks/site.yaml

# Target a single host or group
ansible-playbook playbooks/site.yaml --limit k3s-master
ansible-playbook playbooks/site.yaml --limit control-plane

# Tear down the cluster (removes k3s, token, and local kubeconfig)
ansible-playbook playbooks/reset.yaml

# Dry run
ansible-playbook playbooks/site.yaml --check
```

## Architecture

### Playbook flow (`playbooks/site.yaml`)

1. **common** role — runs on all nodes: updates apt cache and installs base packages (git, curl, wget, htop, net-tools).
2. **k3s-master-role** — runs on `control-plane` group: installs k3s server (traefik disabled), writes node token to `/tmp/k3s-token` on the control machine, and fetches kubeconfig to `~/.kube/config`.
3. **k3s-worker-role** — runs on `workers` group: reads the token from `/tmp/k3s-token` and joins the worker to the cluster via `K3S_URL`.
4. **argocd** role — runs on `control-plane`: not yet implemented.

### Inventory groups

| Group | Host | IP |
|---|---|---|
| `control-plane` | `k3s-master` | `192.168.56.101` |
| `workers` | `k3s-worker` | `192.168.56.102` |

### Key design details

- k3s is installed without Traefik (`--disable traefik`) — a different ingress controller is expected.
- The node join token flows through the control machine's filesystem (`/tmp/k3s-token`) rather than Ansible variables to keep it out of memory/logs.
- `k3s-master-role` uses `creates: /usr/local/bin/k3s` to make the install idempotent.
- The reset playbook uninstalls k3s via the official uninstall scripts and also cleans up the local kubeconfig and token file.
