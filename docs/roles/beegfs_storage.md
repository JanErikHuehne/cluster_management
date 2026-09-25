

Beegfs storage node · MD
# Adding a BeeGFS storage node
 
This page describes how to add a storage server to the BeeGFS file system mounted at `/scratch`. The procedure is manual on purpose. It formats a disk and registers numeric IDs with the management service, and both are hard to undo, so every step ends with a check before the next one starts.
 
The reference is the existing storage server `tuwzc1n-thalamus`. A new node is set up the same way, so that both targets behave identically.
 
## Current layout
 
| Service | ID | Host | Details |
|---|---|---|---|
| Management (mgmtd) | | `tuwzc1n-habenula` (10.157.154.8) | Reached as `beegfs-mgmtd`. Runs without TLS. |
| Metadata | node m:1, target m:1 | see `beegfs node list` | 149.5 GiB |
| Storage | node s:2, target s:201 | `tuwzc1n-thalamus` | 1.3 TiB XFS on `/data/storage`, storage pool s:1 |
 
Update this table when a node is added.
 
All servers run BeeGFS 8.4.1 on Debian 13. The management service has no TLS, so every `beegfs` command needs `--tls-disable`, and it needs `sudo`. The `beegfs` commands on this page run on `tuwzc1n-habenula`.
 
To see the current state at any time:
 
```bash
sudo beegfs health capacity --tls-disable    # all nodes and targets with IDs and free space
sudo beegfs health check --tls-disable       # only the license check is expected to fail
```
 
## Before you start
 
Decide three things and write them down:
 
| Decision | Rule | Next free value |
|---|---|---|
| Numeric node ID | Next unused storage node ID | 5 |
| Numeric target ID | Node ID times 100 plus target number, like 201 on node 2 | 501 |
| Target device | An empty disk or RAID volume on the new node | |
 
Check that the IDs are really unused in `beegfs health capacity`. An ID must never be reused, and it must never change once a target holds data.
 
Some facts about what a new node changes:
 
- It adds space and throughput, not redundancy. New files are striped across all targets, so if either storage node is down, those files cannot be read. Protection against this would need buddy mirroring, which is a licensed BeeGFS feature.
- Existing files stay where they are. Only files written after the new target has joined use it.
- The new target joins the default storage pool (s:1) automatically. No client needs a remount.
Commands on the new node run as root (`sudo -i`).
 
## Step 1: Allow new targets at the management service
 
On habenula, check whether the management service accepts registrations:
 
```bash
grep -n registration-disable /etc/beegfs/beegfs-mgmtd.toml
```
 
If it says `registration-disable = true`, set it to `false` and restart the service. Clients and servers reconnect on their own after the restart.
 
```bash
sudo systemctl restart beegfs-mgmtd
```
 
Remember to set it back in step 7.
 
## Step 2: Install the packages
 
Skip the two `wget` lines if the node is already a BeeGFS client through Ansible; the `beegfs_client` role has set up the repository.
 
```bash
wget https://www.beegfs.io/release/beegfs_8.4/gpg/GPG-KEY-beegfs -O /etc/apt/trusted.gpg.d/beegfs.asc
wget https://www.beegfs.io/release/beegfs_8.4/dists/beegfs-trixie.list -O /etc/apt/sources.list.d/beegfs.list
apt update
apt install beegfs-storage libbeegfs-ib xfsprogs
systemctl stop beegfs-storage
systemctl is-active beegfs-storage    # must not say "active"
```
 
Debian starts a service as soon as its package is installed. The storage service must not run before step 5 has set its IDs, so stop it right away. `libbeegfs-ib` is only needed for InfiniBand; it is installed on thalamus as well and does no harm without InfiniBand hardware.
 
## Step 3: Prepare the target filesystem
 
Identify the device and make sure it is empty. `mkfs` destroys whatever is on the device, and nothing checks for you.
 
```bash
lsblk -o NAME,SIZE,TYPE,FSTYPE,MOUNTPOINT,MODEL
blkid /dev/<device>    # must print nothing
```
 
If `blkid` prints anything, the device holds a filesystem. Stop and find out what is on it.
 
Format it with XFS. The stripe settings must match the RAID layout of the device: `su` is the strip size of the RAID controller and `sw` the number of data disks. Thalamus uses a PERC H755 virtual disk with `su=256k,sw=1`:
 
```bash
mkfs.xfs -d su=256k,sw=1 -l version=2,su=256k /dev/<device>
```
 
For a plain disk without RAID, leave out the `-d` and `-l` options and let `mkfs.xfs` detect the geometry.
 
Mount it at `/data/storage` with the same options as on thalamus:
 
```bash
mkdir -p /data/storage
blkid -s UUID -o value /dev/<device>    # note this UUID, it is needed again in step 5
echo "UUID=<uuid>  /data/storage  xfs  noatime,nodiratime,largeio,inode64,swalloc  0 2" >> /etc/fstab
systemctl daemon-reload
mount /data/storage
findmnt /data/storage    # must show the new device with FSTYPE xfs
```
 
## Step 4: Install the connection auth secret
 
All BeeGFS servers and clients share one secret in `/etc/beegfs/conn.auth`. Copy it from thalamus. On thalamus:
 
```bash
sudo install -m 600 -o $USER /etc/beegfs/conn.auth /tmp/conn.auth
scp /tmp/conn.auth <new node>:/tmp/ && rm /tmp/conn.auth
```
 
