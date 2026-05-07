# Vagrant Development environment

---
## Requirements:
- ### Vagrant v 2.4.9 installed
- ### VirtualBox v 7.2 installed (vm provider for Vagrant)

---
## Setup
### 1. Start vms
```bash
vagrant up
```
### 2. Copy your SSH public key into authorized_keys on VMs
```bash
vagrant ssh <vm-name> #k3s-master or k3s-worker-1
echo <ssh-public-key> >> ~/.ssh/authorized_keys
exit #eixt VM shell
ssh vagrant@192.168.56.<host> #Test SSH Connection to vms
```
