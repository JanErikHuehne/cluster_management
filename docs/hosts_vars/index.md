# Host Vars (`host_vars/`)

Per-node variables consumed by the `slurm_compute`, `slurm_controller`, and
`ldap_client` roles. One file per node, named after the inventory hostname 
e.g. `host_vars/tuwzc1n-brainstem.yml`.

Anything that is identical across nodes belongs in `group_vars/` instead.
Only genuinely host-specific facts e.g. hardware topology and the node's own AD
machine account live here.

## Obtaining the values

Run on the node itself. If `slurmd` is installed, it reports the topology
Slurm will detect at runtime:

```bash
slurmd -C
# NodeName=tuwzc1n-brainstem CPUs=96 Boards=1 SocketsPerBoard=2 \
#   CoresPerSocket=24 ThreadsPerCore=2 RealMemory=322010
```

Before the node is joined and provisioned, the same numbers come from:

```bash
lscpu | grep -E '^CPU\(s\)|^Socket|^Core|^Thread'
free -m | awk '/^Mem:/ {print $2}'
```

## Fields

### `slurm_cpus`

Total schedulable CPUs, i.e. `sockets × cores × threads`. Maps to `CPUs=` in
`slurm.conf`. Take it from `slurmd -C` rather than computing it by hand — the
two must agree or the node is rejected at registration.

### `slurm_sockets`

Physical CPU sockets. Maps to `Sockets=`. Together with the two fields below it
defines the topology Slurm uses for core-binding and NUMA-aware placement, so
these are not merely cosmetic.

### `slurm_cores`

Physical cores **per socket**, not the machine total. Maps to
`CoresPerSocket=`. A 2-socket, 48-core machine has `slurm_cores: 24`.

### `slurm_threads`

Hardware threads per core — 2 with SMT/Hyper-Threading enabled, 1 without.
Maps to `ThreadsPerCore=`. If SMT is later disabled in firmware, this and
`slurm_cpus` both change.

### `slurm_real_memory`

Memory in **MB** available for job allocation. Maps to `RealMemory=`.

Set this slightly **below** the value `slurmd -C` reports. Slurm compares
detected memory against the configured value and drains the node with reason
`Low RealMemory` if it finds less. It never complains about finding more.
Kernel upgrades, hugepages changes, and DIMM replacements all shift the
detected figure by hundreds of MB, so a few GB of margin prevents nodes
draining for no operational reason.

!!! warning
    `slurm_mem_spec_limit` is **not** part of that comparison. Reserving
    memory via spec limits does not protect you from the drain; keeping
    `slurm_real_memory` under the detected value does.

### `slurm_core_spec_count`

Cores withheld from job allocation for system daemons. Maps to
`CoreSpecCount=`. Counted in **cores**, not CPUs e.g. with `slurm_threads: 2`,
a value of `4` removes 8 CPUs. Slurm chooses which cores, taking the
highest-numbered first.

Prefer `CpuSpecList` where the specific CPUs matter. On nodes running a
BeeGFS daemon, naming the reserved CPUs lets you pin the daemon to the same
set (via `AllowedCPUs=` in a systemd slice) and keep it on the NUMA node
closest to the NIC or HBA. The two options are mutually exclusive.

### `slurm_mem_spec_limit`

Memory in MB withheld from job allocation for system daemons. Maps to
`MemSpecLimit=`. Allocatable memory becomes
`slurm_real_memory − slurm_mem_spec_limit`.

With `TaskPlugin=task/cgroup` and `ConstrainRAMSpace=yes`, this also confines
`slurmd` and `slurmstepd` to the reservation. It does **not** confine BeeGFS or
other non-Slurm daemons. Those run outside Slurm's cgroups. What the setting
buys on a storage node is that jobs cannot claim the memory; the OS, page
cache, and filesystem daemon then share it with no arbitration between them.

Size it for the node's role. Metadata daemons depend heavily on the
dentry/inode cache and want considerably more than storage daemons.

!!! note
    Do not both subtract headroom from `slurm_real_memory` *and* set
    `slurm_mem_spec_limit` for the same purpose as this double-reserves and
    makes the effective allocatable figure non-obvious. Use the modest
    `slurm_real_memory` margin only as drain protection.

### `sssd_sasl_authid`

The node's Active Directory machine account (`sAMAccountName`), including the
trailing `$` and excluding the realm. Written into the `[domain/...]` section
of `sssd.conf` as `ldap_sasl_authid`.

**Read this off the node; never derive it.** NetBIOS names are capped at 15
characters, so longer hostnames are truncated. Any hostname over 8 characters is truncated, 
given the 7-character `TUWZC1N-` prefix. Collisions may also cause the 
join tool to append digits.

```bash
sudo klist -ke /etc/krb5.keytab | grep -o '[^ ]*\$@[^ ]*' | sort -u
# TUWZC1N-THALAMU$@ADS.MWN.DE   → strip the realm
```

Quote the value if it passes through a shell or an expanding template; the
`$` is safe unquoted in YAML but not everywhere downstream.

### `slurm_services`

Slurm daemons enabled on this node. Controls which units `site.yml` installs
and starts.

| Value        | Role                                            |
|--------------|-------------------------------------------------|
| `slurmd`     | Compute — runs jobs                             |
| `slurmctld`  | Controller — scheduling and node state          |
| `slurmdbd`   | Accounting — requires a reachable MySQL/MariaDB |

Nodes running `slurmctld` or `slurmdbd` alongside `slurmd` should always set
spec limits: the controller must stay responsive under a full job load.

## Examples

Compute node, no co-located services:

```yaml
# host_vars/tuwzc1n-brainstem.yml
slurm_cpus: 96
slurm_sockets: 2
slurm_cores: 24
slurm_threads: 2
slurm_real_memory: 316000

sssd_sasl_authid: TUWZC1N-BRAINST$
slurm_services: [slurmd]
```

Controller also running jobs and a filesystem daemon:

```yaml
# host_vars/tuwzc1n-brainstem.yml
slurm_cpus: 112
slurm_sockets: 2
slurm_cores: 28
slurm_threads: 2
slurm_real_memory: 257528
slurm_core_spec_count: 4
slurm_mem_spec_limit: 8192

sssd_sasl_authid: TUWZC1N-BRAINST$
slurm_services: [slurmdbd, slurmd, slurmctld]
```

## Adding a node

1. Join the node to AD, then read the machine account from the keytab.
2. Run `slurmd -C` and copy the topology, trimming `RealMemory`.
3. Create `host_vars/<inventory_hostname>.yml`.
4. Add the host to the appropriate groups in `inventory.yml`.
5. Run the playbook, then `scontrol show node <name>` to confirm Slurm's view
   matches the file.