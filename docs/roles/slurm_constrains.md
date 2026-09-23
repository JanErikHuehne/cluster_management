# Slurm constraints (manual)

This page describes how resource limits are set up on the Slurm cluster `compneurocirc`, how users get into Slurm automatically when they first log in, and how to give individual users more resources.

## How limits work in Slurm

Slurm applies limits through three objects in the accounting database (slurmdbd):

- A **QOS** (quality of service) is a named set of limits, for example the maximum number of CPUs per user, the maximum number of jobs per user and the maximum wall time.
- An **account** is a group of users. Each account has a list of QOS its users may use and a default QOS.
- An **association** links one user to one account. A user without an association cannot run jobs once enforcement is enabled.

Each association has two QOS fields:
 
| Field | Meaning |
|---|---|
| `QOS` | The QOS the user is allowed to use. A job requesting any other QOS with `--qos` is rejected. |
| `Def QOS` | The QOS a job gets when the user does not pass `--qos`. For most users this is the QOS that applies in practice. |

## Current setup
 
### QOS
 
| QOS | Purpose |
|---|---|
| `standard` | Limits for normal users. |
| `unlimited` | No limits. |

## Current setup
 
### QOS
 
| QOS | Purpose |
|---|---|
| `standard` | Limits for normal users. |
| `unlimited` | No limits. |


To see the limits currently set:
 
```bash
sacctmgr show qos format=Name%15,MaxTRESPU%30,MaxJobsPU,MaxSubmitPU,MaxWall
```

To change a limit, for example:
 
```bash
sudo sacctmgr -i modify qos standard set MaxTRESPerUser=cpu=64,gres/gpu=2 MaxJobsPerUser=20 MaxWall=2-00:00:00
```
The QOS is not set on the partitions in `slurm.conf`. A partition QOS would apply to every job in that partition and would also cap users in `unlimited`.


### Accounts
 
| Account | Allowed QOS | Default QOS | Who is in it |
|---|---|---|---|
| `labmember` | `standard` | `standard` | Every user, added automatically at first login |
| `unlimited` | `standard`, `unlimited` | `unlimited` | Users moved here by hand |
| `admin` | `standard`, `unlimited` | `unlimited` | Cluster admins, moved here by hand |


The account name gives no special rights. Slurm administrator rights come from the user setting `AdminLevel`, not from membership in `admin`.

The accounts were created with:
 
```bash
sudo sacctmgr -i add qos standard
sudo sacctmgr -i add qos unlimited
sudo sacctmgr -i add account labmember,unlimited,admin
sudo sacctmgr -i modify account labmember set QOS=standard DefaultQOS=standard
sudo sacctmgr -i modify account unlimited set QOS=standard,unlimited DefaultQOS=unlimited
sudo sacctmgr -i modify account admin     set QOS=standard,unlimited DefaultQOS=unlimited
```

Users added to an account inherit its QOS settings, so no per-user QOS configuration is needed.
 
To see all associations:
 
```bash
sacctmgr show assoc format=Account,User,QOS%30,DefaultQOS
```


## Automatic user creation at login
 
Users log in with LDAP credentials through SSSD. Only members of the groups in `simple_allow_groups` in `sssd.conf` can log in. Slurm cannot read LDAP groups, so users are added to Slurm by a PAM hook that runs when an SSH session opens. Every user who can log in and has no association yet is added to `labmember`. Users who already have an association, for example those moved to `unlimited` by hand, are not changed.

### Files
 
| File | Purpose |
|---|---|
| `/usr/local/sbin/slurm-pam-assoc.sh` | The hook script (owner root, mode 700) |
| `/etc/pam.d/sshd` | Calls the hook on SSH login |
| `/var/lib/slurm-pam-assoc/` | One empty file per user already handled on this node |

The last line of `/etc/pam.d/sshd` is:
 
```
session    optional     pam_exec.so quiet /usr/local/sbin/slurm-pam-assoc.sh
```

`optional` means a failure in the script never blocks a login. PAM reads this file on every new connection, so no service restart is needed after changing it.

### What the script does
 
