# GPU nodes (gpu_slurm role)
 
The role turns a Slurm compute node with NVIDIA GPUs into a GPU node. It installs the NVIDIA driver from NVIDIA's repository for Debian 13 and writes `gres.conf`, so that Slurm can hand out single GPUs to jobs.
 
The role covers only the GPU node itself. The Slurm side (the GPU entries in `slurm.conf`) lives in the shared `slurm.conf` template, because `slurm.conf` has to be identical on the controller and every node.
 

## How the pieces fit together
 
| Piece | Where it is defined | Where it ends up |
|---|---|---|
| NVIDIA driver, `nvidia-persistenced` | `gpu_slurm` role | GPU nodes |
| `gres.conf` | `gpu_slurm` role, `templates/gres.conf.j2` | GPU nodes, `/etc/slurm/gres.conf` |
| `GresTypes=gpu`, `AccountingStorageTRES=gres/gpu` | `slurm.conf` template | all nodes |
| `Gres=gpu:<type>:<count>` on the node's `NodeName` line | `slurm.conf` template | all nodes |
| `gpu` partition | `slurm.conf` template, from the `slurm_gpu` group | all nodes |
| Which nodes are GPU nodes | `slurm_gpu` group in `inventory.yml` | |
| GPU type and count | `group_vars/slurm_gpu.yml`, overridable in host_vars | `gres.conf` and `slurm.conf` |
 
`slurm_gpu_type` and `slurm_gpu_count` fill both `gres.conf` on the node and the `Gres=` entry on the controller. The node and the controller therefore always agree on how many GPUs of which type a node has. If they disagree, Slurm drains the node.
 
