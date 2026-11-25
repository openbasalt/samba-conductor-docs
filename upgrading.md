# Upgrading

## Upgrading to conductor 0.3.0

Released on 2026-11-24, together: conductor 0.3.0, conductor-idp 0.2.0,
conductor-sync 0.3.0, conductor-backup 0.2.0 and conductor-provisioner
0.1.2, all built with Go 1.27.4. The packages are in the `stable` suite of
the APT repository at <https://obpkg.org/apt> and in the basalt-tools
repository at <https://obpkg.org/basalt-tools> (x86_64). Read this page
before upgrading from conductor 0.2.x; the release notes of each component
carry the full list of changes.

### Before the upgrade

- Back up the state. The database migrations of conductor and
  conductor-idp are forward only: conductor 0.2.1 does not start on a
  database opened by 0.3.0, and conductor-idp 0.2.0 applies its schema
  migration 0004 at start, also forward only. Copy `/var/lib/conductor` and
  `/var/lib/conductor-idp` if you may need to go back, and take a domain
  backup.
- conductor-provisioner: 0.1.2 refuses to start while a privileged account
  sits below a scope OU, and 0.1.1 does not report these accounts (its
  `status` exits 0). Check by hand that no member, direct or nested, of
  the protected groups (Domain Admins, Enterprise Admins, Schema Admins,
  Group Policy Creator Owners, Domain Controllers, Read-only Domain
  Controllers, Enterprise Read-only Domain Controllers, Key Admins,
  Enterprise Key Admins, the BUILTIN Administrators, Account, Server,
  Print and Backup Operators and Replicator, the domain's DnsAdmins,
  conductor's role groups and the groups in `extra_group_sids`) and no
  account with `adminCount` other than 0 lives under a scope OU, and move
  any that does. Until they are moved out the service does not start and
  systemd keeps retrying it.
- conductor-sync: `google.auth_mode` is new and defaults to
  `"service-account"`, where the service account acts as itself under a
  custom administrator role. An installation that relied on domain-wide
  delegation without setting anything gets failing token requests after
  the upgrade. Either create the role and assign it to the service account
  ([google-credentials.md](https://github.com/openbasalt/samba-conductor-sync/blob/main/docs/google-credentials.md)),
  then remove the delegation and rotate the key, or set
  `auth_mode = "domain-wide-delegation"` (with `admin_subject`) before
  upgrading.
- conductor-idp: an e-mail address is asserted only when no other
  directory object holds it. If relying parties or SAML service providers
  key accounts on e-mail, look for duplicate addresses in `mail`,
  `proxyAddresses` and `userPrincipalName` first.
- conductor-backup: drill hosts need user namespaces (the restored DC runs
  as root of a user namespace mapped from `drill.userns_base`, default
  1879048192, for 4194304 IDs that nothing else on the host may use) and
  must be of the same distribution family as the DCs, since drills across
  Kerberos implementations (Heimdal on Debian, MIT Kerberos on Fedora) are
  refused.

### Upgrade conductor and conductor-idp together

With the Single sign-on section on (`[idp] enabled = true` in
`conductor.toml`, `[api] enabled = true` in `idp.toml`), conductor 0.3.0
speaks version 2 of conductor-idp's management protocol. The protocol has
no version negotiation: conductor 0.3.0 needs conductor-idp 0.2.0, and
conductor-idp 0.2.0 refuses requests from conductor 0.2.x. Upgrade both in
the same step. The second-factor socket protocol between the two is
unchanged.

```sh
# Debian and Ubuntu
sudo apt update
sudo apt install --only-upgrade conductor conductor-idp conductor-sync conductor-backup conductor-provisioner

# Basalt OS and Fedora
sudo dnf upgrade conductor conductor-idp conductor-sync conductor-backup conductor-provisioner
```

Upgrade only the components that are installed on the host.

### Configuration files on Debian and Ubuntu

With `apt`, every configuration file edited since install makes dpkg ask
whether to keep it. conductor-helper's `/etc/conductor/helper.toml` is one
of them in this release: it gains a `[backup.retention_floor]` section. Keep
the installed version; the new default lands next to it as
`helper.toml.dpkg-dist`. For an unattended upgrade pass
`-o Dpkg::Options::=--force-confold`.

### conductor's audit key

conductor 0.3.0 chains new audit rows with HMAC-SHA256 under a root-only
key, `/etc/conductor/credentials/audit-key`, and does not start without
it. The .deb and .rpm packages create the key on install and before an
upgrade restarts conductor, and never replace an existing one; `conductor
setup` creates it when it is missing. Installs from source or with their
own unit file must create it before starting 0.3.0:

```sh
sudo sh -c '(umask 077; od -An -tx1 -N32 /dev/urandom | tr -d " \n" > /etc/conductor/credentials/audit-key)'
```

and add `LoadCredential=audit-key:/etc/conductor/credentials/audit-key` to
the unit (or set `[audit] key_file`).

Back the key up offline, with the TOTP key and the operator's age key.
Neither key is in conductor-backup archives, and without the audit key the
keyed part of the chain cannot be verified after a restore. Container
installs: see [upgrading to 0.3.0](containers.md#upgrading-to-030) in
the container guide.

`conductor audit verify` and `conductor enroll-link` now read the key as
root and then drop to the owner of the database, so they run with `sudo`
(`sudo conductor audit verify`) instead of `sudo -u conductor`. Right after
the upgrade the check can exit 1 with "not keyed yet": the `audit.key_start`
marker is written with the first audited event, for example a sign-in. Run
it after that event. Rows written before the upgrade keep their plain
SHA-256 hash, and the command says from which row the chain is keyed.

### After the upgrade

- conductor-provisioner: `sudo conductor-provisioner status` lists the
  privileged accounts below the scope OUs and exits 3 when there are any;
  `delegation preview` shows the same list for an OU.
- conductor-sync: run `conductor-sync check-config` and a `plan` before the
  first apply after switching to the service-account mode. With
  `adopt = "email"`, matches are now candidates that wait for
  `conductor-sync adopt approve`; with `password_reset =
  "created-and-adopted"`, accounts adopted before this version need an
  approval too.
- conductor-idp: `email_verified` is now a per-client opt-in, off by
  default. Turn it on for the relying parties that require it (the
  "Assert email_verified" box in conductor's client form, or the
  `-email-verified` flag of the client commands). Second-factor enrollment
  with the local backend goes through a one-time link from an
  administrator by default (`mfa.enrollment = "link"`).
- conductor: the protected account set is wider (read-only domain
  controllers, key admins, Replicator, DnsAdmins, conductor's helpdesk and
  auditor groups, accounts with `adminCount=1`, and groups nested into
  them); helpdesk can no longer act on these accounts. conductor-helper
  refuses backup policies below the retention floor (7 daily, 4 weekly, 12
  monthly by default).
- conductor-backup: the drill report has a new `sandbox` check and a new
  `sysvol_acl` check. Restores and drills rebuild the SYSVOL ACLs from the
  group policy objects in the directory; ACLs set by hand on SYSVOL outside
  group policy management must be set again after a restore. A drill host
  refuses an older "newest backup" until `conductor-backup drill --now
  --accept-older`.