1. Exits unless a session is opening, so logouts do nothing.
2. Skips root and system accounts (UID below 1000).
3. Exits if the user has a cache file in `/var/lib/slurm-pam-assoc/`, meaning this node has already handled them.
4. Asks slurmdbd whether the user has any association. If they do, it leaves them alone.
5. If not, adds them to `labmember` with `sacctmgr`. Both `sacctmgr` calls have a timeout, so a slow slurmdbd cannot hang the login.
6. On success, creates the cache file. On failure it logs the error and creates no cache file, so it retries at the next login.
The script always exits with 0.


```bash
#!/bin/bash
# slurm-pam-assoc.sh
#
# PAM session hook: puts every new user into the Slurm account "labmember"
# at their first login. Users who already have a Slurm association are left alone.
# Always exits 0, so a Slurm problem never blocks a login.
 
CLUSTER=compneurocirc
# PAM starts this script with an almost empty environment, so set PATH here.
# The first entry must be the directory printed by `which sacctmgr`.
export PATH=<slurm-bin-dir>:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
# If slurm.conf is not in the default location, uncomment and set:
# export SLURM_CONF=/etc/slurm/slurm.conf
ACCOUNT=labmember
CACHE_DIR=/var/lib/slurm-pam-assoc
MIN_UID=1000
 
log() { logger -t slurm-pam-assoc "$*"; }
 
[ "$PAM_TYPE" = "open_session" ] || exit 0
USER_NAME="$PAM_USER"
[ -n "$USER_NAME" ] || exit 0
 
uid=$(id -u "$USER_NAME" 2>/dev/null) || exit 0
[ "$uid" -ge "$MIN_UID" ] || exit 0
 
mkdir -p "$CACHE_DIR"
[ -e "$CACHE_DIR/$USER_NAME" ] && exit 0
 
existing=$(timeout 10 sacctmgr -nP show assoc where user="$USER_NAME" \
           cluster="$CLUSTER" format=user 2>/dev/null)
 
if [ -z "$existing" ]; then
  if out=$(timeout 15 sacctmgr -i add user "$USER_NAME" account="$ACCOUNT" cluster="$CLUSTER" \
       DefaultAccount="$ACCOUNT" 2>&1); then
    log "added $USER_NAME to account $ACCOUNT"
  else
    log "FAILED to add $USER_NAME to account $ACCOUNT (exit $?): $(echo "$out" | tr '\n' ' ')"
    exit 0
  fi
fi
 
touch "$CACHE_DIR/$USER_NAME"
exit 0
```

### Installing on a login node
 
1. Copy the script to `/usr/local/sbin/slurm-pam-assoc.sh`.
2. Set the permissions:
```bash
    sudo chown root:root /usr/local/sbin/slurm-pam-assoc.sh
    sudo chmod 700 /usr/local/sbin/slurm-pam-assoc.sh
```

4. Test it by hand on a user without an association:
```bash
    sudo env PAM_TYPE=open_session PAM_USER=<user> /usr/local/sbin/slurm-pam-assoc.sh
    sudo journalctl -t slurm-pam-assoc -n 5
```

5. Keep a root shell open, then add the PAM line:
```bash
    echo 'session    optional     pam_exec.so quiet /usr/local/sbin/slurm-pam-assoc.sh' | sudo tee -a /etc/pam.d/sshd
```
6. Log in fresh over SSH as a user without an association and check the log again.

Repeat this on every login node.

### Give temporary access
 
Slurm has no expiry for QOS access. Schedule the removal with `at` when granting it:
 
```bash
sudo sacctmgr -i modify user <user> set QOS+=unlimited
echo "sacctmgr -i modify user <user> set QOS-=unlimited" | sudo at now + 7 days
```
 
`sudo atq` lists scheduled removals and `sudo atrm <number>` cancels one.
 
### Change the QOS of a pending job
 
The QOS must be in the user's allowed list first:
 
```bash
sudo sacctmgr -i modify user <user> set QOS+=unlimited
sudo scontrol update jobid=<jobid> qos=unlimited
```
 
This only works while the job is pending.
