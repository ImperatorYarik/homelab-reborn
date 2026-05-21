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

All Ansible commands run from `Ansible/`. The `ansible.cfg` there sets the inventory, roles path, and SSH options — no extra flags needed.

```bash
# Verify connectivity
ansible all -m ping

# Provision the full cluster
ansible-playbook playbooks/site.yaml

# Run a specific layer only (tags: common, k3s, k3s-master, k3s-worker, argocd)
ansible-playbook playbooks/site.yaml --tags argocd

# Target a single host or group
ansible-playbook playbooks/site.yaml --limit k3s-master

# Tear down the cluster (removes k3s, token, and local kubeconfig)
ansible-playbook playbooks/reset.yaml

# Dry run
ansible-playbook playbooks/site.yaml --check
```

## Architecture

### Playbook flow (`playbooks/site.yaml`)

1. **common** — all nodes: updates apt cache, installs base packages.
2. **k3s-master-role** — `control_plane` group: downloads and installs the pinned k3s version, saves the join token to `~/.local/share/k3s-token` (mode 600), fetches the kubeconfig and patches the server address from `127.0.0.1` to the actual node IP before writing to `~/.kube/config`.
3. **k3s-worker-role** — `workers` group: reads the join token and installs k3s agent.
4. **argocd** — `control_plane` group: applies the ArgoCD install manifest, exposes `argocd-server` as NodePort, waits for all components, and prints the UI URL and initial admin password.

### Inventory groups

| Group | Host | IP |
|---|---|---|
| `control_plane` | `k3s-master` | `192.168.56.101` |
| `workers` | `k3s-worker` | `192.168.56.102` |

### Key variables (role defaults)

| Variable | Default | Role |
|---|---|---|
| `k3s_version` | `v1.32.4+k3s1` | k3s-master-role, k3s-worker-role |
| `k3s_extra_server_args` | `--disable traefik` | k3s-master-role |
| `k3s_token_path` | `~/.local/share/k3s-token` | k3s-master-role, k3s-worker-role |
| `k3s_kubeconfig_dest` | `~/.kube/config` | k3s-master-role |
| `argocd_version` | `v2.13.3` | argocd |
| `argocd_namespace` | `argocd` | argocd |

Override any variable with `-e` or in `inventory/hosts.yaml`:
```bash
ansible-playbook playbooks/site.yaml --tags argocd -e argocd_version=v2.14.0
```

## CI

Three GitHub Actions jobs run on any push that touches `Ansible/` (`.github/workflows/ansible-ci.yml`). Dependencies are pinned in `.github/requirements/`.

| Job | Tool | What it checks |
|-----|------|----------------|
| `lint` | yamllint + ansible-lint | YAML formatting, Ansible best practices |
| `syntax-check` | ansible-playbook | Both playbooks parse and all roles resolve |
| `molecule` | molecule + Docker | `common` role actually installs packages correctly |

Each job writes a Markdown status table to the GitHub Step Summary; failures append the tool output below the table. yamllint also emits inline PR annotations via `-f github`.

### Linting rules

ansible-lint runs with `profile: basic`. Constraints to keep in mind when writing new tasks or roles:

- All YAML files must start with `---`.
- `galaxy_info` in `meta/main.yaml` requires `author`, `license`, and `namespace` — all set to `ytsalko` / `MIT`.
- Role directory names must match `^[a-z][a-z0-9_]*$` (underscores only, no hyphens). `k3s-master-role` and `k3s-worker-role` are grandfathered via `skip_list` in `.ansible-lint`.
- Variables defined inside a role must be prefixed with the role name (e.g. `argocd_`, `k3s_`).
- `ansible.builtin.shell` is only valid when the command uses shell features (pipes, globs, redirects). Use `ansible.builtin.command` otherwise.
- `ansible.builtin.command` for `kubectl` is intentional (no `kubernetes.core` dependency) — demoted to warning in `.ansible-lint`.
- `.yamllint.yml` must include specific rules required by ansible-lint: `comments-indentation: false`, `braces.max-spaces-inside: 1`, `octal-values.forbid-*: true`.

### Molecule

Tests live in `roles/common/molecule/default/`. The Docker driver requires the `community.docker` Ansible collection, installed automatically via `requirements.yml` in the scenario directory. The test image is `geerlingguy/docker-ubuntu2404-ansible` (pre-built, includes Python and sudo).

To run molecule locally from `Ansible/roles/common/`:
```bash
molecule test          # full sequence
molecule converge      # apply role only
molecule verify        # run assertions only
molecule destroy       # clean up container
```

### Key design details

- k3s is installed without Traefik (`--disable traefik`) — a different ingress controller is expected.
- The install script is downloaded to `/tmp/k3s-install.sh` before execution (not piped directly to shell) and `INSTALL_K3S_VERSION` pins the version.
- The join token is passed to the worker via environment variable (`K3S_TOKEN`) with `no_log: true` — it never appears in Ansible output.
- All `kubectl` tasks in the argocd role set `KUBECONFIG: /etc/rancher/k3s/k3s.yaml` explicitly, since the play runs as root via `become: true` and k3s does not write to `/root/.kube/config`.
- ArgoCD manifests are cached at `/tmp/argocd-<version>-install.yaml` — changing `argocd_version` triggers a fresh download automatically.
- The reset playbook resets workers before the master (workers must leave first), then cleans up the local token and kubeconfig.
