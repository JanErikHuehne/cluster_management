# `ldap_client` Role

Joins the node to the Active Directory domain and configures SSSD so AD
accounts resolve as POSIX users. This is what makes a single user identity e.g.
same UID, same GID, same home path valid on every node in the cluster.

Correct identity resolution is a prerequisite for both Slurm and the shared
filesystem. Slurm records the submitting UID in accounting and enforces it on
the execution node; BeeGFS stores numeric ownership on disk. A node resolving
a user to a different UID than its peers produces permission errors on shared
storage and jobs that cannot read their own files.

## What the role does

1. Installs the AD integration stack (`sssd-ad`, `realmd`, `adcli`, Kerberos).
2. Deploys `/etc/krb5.conf`.
3. Joins the domain via `realm join`, if not already joined.
4. Deploys `/etc/sssd/sssd.conf`, overwriting what `realm join` generated.
5. Enables home directory creation on first login.
6. Enables and starts `sssd`.

The join step is skipped when `realm list` already reports the domain, so the
role is safe to re-run.

## Variables

### Defaults (`defaults/main.yml`)

| Variable | Default | Purpose |
|----------|---------|---------|
| `sssd_ad_domain` | `ads.mwn.de` | AD domain, lowercase |
| `sssd_krb5_realm` | `ADS.MWN.DE` | Kerberos realm — **always uppercase** |
| `sssd_user_search_base` | `OU=Users,OU=TU,OU=IAM,DC=ads,DC=mwn,DC=de` | Where to look for users |
| `sssd_group_search_base` | `OU=Groups,OU=TU,OU=IAM,DC=ads,DC=mwn,DC=de` | Where to look for groups |
| `sssd_access_provider` | `deny` | Login authorisation policy |
| `sssd_allow_groups` | `[]` | Groups permitted to log in |

Narrow search bases matter on a large directory: they keep SSSD from walking
the entire tree on every lookup. If a legitimate user fails to resolve, an
overly narrow base is the first thing to check.

### Required, supplied elsewhere

| Variable | Source | Purpose |
|----------|--------|---------|
| `ad_join_user` | `group_vars/all/vault.yml` | AD account authorised to join computers |
| `ad_join_password` | `group_vars/all/vault.yml` | Its password |
| `sssd_sasl_authid` | `host_vars/<node>.yml` | This node's machine account |

The join task sets `no_log: true` so the password never reaches the Ansible
log. See [Host Vars](../host_vars/index.md) for obtaining `sssd_sasl_authid`.


## Key configuration choices

### `ldap_id_mapping = False`

SSSD reads `uidNumber` and `gidNumber` directly from AD rather than
algorithmically deriving IDs from the user's SID.

!!! important
    This is the single most consequential setting in the file. With
    `ldap_id_mapping = True`, IDs are computed from the SID and a per-domain
    range — and any node that computes a different range assigns *different
    UIDs to the same user*. On shared storage that means a user losing access
    to their own files.

    Reading authoritative POSIX attributes from the directory removes that
    risk entirely, but requires those attributes to be populated in AD. They
    are, under the IAM OU structure above.

### `ldap_schema = rfc2307bis`

Matches AD's schema for group membership, where groups list members via
`member` attributes containing full DNs. The older `rfc2307` schema assumes
plain usernames and silently resolves no group memberships against AD.

### `enumerate = False`

SSSD does not download the full user and group lists.

### `ignore_group_members = True`

Group lookups return the group without expanding its member list. `id
<username>` still reports all of a user's groups correctly, which is what
matters for file access and Slurm accounts. What breaks is the reverse
direction: `getent group <groupname>` returns an empty member list.

Substantial performance win on large groups. If something needs to enumerate
a group's membership, it must query AD directly.

### `cache_credentials = True` and `krb5_store_password_if_offline = True`

Successful logins are cached, so users can authenticate while the domain
controllers are unreachable.

### `ad_gpo_access_control = disabled`

AD Group Policy is not consulted for login authorisation. GPOs are authored
for Windows clients and their logon-right semantics translate poorly to Linux;
leaving this enabled commonly denies all access for reasons that are difficult
to diagnose from the Linux side. Access control is handled by
`access_provider` instead.

