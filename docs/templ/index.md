# Templates

Jinja2 templates rendered by the roles into configuration on the target nodes.
Variables resolve from `group_vars/`, `host_vars/`, and role defaults in
Ansible's usual precedence order.

| Template | Renders to | Role |
|----------|------------|------|
| [`slurm.conf.j2`](slurm_conf_j2.md) | `/etc/slurm/slurm.conf` | `slurm_compute` |
| `task_prolog.sh.j2` | `/etc/slurm/task_prolog.sh` | `slurm_compute` |