Jobs only see the GPUs they requested. Slurm sets `CUDA_VISIBLE_DEVICES` for each job, and `ConstrainDevices=yes` in `cgroup.conf` blocks access to the other GPUs. The test in [Checking a GPU node](#checking-a-gpu-node) shows whether the blocking is active.
 
## What the role does
 
The role runs after `slurm_compute` in `site.yml`, because it writes into `/etc/slurm` and restarts `slurmd`. Its tasks, in order:
 
| Task | Effect on the node |
|---|---|
| NVIDIA repo | Installs NVIDIA's `cuda-keyring` package, which adds the signing key and the apt source for NVIDIA's Debian 13 repository. |
| Pin the driver branch | Installs `nvidia-driver-pinning-595`, so apt never moves to a newer driver branch on its own. |
| Driver | Installs `nvidia-driver-cuda` (the driver without desktop components), `nvidia-kernel-open-dkms` (the open kernel modules, rebuilt by DKMS for every kernel), `nvidia-persistenced`, `linux-headers-amd64` and the headers for the running kernel. |
| Keep nouveau off the GPUs | Writes `/etc/modprobe.d/blacklist-nouveau.conf` and rebuilds the initramfs if the file changed. |
| Reboot into the driver | Reboots the node, but only if the NVIDIA driver has never been loaded on it (`/proc/driver/nvidia/version` is missing). |
| nvidia-persistenced | Enables and starts `nvidia-persistenced`, which keeps the GPUs initialized between jobs. |
| gres.conf | Writes `/etc/slurm/gres.conf` and restarts `slurmd` if it changed (handler `Restart slurmd for GPUs`). |
 
For two A40 the generated `gres.conf` is:
 
```
Name=gpu Type=a40 File=/dev/nvidia[0-1]
```

 
## Files and variables
 
| File | Contents |
|---|---|
| `roles/gpu_slurm/tasks/main.yml` | The tasks listed above. |
| `roles/gpu_slurm/handlers/main.yml` | The `Restart slurmd for GPUs` handler. |
| `roles/gpu_slurm/defaults/main.yml` | `nvidia_driver_branch`. |
| `roles/gpu_slurm/templates/gres.conf.j2` | Template for `/etc/slurm/gres.conf`. |
| `group_vars/slurm_gpu.yml` | GPU type and count shared by all GPU nodes. |
| `inventory.yml` | The `slurm_gpu` group. |
| `site.yml` | The role entry, after `slurm_compute`, tagged `gpu`. |
 
| Variable | Value | Set in | Purpose |
|---|---|---|---|
| `nvidia_driver_branch` | `"595"` | role defaults | Driver branch the node is pinned to. |
| `slurm_gpu_type` | `a40` | `group_vars/slurm_gpu.yml` | GPU type label in `gres.conf` and `slurm.conf`. Users request it with `--gres=gpu:a40:1`. |
| `slurm_gpu_count` | `2` | `group_vars/slurm_gpu.yml` | Number of GPUs per node. |
 
A node with different GPUs sets `slurm_gpu_type` and `slurm_gpu_count` in its own host_vars, which take precedence over `group_vars/slurm_gpu.yml`. The type is a free label; use the lowercase model name.
 
## Adding a GPU node
 
Add the host to `slurm_compute` and `slurm_gpu` in `inventory.yml`, and to `ldap_client` and `beegfs_client` like the other compute nodes. Give it its `ansible_host` address once, anywhere in the inventory.
 
```yaml
    slurm_gpu:
      hosts:
        tuwzc1n-thalamus:
        <new node>:
```
 
Create host_vars for the node with `slurm_cpus`, `slurm_sockets`, `slurm_cores`, `slurm_threads` and `slurm_real_memory`. Every node in `slurm_compute` needs them, or rendering `slurm.conf` fails for all nodes. Before Slurm is installed, read the values with:
 
```bash
lscpu | grep -E '^(CPU\(s\)|Socket|Core|Thread)'; free -m | awk '/Mem:/{print $2}'
```
 
Set `slurm_real_memory` somewhat below the `free -m` total. If the node's GPUs are not two A40, also set `slurm_gpu_type` and `slurm_gpu_count` there.
 
 The node reboots once during the run to load the driver (if a new driver is requested).
 

## Checking a GPU node
 
Run these as a normal user on the login node, `tuwzc1n-amygdala`.
 
```bash
sinfo -N -o '%N %P %T %G'
```
 
The GPU node appears in `compute` and `gpu`, with state `idle` and `gpu:a40:2`.
 
```bash
srun -p gpu --gres=gpu:1 bash -c 'hostname; echo CUDA_VISIBLE_DEVICES=$CUDA_VISIBLE_DEVICES; nvidia-smi -L'
```
 
The expected output on thalamus is the host name, `CUDA_VISIBLE_DEVICES=0` and exactly one line `GPU 0: NVIDIA A40`. `nvidia-smi` ignores `CUDA_VISIBLE_DEVICES`, so it lists only one GPU if the job's cgroup really blocks the other one. If it lists both, device isolation is not active; see [Troubleshooting](#troubleshooting).
 
```bash
srun -p gpu --gres=gpu:2 nvidia-smi -L                              # lists both A40
sacct -u $USER -S today -o JobID,State,AllocTRES%60 | tail -3       # AllocTRES contains gres/gpu=1 or 2
```
 

 
## Changing the driver
 
Changing `nvidia_driver_branch` alone only swaps the pin. The role installs missing packages but does not upgrade installed ones, and it reboots only nodes that have never loaded the driver. A driver change is a maintenance step on drained nodes, because upgrading the packages under running jobs makes every new CUDA process fail with "Driver/library version mismatch" until the node reboots.
 
List the branches NVIDIA offers pinning packages for on Debian 13 (on a GPU node):
 
```bash
apt-cache search nvidia-driver-pinning
```
 
Then, starting with one node:
 
```bash
# on habenula: stop new jobs, then wait until squeue -w <nodes> is empty
sudo scontrol update NodeName=<nodes> State=DRAIN Reason="nvidia driver upgrade"
 
# set nvidia_driver_branch in roles/gpu_slurm/defaults/main.yml, then swap the pin
ansible-playbook -i inventory.yml site.yml -l slurm_gpu --tags gpu -K --ask-vault-pass
 
# upgrade to what the new pin allows and boot into it
ansible -i inventory.yml slurm_gpu -m apt -a 'upgrade=dist update_cache=true' -K --ask-vault-pass
ansible -i inventory.yml slurm_gpu -m reboot -K --ask-vault-pass
 
# check the version, then let jobs back on
ansible -i inventory.yml slurm_gpu -a 'nvidia-smi --query-gpu=driver_version --format=csv,noheader' -K --ask-vault-pass
sudo scontrol update NodeName=<nodes> State=RESUME
```
 
A bugfix release within the same branch follows the same steps without changing `nvidia_driver_branch`. The `upgrade=dist` step upgrades all packages on the GPU nodes, not only the NVIDIA ones.
 
Kernel updates need no special handling. DKMS builds the NVIDIA module for a new kernel when the kernel is installed, using the headers that `linux-headers-amd64` pulls in, and the next reboot loads it.
 
## Troubleshooting
 
| Symptom | Cause | Fix |
|---|---|---|
| `sinfo` or `srun` on a node fails with "Protocol authentication error" | munge on that node uses a different key than habenula. On a fresh node this is usually the random key from the Debian package, still loaded in `munged` even after the correct key file was installed. | Compare `sudo sha256sum /etc/munge/munge.key` with habenula. If the files match, run `sudo systemctl restart munge slurmd`. The munge test below the table must then report `Success (0)`. |
| `srun` fails with "Invalid generic resource (gres) specification" | `GresTypes=gpu` is missing from the `slurm.conf` that `srun` reads or that `slurmctld` has loaded. | `scontrol show config` must list `GresTypes = gpu`. Fix the template, rerun the playbook, restart `slurmctld`. |
| The GPU node is missing from `sinfo` | `slurmctld` runs with a `slurm.conf` that does not contain the node, either because the file on habenula was not updated or because `slurmctld` was not restarted. | `grep <node> /etc/slurm/slurm.conf` on habenula, then `sudo systemctl restart slurmctld`. |
| Node drained with "gres/gpu count reported lower than configured" | `slurmd` registered before it knew about the GPUs. | Check `sudo slurmd -G` on the node, restart `slurmd`, resume the node. |
| A job with `--gres=gpu:1` sees both GPUs in `nvidia-smi -L` | Device isolation is off. | Set `ConstrainDevices=yes` in `cgroup.conf` and restart `slurmd`. |
| `nvidia-smi` reports "Driver/library version mismatch" | The driver packages were upgraded, but the old kernel module is still loaded. | Drain the node and reboot it. |
 
The munge test, run on the node in question:
 
```bash
munge -n | ssh tuwzc1n-habenula unmunge | grep STATUS
```
 
Logs on a GPU node:
 
```bash
sudo journalctl -u slurmd -n 50 --no-pager
sudo journalctl -u nvidia-persistenced -n 20 --no-pager
```
 
## Not covered by the role
 
The A40 does not support MIG, so the role has no MIG configuration. `gres.conf` binds GPUs to no specific CPU cores, so jobs can use a GPU from any core; on thalamus, GPU 0 is closest to CPUs 12-14 and 60-62 and GPU 1 to CPUs 42-44 and 90-92 (`nvidia-smi topo -m`). A node with two different GPU models needs one `gres.conf` line per model, which the template does not produce. Slurm's NVML plugin for automatic GPU detection is not used. The role installs no CUDA toolkit, no container toolkit and no GPU monitoring.