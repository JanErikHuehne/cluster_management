# BeeGFS client role

This role installs the BeeGFS 8.4 client on Debian 13 nodes and mounts the shared file system at '/scratch'. The role expects a running BeeGFS management service (mgmtd) running on a cluster node.

## What the role does
 
The role runs these tasks in order:
 
| Task | Effect on the node |
|---|---|
| BeeGFS repo key | Downloads the ThinkParQ signing key to `/etc/apt/trusted.gpg.d/beegfs.asc`. |
| BeeGFS repo | Writes `/etc/apt/sources.list.d/beegfs.list` for the release in `beegfs_version`. |
| Client, tools and what autobuild needs to compile the module | Installs `beegfs-client`, `beegfs-tools`, `gcc`, `make`, `linux-headers-amd64` and the headers for the running kernel. |
| Read current auth secret, Connection auth secret | Writes `/etc/beegfs/conn.auth` from the vault, but only if it differs from the file on the node. |
| Client settings | Sets `sysMgmtdHost` and `connAuthFile` in `/etc/beegfs/beegfs-client.conf`. |
| Mount point | Sets the mount point in `/etc/beegfs/beegfs-mounts.conf`. |
| beegfs-client enabled and running | Enables and starts the `beegfs-client` service, which mounts `/scratch`. |
 
BeeGFS does not ship the client kernel module as a binary. The `beegfs-client` service compiles it for the running kernel the first time it starts (BeeGFS calls this autobuild), which is why the role installs a compiler and the kernel headers. The module ends up in `/lib/modules/<kernel>/updates/fs/beegfs_autobuild/`.
 
The role edits the configuration files that the package installs instead of replacing them with templates. It changes only the lines listed above and keeps their original spacing. All other settings stay at the package defaults, and a hand-built node shows no difference in a dry run.
 
A change to the secret or to either config file notifies the handler `Restart beegfs-client`. A restart unmounts and remounts `/scratch`, so read [Changing configuration on running nodes](#changing-configuration-on-running-nodes) before rolling out a change.

## Files and variables
 
| File | Contents |
|---|---|
| `roles/beegfs_client/tasks/main.yml` | The tasks listed above. |
| `roles/beegfs_client/handlers/main.yml` | The `Restart beegfs-client` handler. |
| `roles/beegfs_client/defaults/main.yml` | Defaults for `beegfs_version`, `beegfs_mountpoint` and `beegfs_mgmtd_host`. |
| `group_vars/all/beegfs.yml` | Name and IP address of the management server, also used by the `/etc/hosts` pre-task. |
| `group_vars/all/vault.yml` | The connection auth secret, encrypted with ansible-vault. |
| `site.yml` | The `/etc/hosts` pre-task and the role entry, tagged `beegfs`. |
| `inventory.yml` | The `beegfs_client` group. |
 
| Variable | Value | Set in | Purpose |
|---|---|---|---|
| `beegfs_version` | `"8.4"` | role defaults | BeeGFS release used in the repo URL. |
| `beegfs_mountpoint` | `/scratch` | role defaults | Where the file system is mounted. |
| `beegfs_mgmtd_host` | `beegfs-mgmtd` | `group_vars/all/beegfs.yml` | Management server name, written to `sysMgmtdHost`. |
| `beegfs_mgmtd_ip` | `10.157.154.8` | `group_vars/all/beegfs.yml` | Address that `beegfs_mgmtd_host` resolves to through `/etc/hosts`. |
| `beegfs_conn_auth_b64` | secret | `group_vars/all/vault.yml` | Content of `conn.auth`, base64 encoded. |

## The connection auth secret
 
Every BeeGFS server and client uses the same shared secret in `/etc/beegfs/conn.auth` (mode `0400`, owner `root`). If a client has a different secret, the management server drops its connection and the mount fails.

The vault holds the file base64 encoded, because the file can contain arbitrary bytes. The role decodes it on the node and rewrites the file only when the content differs, so an unchanged secret never triggers a restart.

To set or replace the vault value,  put the key, in double quotes, into the vault with `ansible-vault edit group_vars/all/vault.yml`:
 
```yaml
beegfs_conn_auth_b64: "<output of the command above>"
```

## Adding a new client
 
Add the host to `beegfs_client` in `inventory.yml` and make sure it is also in one of the Slurm groups. A host whose `ansible_host` is already set elsewhere in the inventory needs no further keys here:
 
```yaml
    beegfs_client:
      hosts:
        tuwzc1n-insula:
        tuwzc1n-amygdala:
        <new host>:
```