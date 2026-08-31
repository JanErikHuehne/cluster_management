# `roles/ldap_client/templates/krb5.conf.j2`

Renders `/etc/krb5.conf`, the system-wide Kerberos configuration. Deployed by
the [`ldap_client`](../roles/ldap_client.md) role before the domain join,
since `realm join` needs working Kerberos to obtain a ticket.

Read by every Kerberos-aware program on the node — SSSD, `kinit`, `ssh` with
GSSAPI — not just SSSD.

## Realm defaults

```jinja
default_realm = {{ sssd_krb5_realm }}
```

The realm assumed when a principal is given without one, so `kinit alice`
works instead of requiring `kinit alice@ADS.MWN.DE`.

!!! important
    Kerberos realms are **case-sensitive and conventionally uppercase**, while
    DNS domains are lowercase. `ADS.MWN.DE` and `ads.mwn.de` are different
    realms to Kerberos. This is why `sssd_krb5_realm` and `sssd_ad_domain`
    are separate variables rather than one derived from the other.

## Service discovery

```jinja
dns_lookup_realm = true
dns_lookup_kdc = true
```

Locates KDCs through DNS SRV records (`_kerberos._tcp.ads.mwn.de`) rather than
a static list. Domain controllers can be added, removed, or replaced without
touching cluster configuration, and clients follow whatever AD advertises.

The explicit `[realms]` block below still names servers as a fallback for when
SRV lookups fail.

```jinja
rdns = false
```

Disables reverse-DNS lookups when canonicalising service principals. Reverse
records in cluster networks are frequently absent or inconsistent, and relying
on them makes authentication depend on PTR records that nobody maintains.
Forward lookups only is both faster and more predictable.

## Encryption

```jinja
allow_weak_crypto = false
default_tkt_enctypes = aes256-cts-hmac-sha1-96 aes128-cts-hmac-sha1-96
default_tgs_enctypes = aes256-cts-hmac-sha1-96 aes128-cts-hmac-sha1-96
permitted_enctypes = aes256-cts-hmac-sha1-96 aes128-cts-hmac-sha1-96
```

AES only. RC4-HMAC and single-DES are refused outright — both are considered
broken, and RC4 in particular enables Kerberoasting attacks against service
accounts.

The three settings cover different exchanges: `tkt` for the initial TGT
request, `tgs` for service tickets, `permitted` for what will be accepted at
all. Setting all three closes the gaps.

!!! note
    AD has supported AES since Server 2008, so this is safe against any
    current domain. If a join fails with `KDC has no support for encryption
    type`, the machine account's `msDS-SupportedEncryptionTypes` attribute is
    the thing to inspect — accounts created by older tooling sometimes carry
    an RC4-only value.

## Ticket lifetimes

```jinja
ticket_lifetime = 24h
renew_lifetime = 7d
forwardable = true
```

A ticket is valid for 24 hours and renewable without re-authenticating for up
to 7 days. The KDC's own policy caps these; AD's defaults are 10 hours and 7
days, and the shorter of the two wins.

`forwardable = true` allows a ticket to be delegated to another host, so a
ticket obtained on a login node can authenticate an `ssh` onward to a compute
node.

!!! warning
    Batch jobs outliving the ticket lifetime is a recurring source of
    confusion. A job that runs longer than 24 hours will find its user's
    ticket expired mid-run, and any Kerberos-dependent operation — including
    access to a Kerberised filesystem — then fails partway through.

    `MaxTime=INFINITE` on the `compute` partition means jobs can easily exceed
    this. If jobs need Kerberos credentials for their whole duration, ticket
    renewal must be arranged explicitly. Note this does **not** affect plain
    identity resolution: `id` and UID-based file ownership keep working with
    no ticket at all.

## Transport

```jinja
udp_preference_limit = 0
```

Forces TCP for all KDC communication.

AD embeds a PAC — the user's full group membership — in every ticket, which
for a user in many groups produces packets well beyond the practical UDP
datagram size. The resulting fragmentation causes authentication failures that
are intermittent, user-specific, and extremely hard to reproduce: the same
credentials work for one account and fail for another purely because of group
count. Forcing TCP eliminates the entire class.

## Realm and mapping blocks

```jinja
[realms]
    {{ sssd_krb5_realm }} = {
        kdc = {{ sssd_ad_domain }}
        admin_server = {{ sssd_ad_domain }}
    }

[domain_realm]
    .{{ sssd_ad_domain }} = {{ sssd_krb5_realm }}
    {{ sssd_ad_domain }} = {{ sssd_krb5_realm }}
```

`[realms]` names the KDC by domain name, which resolves to whichever DC
answers — a fallback for when SRV discovery is unavailable.

`[domain_realm]` maps DNS names onto the realm. Both forms are needed: the
leading-dot entry covers subdomains such as `tuwzc1n-thalamus.ads.mwn.de`,
and the bare entry covers the domain itself. Omitting the first is a common
mistake that breaks host-principal resolution while leaving user
authentication apparently fine.

## Verifying

```bash
kinit <ad-username>
klist -e            # -e shows the enctype; expect aes256-cts-hmac-sha1-96
```

Machine account and its principal:

```bash
sudo klist -ke /etc/krb5.keytab
```

Confirm SRV discovery resolves:

```bash
dig +short SRV _kerberos._tcp.ads.mwn.de
```

`kinit` failing with clock-skew errors means NTP is broken — Kerberos rejects
timestamps more than five minutes out, and this is a frequent cause of a node
that joined successfully but cannot authenticate afterwards.