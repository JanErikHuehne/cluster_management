# Slurm Config (`templates/slurm_conf.j2`)

Variables come from `group_vars/`, `host_vars/`, and role defaults, in
Ansible's usual precedence order.


This renders `/etc/slurm/slurm.conf`. This file must be **byte-identical on every
node in the cluster** — controller, compute, and login alike. Slurm compares
it at registration and rejects nodes that disagree. That is why it is
generated from a single template and pushed everywhere, rather than edited
per host.

Changing it requires `scontrol reconfigure`; changes to node topology or
partitions require a full `slurmctld` restart.

### Cluster identity

```jinja
ClusterName={{ slurm_cluster_name }}
SlurmctldHost={{ slurm_controller_host }}
```

`ClusterName` is also the key under which accounting records are filed in
slurmdbd. Renaming a live cluster orphans its historical accounting data.

### Authentication

```jinja
AuthType=auth/munge
CredType=cred/munge
```

Both rely on a shared munge key, identical across all nodes and deployed by
the `munge` role from the vaulted copy in the repository. A node whose key
differs cannot authenticate to `slurmctld` at all.

### Users and paths

```jinja
SlurmUser={{ slurm_user }}
SlurmdUser=root
StateSaveLocation={{ slurm_state_save_location }}
SlurmdSpoolDir={{ slurm_spool_dir }}
```

`slurmctld` runs as an unprivileged `SlurmUser`; `slurmd` must run as root to
set up cgroups and switch to the job's user.

`StateSaveLocation` holds the controller's job and node state. It must be on
persistent local storage owned by `SlurmUser` — losing it loses the queue.

### Ports

```jinja
SlurmctldPort=6817
SlurmdPort=6818
SrunPortRange=60000-60999
```

`SrunPortRange` constrains the ports `srun` opens for job I/O, so the firewall
does not need an unbounded range. All three must be open between all nodes.

### Process tracking and task layout

```jinja
ProctrackType=proctrack/cgroup
TaskPlugin=task/affinity,task/cgroup
TaskPluginParam=Threads
```

`proctrack/cgroup` guarantees that every process a job spawns is cleaned up at
job end, including anything that tried to escape its parent.

`task/cgroup` is what makes `MemSpecLimit` and `CoreSpecCount` actually
constrain anything. Without it they are scheduling hints only. It requires a
matching `cgroup.conf` with `ConstrainRAMSpace=yes` and `ConstrainCores=yes`.

`TaskPluginParam=Threads` binds tasks at hardware-thread granularity, matching
`ThreadsPerCore=2` on the SMT-enabled nodes.

### Scheduling

```jinja
SelectType=select/cons_tres
SelectTypeParameters=CR_CPU_Memory
SchedulerType=sched/backfill
```

`cons_tres` allocates individual CPUs and memory rather than whole nodes, so
several jobs can share a node. `CR_CPU_Memory` makes memory a consumable
resource. A job that does not request memory gets the partition default, and
a node with free CPUs but no free memory will not take more work.

### Node behaviour

```jinja
ReturnToService=2
SlurmdDebug=info
```

`ReturnToService=2` returns a DOWN node to service whenever `slurmd` restarts,
regardless of why it went down.

!!! warning
    This masks `Low RealMemory` drains: a node whose detected memory has
    fallen below its configured `RealMemory` will silently rejoin on every
    restart, and the mismatch is never noticed. `ReturnToService=1` — which
    returns a node only if it went down because it was unresponsive — makes
    such configuration errors visible. See
    [Host Vars](../host_vars/index.md) for how to size `RealMemory`.

### Limits

```jinja
MaxArraySize=50000
MaxJobCount=100000
```

`MaxJobCount` bounds the jobs `slurmctld` will hold at once; each consumes
controller memory. `MaxArraySize` caps the largest index in a job array.

### Prolog

```jinja
TaskProlog=/etc/slurm/task_prolog.sh
```

Runs as the job user before each task. Deployed from
`roles/slurm_compute/templates/task_prolog.sh.j2`.

A non-zero exit **fails the job**, so keep the script defensive. Output on
stdout of the form `export NAME=value` is injected into the job environment;
anything else printed there is an error.

### Accounting

```jinja
AccountingStorageType=accounting_storage/slurmdbd
AccountingStorageHost={{ slurm_controller_host }}
AccountingStoragePort=6819
```

Requires `slurmdbd` on the controller and a reachable MariaDB behind it. If
slurmdbd is unavailable, `slurmctld` caches records and replays them on
reconnect, so brief outages do not lose accounting data.

### Node definitions

```jinja
{% raw %}{% for host in groups['slurm_compute'] %}
NodeName={{ host }} CPUs={{ hostvars[host].slurm_cpus }} ... State=UNKNOWN
{% endfor %}{% endraw %}
```

One line per host in the `slurm_compute` inventory group, built from that
host's variables. `slurm_core_spec_count` and `slurm_mem_spec_limit` are
emitted only where defined, so compute nodes without reservations get a
shorter line.

`State=UNKNOWN` lets the node's real state be determined at registration
rather than asserted by the config.

!!! note
    The loop iterates over the inventory, so **every node must be in
    `groups['slurm_compute']` to appear in `slurm.conf` at all** — including
    controller nodes that also run jobs. A node absent from this loop is
    unknown to Slurm even with `slurmd` running.

Because the loop reads `hostvars`, rendering requires facts for every host in
the group. Running the playbook with `--limit` against a subset will produce
an **incomplete** `slurm.conf` unless the other hosts are reachable for fact
gathering. Use `--limit` with care on this role.

### Partitions

```jinja
PartitionName=compute Nodes=ALL Default=YES MaxTime=INFINITE State=UP
```

A single partition containing every defined node. `MaxTime=INFINITE` places no
wall-clock limit on jobs — practical for a small cluster, but it means a
runaway job holds its allocation until killed by hand.

## Verifying a rendered config

Check the output before restarting anything:

```bash
ansible-playbook site.yml --check --diff --tags slurm_conf
```

On the node, after deployment:

```bash
scontrol show config | head -40
scontrol show node tuwzc1n-thalamus
```

The node's reported `CPUTot` and `RealMemory` should match the host vars. A
mismatch is what produces `Low RealMemory` or `Invalid argument` at
registration.