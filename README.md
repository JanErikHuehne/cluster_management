# cluster_management


This repository contains an Ansible build for the CompNeuro compute cluster.

# cluster_management

Ansible configuration for the CompNeuro compute cluster: Slurm with GPU support, BeeGFS clients, AD login and Open OnDemand.

Documentation: <https://janerikhuehne.github.io/cluster_management/>

## Layout

Everything Ansible needs is in `cluster-config/`:

| Path | Contents |
|---|---|
| `site.yml` | The main playbook |
| `inventory.yml` | The nodes and their groups |
| `group_vars/`, `host_vars/` | Settings per group and per node; `group_vars/all/vault.yml` is encrypted with ansible-vault |
| `roles/` | One role per component: munge, Slurm, GPU nodes, BeeGFS, AD login, Open OnDemand |

## Usage

Run from `cluster-config/`:

```bash
ansible-playbook -i inventory.yml site.yml -K --ask-vault-pass
```

`-K` asks for the sudo password on the nodes, and `--ask-vault-pass` for the vault password. To run only part of the setup, add a tag such as `--tags beegfs` or `--tags gpu`. To limit the run to certain nodes, add `-l <host>`.