### `fallback_homedir = /home/%u`
Home directory path when AD supplies none.

## Access control

`access_provider` determines who may log in *after* successful authentication.

| Value | Effect |
|-------|--------|
| `deny` | No one may log in |
| `simple` | Only members of `simple_allow_groups` may log in |
| `ad` | Honour AD's own logon restrictions |

!!! danger
    `simple_allow_groups` is read **only** when `access_provider = simple`.
    The current template hardcodes `deny` and ignores the
    `sssd_access_provider` variable, so the group list has no effect and all
    interactive logins are refused.


    On compute nodes reached only through Slurm this may be intended. On
    login nodes it is not. Set `access_provider = {{ sssd_access_provider }}`
    in the template and `sssd_access_provider: simple` in the login group's
    vars.

Note that identity resolution is unaffected by this setting — `id` and file
ownership work regardless. Only interactive login is gated.

## Kerberos configuration

`krb5.conf.j2` permits AES enctypes only, with `allow_weak_crypto = false`.
RC4 and DES are refused. AD has supported AES since Server 2008; if a join
fails with an enctype error, the machine account's
`msDS-SupportedEncryptionTypes` is the place to look.

`udp_preference_limit = 0` forces TCP for all KDC traffic. Kerberos tickets
carrying large AD PAC structures exceed typical UDP datagram limits, and the
resulting fragmentation causes intermittent, hard-to-reproduce auth failures.
TCP avoids the whole class of problem.

`dns_lookup_kdc = true` locates domain controllers via SRV records rather than
pinning specific hosts, so DC changes need no config update.

## Verifying

After the role runs:

```bash
realm list                        # domain shown, configured: kerberos-member
systemctl status sssd
```


Identity resolution — the check that matters:

```bash
id <ad-username>
getent passwd <ad-username>       # NOT bare `getent passwd`, see enumerate
```

Confirm the UID matches on another node:

```bash
ssh tuwzc1n-thalamus id <ad-username>
```

Differing UIDs between nodes mean `ldap_id_mapping` is not doing what it
should, and shared-filesystem permissions will be broken.

Kerberos:

```bash
sudo klist -ke /etc/krb5.keytab   # machine account principal
kinit <ad-username> && klist      # obtain a user ticket
```

## Troubleshooting

Raise SSSD's verbosity by adding `debug_level = 6` to the `[domain/...]`
section and restarting. Logs land in `/var/log/sssd/`, one file per domain
plus `sssd_nss.log` and `sssd_pam.log`.

The cache masks configuration changes — after editing search bases or ID
settings, clear it:


Identity resolution — the check that matters:

```bash
id <ad-username>
getent passwd <ad-username>       # NOT bare `getent passwd`, see enumerate
```

Confirm the UID matches on another node:

```bash
 ssh tuwzc1n-brainstem id <ad-username>
```

Differing UIDs between nodes mean `ldap_id_mapping` is not doing what it
should, and shared-filesystem permissions will be broken.

Kerberos:

```bash
sudo klist -ke /etc/krb5.keytab   # machine account principal
kinit <ad-username> && klist      # obtain a user ticket
```

## Troubleshooting

Raise SSSD's verbosity by adding `debug_level = 6` to the `[domain/...]`
section and restarting. Logs land in `/var/log/sssd/`, one file per domain
plus `sssd_nss.log` and `sssd_pam.log`.

The cache masks configuration changes, after editing search bases or ID
settings, clear it:

```bash
sudo systemctl stop sssd
sudo rm -f /var/lib/sss/db/*
sudo systemctl start sssd
```

If a node was removed from AD and rejoined, its keytab is stale and every
lookup fails with a credentials error. Leave and rejoin properly:

```bash
sudo realm leave ads.mwn.de
sudo realm join --user=<admin> --computer-name=<NAME> ads.mwn.de
```

Note the machine account name is capped at 15 characters — a rejoin using the
untruncated hostname creates a *second*, different account.