On the new node:
 
```bash
install -m 400 -o root -g root /tmp/conn.auth /etc/beegfs/conn.auth && rm /tmp/conn.auth
sha256sum /etc/beegfs/conn.auth    # must start with 01e33e8b, like on thalamus
```
 
## Step 5: Initialize the target
 
Check that the node resolves the management server by name. Nodes in the Slurm groups get the entry from the `/etc/hosts` pre-task in `site.yml`; on any other node, add `10.157.154.8 beegfs-mgmtd` to `/etc/hosts` by hand.
 
```bash
getent hosts beegfs-mgmtd    # must print 10.157.154.8
```
 
Run the setup script with the IDs from "Before you start". It creates the target directory, writes the IDs into it and edits `/etc/beegfs/beegfs-storage.conf`:
 
```bash
/opt/beegfs/sbin/beegfs-setup-storage -p /data/storage/beegfs_storage -s <next_id> -i <next_numeric_target_id> -m beegfs-mgmtd
```
 
Point the service at the auth secret:
 
```bash
sed -i 's#^\(\s*connAuthFile\s*=\).*#\1 /etc/beegfs/conn.auth#' /etc/beegfs/beegfs-storage.conf
```
 
Check the result against thalamus before starting anything:
 
```bash
cat /data/storage/beegfs_storage/nodeNumID /data/storage/beegfs_storage/targetNumID    # 3 and 301
grep -nE '^\s*(sysMgmtdHost|storeStorageDirectory|storeAllowFirstRunInit|storeFsUUID|connAuthFile)\s*=' /etc/beegfs/beegfs-storage.conf
```
 
The config lines must look like this, with the UUID of the new filesystem from step 3. The leading commas are written by the setup script and are also present on thalamus.
 
```
sysMgmtdHost                 = beegfs-mgmtd
storeStorageDirectory        = ,/data/storage/beegfs_storage
storeAllowFirstRunInit       = false
storeFsUUID                  = ,<uuid>
connAuthFile                 = /etc/beegfs/conn.auth
```
 
`storeFsUUID` ties the target to its filesystem, so the service refuses to start if `/data/storage` is not mounted instead of writing into the root filesystem. If the line is missing or holds a different UUID, correct it before step 6.
 
## Step 6: Start the service and verify
 
On the new node:
 
```bash
systemctl enable --now beegfs-storage
journalctl -u beegfs-storage -n 30 --no-pager
```
 
On habenula:
 
```bash
sudo beegfs health capacity --tls-disable    # new target s:301 on node s:3, storage pool s:1
sudo beegfs health check --tls-disable       # only the license check may fail
```
 
On any client, the new server appears in the list of storage nodes within a minute or so:
 
```bash
cat /proc/fs/beegfs/*/storage_nodes
```
 
To see the new target receive data, write a test file of a few GB into `/scratch` from a client, check that `SPACE_USED` of s:301 grows in `beegfs health capacity`, and delete the file again.
 
## Step 7: Close registration again
 
If you changed `registration-disable` in step 1, set it back to `true` in `/etc/beegfs/beegfs-mgmtd.toml` on habenula and restart the service:
 
```bash
sudo systemctl restart beegfs-mgmtd
```
 
This prevents a misconfigured server from registering an unintended target later.
 
## Afterwards
 
Add the node to the table in [Current layout](#current-layout). If the node should also mount `/scratch`, add it to the `beegfs_client` group in `inventory.yml` and run the playbook with `--tags beegfs`.
 
If the node runs a firewall, it must accept TCP and UDP on port 8003 (`connStoragePort`) from the other BeeGFS nodes and the clients. The existing nodes run no firewall.
 
## What can go wrong
 
The storage log (`journalctl -u beegfs-storage` on the node) and the management log (`journalctl -u beegfs-mgmtd` on habenula) name the reason in almost every case.
 
| Symptom | Cause | Fix |
|---|---|---|
| The new target appears with a random or unexpected ID | `beegfs-storage` ran before step 5 and initialized the directory by itself, or the setup ran twice. | Stop the service. While the target holds no data, remove its registration at the management service (`beegfs target --help` and `beegfs node --help` list the delete commands) and empty the target directory, then repeat from step 5. Never do this with a target that holds data. |
| The node does not appear at all, and the management log shows a refused registration | `registration-disable = true` | Step 1. |
| The storage log shows authentication or connection errors towards the management server | `conn.auth` differs from the other nodes. | Compare `sha256sum /etc/beegfs/conn.auth` with thalamus and repeat step 4. |
| The storage log says it cannot reach or resolve the management host | `beegfs-mgmtd` does not resolve on the node. | `getent hosts beegfs-mgmtd`, then fix `/etc/hosts`. |
| The service refuses to start and mentions the filesystem or its UUID | `/data/storage` is not mounted, or `storeFsUUID` does not match the mounted filesystem. | `findmnt /data/storage`, compare `blkid -s UUID -o value /dev/<device>` with `storeFsUUID`. |
| After a reboot the service fails | The fstab line from step 3 is wrong, so `/data/storage` was not mounted. | `mount /data/storage` shows the error; fix the fstab line. |
| Clients do not use the new target | Clients refresh their target list periodically, and only new files use the new target. | Check `/proc/fs/beegfs/*/storage_nodes` on a client after a minute, and test with a new file as in step 6. |
 
A `mkfs` on the wrong device cannot be undone. The `blkid` check in step 3 exists for this reason; do not skip it.