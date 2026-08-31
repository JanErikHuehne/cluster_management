# `roles/ldap_client/templates/sssd.conf.j2`

Renders `/etc/sssd/sssd.conf`, which governs how the node resolves and
authenticates Active Directory accounts. Deployed by the
[`ldap_client`](../roles/ldap_client.md) role **after** the domain join,
deliberately overwriting the minimal file `realm join` generates.

Installed as `0600` owned by root. SSSD refuses to start on a
group- or world-readable config, so a permissions mistake presents as a
service that will not start rather than a warning.

## Service section

```jinja
[sssd]
domains = {{ sssd_ad_domain }}
config_file_version = 2
services = nss, pam
```

`nss` provides identity resolution (`id`, `getent`, file ownership); `pam`
provides authentication. `ssh` and `sudo` responders are not enabled —
`ssh` is unnecessary unless caching AD-published host keys, and `sudo` rules
are managed locally rather than in the directory.

## Providers

```jinja
ad_domain = {{ sssd_ad_domain }}
id_provider = ad
auth_provider = ad
chpass_provider = ad
krb5_realm = {{ sssd_krb5_realm }}
```

The `ad` provider handles the AD-specific behaviour — site discovery, DC
failover, and the AD schema — that the generic `ldap` provider would need
configured by hand.

## Identity mapping

```jinja
ldap_id_mapping = False
ldap_schema = rfc2307bis
ldap_user_search_base = {{ sssd_user_search_base }}
ldap_group_search_base = {{ sssd_group_search_base }}
```

!!! important
    `ldap_id_mapping = False` is the most consequential line in this file.
    UIDs and GIDs are read from the directory's `uidNumber` and `gidNumber`
    attributes, so every node resolves a user to the same numeric ID.

    With `True`, SSSD derives IDs algorithmically from each user's SID. Any
    node computing a different range assigns *different UIDs to the same
    user* — and since the shared filesystem stores numeric ownership on disk,
    that means users losing access to their own files, with no error that
    points at the cause.

    This requires POSIX attributes to be populated in AD. They are, under
    the IAM OU structure in use.

`rfc2307bis` matches AD's group membership model, where groups reference
members by full DN. The older `rfc2307` schema expects bare usernames and
silently resolves no memberships at all — the failure mode is a user with
correct UID but no supplementary groups.

The search bases scope lookups to the relevant OUs instead of walking the
whole directory. A user who cannot be resolved despite existing in AD is
usually outside the configured base.

## Performance

```jinja
enumerate = False
ignore_group_members = True
```

!!! note
    With `enumerate = False`, a bare `getent passwd` shows **only local
    users**, while `getent passwd <username>` resolves AD accounts normally.
    This surprises nearly everyone testing a fresh join. Always query a
    specific name.

    Enumerating a directory this size on every cache refresh would be
    prohibitively expensive.

`ignore_group_members = True` returns groups without expanding membership.
`id <username>` still lists all of a user's groups — the direction that
matters for file access and Slurm — but `getent group <groupname>` returns an
empty member list. Anything needing a group's roster must query AD directly.

## Caching

```jinja
cache_credentials = True
krb5_store_password_if_offline = True
```

Successful authentications are cached so users can log in while the DCs are
unreachable, and password changes made offline are replayed on reconnect.

Worth having on a cluster: a domain outage should not prevent access to
running jobs. The trade-off is that the cache masks configuration changes —
after editing search bases or ID settings, clear `/var/lib/sss/db/` or the old
answers persist.

## Access control

```jinja
access_provider = deny
simple_allow_groups = {{ sssd_allow_groups | join(' ') }}
```

!!! danger
    These two lines contradict each other. `simple_allow_groups` is consulted
    **only** when `access_provider = simple`; under `deny`, every interactive
    login is refused no matter what the group list contains.

    The `sssd_access_provider` variable exists in `defaults/main.yml` but the
    template does not use it. To make the group list effective:

```jinja
    access_provider = {{ sssd_access_provider }}
    {% raw %}{% if sssd_allow_groups %}
    simple_allow_groups = {{ sssd_allow_groups | join(',') }}
    {% endif %}{% endraw %}
```

    Then set `sssd_access_provider: simple` for login nodes and leave compute
    nodes on `deny`.

    Note `simple_allow_groups` expects a **comma**-separated list; the
    current `join(' ')` produces spaces, which parses as a single group name
    containing spaces.

Identity resolution is unaffected by this setting. `id`, file ownership, and
Slurm's UID handling all work under `deny` — only interactive login is gated.

```jinja
ad_gpo_access_control = disabled
ad_gpo_ignore_unreadable = True
```

AD Group Policy is not consulted for login authorisation. GPO logon rights are
authored for Windows semantics and translate poorly; leaving this enabled
commonly denies all access for reasons invisible from the Linux side.

## Home directories and shell

```jinja
fallback_homedir = /home/%u
default_shell = /bin/bash
```

Used when AD supplies no POSIX values of its own.

!!! warning
    If `/home` sits on the shared filesystem, `pam_mkhomedir` creates the
    directory on whichever node the user first reaches. A home directory
    created before BeeGFS mounts is written to the local disk and then hidden
    by the mount — presenting as a user whose files exist on one node and not
    others. Verify mount ordering against PAM.

## Machine account

```jinja
ldap_sasl_authid = {{ sssd_sasl_authid }}
```

The account SSSD uses to bind to AD, via GSSAPI against the host keytab —
which is why no password appears in this file.

Must match the keytab principal exactly, including the trailing `$`. NetBIOS
truncation at 15 characters means it frequently differs from the hostname:
`tuwzc1n-thalamus` joins as `TUWZC1N-THALAMU$`. See
[Host Vars](../host_vars/index.md) for reading the real value.

A mismatch produces bind failures that look like network or DC problems rather
than a naming error.

## Verifying

```bash
sudo sssctl config-check      # syntax and permissions
systemctl status sssd
id <ad-username>
getent passwd <ad-username>
```

Same user on two nodes, which is the check that catches ID-mapping problems:

```bash
id <ad-username>
ssh tuwzc1n-thalamus id <ad-username>
```

After changing this file, restart and — if the change concerns identity —
clear the cache:

```bash
sudo systemctl stop sssd
sudo rm -f /var/lib/sss/db/*
sudo systemctl start sssd
```

Raise `debug_level = 6` in the domain section for detail; logs are in
`/var/log/sssd/`.