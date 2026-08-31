# `munge` Role

Installs and configures MUNGE, the shared-secret authentication service Slurm
uses for all inter-daemon communication. Every node in the cluster (controller, compute, and login) 
must run `munged` with an **identical key**.

MUNGE is a hard dependency of Slurm: `slurmctld`, `slurmd`, and `slurmdbd` all
set `AuthType=auth/munge`. A node whose key differs cannot authenticate at all,
and its `slurmd` will fail to register with symptoms that look like a network
problem rather than an auth one.

## What the role does

1. Creates the `munge` group and user with fixed, cluster-wide IDs.
2. Installs the MUNGE packages.
3. Enforces ownership and permissions on MUNGE's directories.
4. Deploys the shared key from the repository.
5. Relaxes the runtime directory mode so non-root users can reach the socket.
6. Enables and starts `munged`.

## Variables

Defined in `defaults/main.yml`.

| Variable | Default | Purpose |
|----------|---------|---------|
| `munge_user` | `munge` | Service account owning the daemon and key |
| `munge_group` | `munge` | Primary group for the above |
| `munge_uid` | `64031` | Fixed UID — see below |
| `munge_gid` | `64031` | Fixed GID — see below |
| `munge_key_path` | `/etc/munge/munge.key` | Where the shared key is installed |

### Why the UID and GID are pinned

MUNGE credentials encode the sending UID. If `munge` resolves to different
numeric IDs on different nodes, credential verification produces confusing
mismatches rather than a clean failure. Pinning them removes that class of
problem entirely.

`64031` sits in the range Debian reserves for statically-allocated system
accounts, so it will not collide with dynamically assigned ones. The same
reasoning applies to the `slurm` user — both should be pinned cluster-wide.

!!! warning
    Do not change these values on a running cluster. Files owned by the old
    UID (`/etc/munge/munge.key`, `/var/lib/munge`) will not be re-owned by
    changing the variable alone, and `munged` will refuse to start on a key it
    does not own.

## The shared key

`files/munge.key` is stored **ansible-vault encrypted** in the repository and
decrypted at deploy time. It installs as `0400`, owned by `munge`. `munged`
refuses to start on a key with looser permissions.

### Generating a key

Only needed when bootstrapping a new cluster or rotating:

```bash
dd if=/dev/urandom bs=1 count=1024 > munge.key
ansible-vault encrypt munge.key --output roles/munge/files/munge.key
```

### Rotating

The key cannot be rotated node-by-node — during the rollout, updated and
un-updated nodes cannot authenticate to each other. Drain the cluster, deploy
to all nodes in one run, then restart `munged` and all Slurm daemons
everywhere.

!!! danger
    **The munge key is the cluster's authentication secret.** Anyone holding
    it can forge credentials for any user to any `slurmd`, which means
    arbitrary code execution as any user on any node.

    Vault encryption is only as strong as the vault password, and an
    encrypted file in git is offline-brute-forceable by anyone who obtains the
    repository. Keep the repository private, keep the vault password out of
    git, and rotate the key if either assumption is ever broken.

## The systemd override

```ini
[Service]
RuntimeDirectoryMode=0755
```

By default systemd creates `/run/munge` as `0700`, reachable only by the
`munge` user. Slurm clients (`srun`, `sbatch`, `scontrol`) run as ordinary
users and must reach `/run/munge/munge.socket.2` to obtain credentials, so
the directory needs to be traversable.

The socket itself remains `0777` by MUNGE's own design — obtaining a
credential is not privileged; *verifying* one requires the key.

## Directory permissions

| Path | Mode | Contents |
|------|------|----------|
| `/etc/munge` | `0700` | The shared key |
| `/var/lib/munge` | `0700` | Daemon state (replay cache) |
| `/var/log/munge` | `0700` | `munged.log` |

`munged` validates these at startup and exits if they are group- or
world-writable. A node that silently fails to start MUNGE presents as a Slurm
authentication failure, so check `munged` first when `slurmd` will not
register.

## Verifying

On a single node:

```bash
systemctl status munge
munge -n | unmunge          # should print STATUS: Success (0)
```

Between two nodes — this is the check that matters, since it proves the keys
match:

```bash
munge -n | ssh tuwzc1n-brainstem unmunge
```

A `STATUS: Invalid credential` here means the keys differ. Compare digests
without exposing the key material:

```bash
sudo md5sum /etc/munge/munge.key    # run on each node, compare
```

