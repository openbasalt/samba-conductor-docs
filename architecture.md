# Architecture

Samba Conductor is a set of Go programs that administer a Samba Active
Directory domain from the browser and connect it to other systems. This
page describes how the pieces fit together and the rules they share. Each
component repository has a `docs/design.md` with its own details.

Samba Conductor v2 is a rewrite in Go of an earlier Meteor and MongoDB
application; it shares no code with it.

## Goals

- The web administrator for Samba Active Directory: users, groups, OUs,
  computers, DNS, Group Policy links, password policies (domain and
  fine-grained), lockouts and account health, bulk operations, plus a
  self-service portal for every user of the domain, with invitations and
  password reset by e-mail.
- Integration components that are deployed separately and only when
  needed: delegated directory writes on managed OUs, an identity provider
  backed by AD, provisioning from AD to other directories (and, opt-in,
  from Google Workspace to AD), domain backup and restore, and file shares
  on member servers.
- Native services: each program runs as a systemd unit on the host, with
  its own user and sandbox. No container runtime is required.
- Security over convenience: least privilege, no stored user passwords,
  every change audited, every write previewed and confirmed.

Out of scope: packaging Samba itself (the distribution does that), a
real-time interface, and multi-domain forests.

## Supported platforms

| Platform | Samba | Packages | Support |
|---|---|---|---|
| Debian 13 (trixie) | 4.22 (newer from backports) | .deb | primary |
| Ubuntu 26.04 LTS | 4.23 | .deb | primary |
| Basalt OS | samba-dc (Fedora 44 based) | .rpm, SELinux policy | primary |
| Fedora 44 | samba-dc | .rpm, SELinux policy | supported (the Basalt OS packages) |
| Ubuntu 24.04 LTS | 4.19 | .deb | best effort |

Requirements: Samba 4.19 or later, domain functional level 2016, systemd.
Architectures amd64 and arm64 (x86_64 and aarch64 for RPM).

Debian and Ubuntu build the Samba AD DC with Heimdal Kerberos; Fedora and
Basalt OS build it with the MIT Kerberos KDC. Both are supported and
tested. Samba 4.19 refuses GSSAPI binds over LDAPS without a SASL security
layer even when TLS channel bindings are present; on a 4.19 DC the install
documents set `ldap server require strong auth = allow_sasl_over_tls`.

## Components

Each component is its own repository, Go module and package. Only
conductor is required.

| Program | Role | AD identity | Root |
|---|---|---|---|
| `conductor` | Web administration, self-service | the signed-in user's own Kerberos ticket (AD ACLs apply) | no |
| `conductor-helper` | The few operations that need root on the DC: domain information read locally, online domain backups | root on the local host, over loopback | yes, narrow |
| `conductor-provisioner` | Typed directory writes for conductor that no signed-in user makes: setting a password through a one-time link, enabling an invited account, the Google-first plan operations | a delegated account with rights on user objects below the managed OUs only | no |
| `conductor-idp` | OpenID Connect provider and SAML 2.0 identity provider | users sign in with their own password; claims read with a read-only service account | no |
| `conductor-sync` | Provisioning from AD to Google Workspace | read-only service account | no |
| `conductor-backup` | Encrypted backups to local and S3-compatible storage, restore, restore drills | none (the helper makes the archive) | no on the DC; drills run as root on a separate drill host |
| `conductor-files` | Agent on domain-member file servers: shares, NT ACLs, sessions | the member server's machine account (winbind), acting for the conductor user named in each request | yes, narrow capability set |

conductor and conductor-helper run on a domain controller and ship in one
package. conductor-provisioner runs on the same host as conductor, since
conductor reaches it only through a local socket. conductor-idp and
conductor-sync need only LDAPS and Kerberos to
the DCs, so they run on a DC or on any other host. conductor-files runs on
member servers and refuses to start on a DC: Samba recommends that DCs
host only `sysvol` and `netlogon`. conductor-backup runs on the DC that
takes the backups and, in its drill role, on a dedicated host that is not
a DC.

conductor talks to the other components through narrow typed protocols: a
Unix socket to conductor-helper, a Unix socket to conductor-provisioner,
Unix sockets to the management APIs of conductor-sync and conductor-idp
(all check the peer's UID with `SO_PEERCRED`), and mutually pinned TLS to
each conductor-files agent. In the other direction, conductor serves its
second factor to conductor-idp on a Unix socket only conductor-idp's user
may use. The protocol packages (`ad/helper`, `provapi`, `syncapi`,
`idpapi`, `filesapi`) are public Go packages of the component that serves
them (`idpapi` also holds the second-factor protocol conductor serves).
conductor sends e-mail only through the SMTP relay it is configured with.

## How AD is accessed

- Reads and writes go over LDAPS (636) to a DC with the user's identity.
  At sign-in the user's password is used once, for a Kerberos AS exchange;
  the resulting ticket lives in memory for the session and every LDAP
  operation is a SASL/GSSAPI bind with it. The password is never kept.
  GSSAPI binds carry RFC 5929 `tls-server-end-point` channel bindings,
  which Samba requires.
- An LDAP simple bind over TLS is an explicit, off-by-default fallback for
  hosts that cannot reach a KDC; the password is then held sealed in
  memory, under a per-process key, for the session only.
- AD decides what a user may do. conductor's own role check (by group SID)
  is a second layer, never the only one. No service account acts for a
  signed-in user, with one bounded exception: conductor-provisioner's
  delegated account, for the few typed operations listed below, which
  conductor-provisioner re-checks on its own.
- No access to `sam.ldb` from the web process. Operations that only
  `samba-tool` offers locally go through conductor-helper: a separate
  process on a Unix socket, an allowlist of typed operations (no free-form
  arguments), validated values placed after `--`, credentials passed on an
  inherited pipe or as a credential cache path (never in argv), output
  parsed into structured results, every call logged and audited with the
  caller.
- Group Policy objects are created and deleted with `samba-tool gpo`, run
  by conductor itself with the signed-in user's ticket written to a
  short-lived credential cache in the unit's private `/tmp`. No password,
  no root, no helper.
- DNS is managed over LDAP on the DNS application partitions (`dnsZone`
  and `dnsNode` objects, `dnsRecord` encoded per MS-DNSP) with the user's
  own connection, so the same ACLs and previews apply. Every change
  increments the zone's SOA serial with the old value asserted, so
  concurrent changes conflict instead of overwriting each other. Zones and
  records that AD manages (the domain and `_msdcs` zones, locator records,
  DC host records) are read-only.
- LDAP hygiene: every search is paged (RFC 2696), sorted lists use the
  RFC 2891 sort control, filters are built only by an RFC 4515 builder and
  DNs escaped per RFC 4514, LDAPS requires a pinned CA (TLS 1.2 or later).
- Multi-DC: DCs are discovered from DNS SRV records and computer accounts,
  never configured by name; the local DC is preferred and others are
  used on failure. A refused password is never retried on another DC. A
  session sends its ticket requests to the KDC that issued its TGT first,
  so a change is not read back from a DC that has not replicated it.
  Unlocking an account writes `lockoutTime` on every writable DC and
  reports the result per DC; lockout state is read from every DC.

## Security model

- Sign-in: AD username and password, then a second factor: TOTP (with
  recovery codes) or a WebAuthn security key or platform authenticator.
  A second factor is mandatory for administrators, and by default for the
  delegated roles; for everyone else it is off, optional or required by
  configuration. Administrators without a second factor enroll only
  through a one-time link (24 hours, bound to the username), so a stolen
  administrator password is not enough to register an authenticator.
  WebAuthn can be made mandatory for administrators.
- Roles: administrator = member of a configured AD group matched by SID
  (default Domain Admins, RID 512). Delegated roles (helpdesk: reset
  passwords, unlock, enable and disable; auditor: read-only) map to other
  groups by SID. Roles are re-checked against AD (the user's
  `tokenGroups`, read with the user's own ticket) on every privileged
  request, cached at most 60 seconds; a failed re-check ends the session.
- Protected accounts: members of the administrator groups and of AD's
  built-in privileged groups. Helpdesk never acts on them; administrators
  re-authenticate (password and second factor, a fresh Kerberos sign-in)
  for any write to a protected account or to the membership of a
  protected group, and for settings such as password policy and backups.
- Previews: every write is built as one operation, kept server-side in the
  session (single use, 10 minutes) and shown as its exact LDIF or command
  before confirmation (passwords redacted). The confirmation applies that
  same object; nothing is rebuilt from the confirming request and no
  password travels back to the browser.
- Sessions: server-side, opaque `__Host-` cookie (HttpOnly, Secure,
  SameSite=Strict), idle timeout 15 minutes, absolute 8 hours, a new ID
  after the second factor, "sign out everywhere".
- Web hardening: server-rendered HTML with no JavaScript (CSP
  `script-src 'none'`), except one self-hosted script on the
  second-factor pages for the WebAuthn browser API, loaded with a
  per-response nonce and Subresource Integrity and making no requests of
  its own. CSRF tokens on every form plus Fetch metadata and Origin checks,
  `frame-ancestors 'none'`, HSTS, `Referrer-Policy: no-referrer`, no
  third-party assets. Errors never show LDAP or samba-tool output.
- Rate limits: per client address and per account on sign-in, second
  factor and password pages. The per-account limit sits below the domain
  lockout threshold, so conductor stops before AD locks the account.
  conductor warns when the domain has no lockout policy.
- Audit log: an append-only SQLite table (database triggers refuse
  updates and deletes) with actor, target, change and source address;
  each entry carries the SHA-256 of the previous one, `audit verify` checks
  the chain, and entries export as JSON lines. The chain detects accidental
  or partial edits. It is not keyed and not anchored outside the database,
  so it does not protect against someone with write access to the database
  file, who can rewrite entries and recompute the chain. Protect the
  database file and ship the exported log off the host if you need tamper
  evidence. conductor-idp, conductor-sync and conductor-files keep
  hash-chained audit logs of their own, with the same limits.
  conductor-provisioner's chain is keyed instead (HMAC-SHA-256 under a key
  derived from its state key credential), so a row cannot be changed,
  removed or added with a valid chain without that key.
- State: SQLite (WAL) in `/var/lib/<component>`, holding sessions,
  settings, second-factor secrets, the audit log and job history. TOTP
  secrets are sealed with AES-256-GCM (bound to the user's SID) under a
  key delivered by systemd `LoadCredential=`. AD stays the source of
  truth: no copy of the directory is kept.
- Secrets: files under `/etc/<component>/credentials/`, root 0600, handed
  to the service by systemd credentials; the directory itself is
  inaccessible inside the sandbox. Packages never ship or generate a
  secret.
- systemd sandboxing for every unit: dedicated user, `NoNewPrivileges`,
  `ProtectSystem=strict`, `PrivateTmp`, `PrivateDevices`, restricted
  address families and namespaces, `SystemCallFilter=@system-service`,
  `MemoryDenyWriteExecute` where possible, and an empty capability set
  except for conductor-helper and conductor-files, which keep only the
  file capabilities they need. conductor-helper may reach only localhost.
  On Fedora and Basalt OS a SELinux policy module per component adds a
  mandatory layer that mirrors the sandbox ([packaging](packaging.md)).
- Listening: conductor serves HTTPS on 8443 by default, so its unit needs
  no capability; port 443 is available through systemd socket activation
  or a TLS reverse proxy on loopback. Plain HTTP is served only to a
  loopback proxy.
- Supply chain: reproducible builds, signed checksums and repositories,
  CycloneDX SBOMs, pinned tools and CI actions, `govulncheck` on every
  change ([packaging](packaging.md), [verifying
  releases](verifying-releases.md)).

## Integration components

### conductor-provisioner

conductor's web process holds no standing write credential on AD: every
change it makes runs with the signed-in user's own ticket. Some changes
have no signed-in user to run them: the first password of an invited
account, a password reset from an e-mailed link, and the accounts that the
Google-first mode creates and updates. conductor-provisioner makes those,
and holds the only standing AD write credential in Samba Conductor.

- The delegated account is a member of Domain Users only, marked
  sensitive (never delegated), and has access entries on the managed OUs,
  inherited by the user objects below them only: Reset Password and
  writes to `lockoutTime`, `pwdLastSet` and `userAccountControl`. On the
  managed OU of a Google-first scope it also gets create and delete for
  user objects (a move needs both) and writes to the Google-owned
  attributes and the marker. It has no rights on the domain head,
  `CN=Users`, `CN=System` or the `Domain Controllers` OU, no `WRITE_DAC`
  or `WRITE_OWNER`, and no write access to `member`, `adminCount` or
  security descriptors. `conductor-provisioner delegation preview` prints
  the exact `samba-tool dsacl` commands.
- It runs as its own user under its own sandboxed, socket-activated unit
  on conductor's host, with no network listener. Its API socket
  (`/run/conductor-provisioner/api.sock`, group `conductor`, 0660) admits
  only conductor's user (`SO_PEERCRED`). Requests are typed operations
  from an allowlist, decoded strictly; there is no free-form LDAP.
- Invariants it checks on every operation, whatever conductor sends: the
  target is read fresh by SID and must be strictly below a configured
  scope OU (parsed DN comparison, no referrals; built-in containers cannot
  be configured); it must be a user account; it must not be privileged
  (nested member of the administrative groups or of conductor's role
  groups, `adminCount` set, or holding more than read on the domain head,
  an OU, AdminSDHolder or a Group Policy object); hourly ceilings per kind of
  operation and a daily limit of tokens per account come from its own
  root-owned configuration. The delegated account itself holds rights on
  the scope OUs, so it is privileged and can never act on itself.
- One-time tokens for invitations and resets are issued, checked and
  consumed by conductor-provisioner, stored only as a hash and bound to the
  account's SID, objectGUID and `pwdLastSet`: a password change by any
  other path makes them stale. A new password crosses the socket once and
  is never logged, audited, stored or returned.
- The Google-first operations act only on an account carrying the marker
  of the Google account the request names, and replace an attribute only
  when its current value is still the one the plan read. Nothing writes
  the marker after the account is created, and nothing deletes.

What a compromised conductor web process can and cannot do through it:

| It can | It cannot |
|---|---|
| issue and use tokens, so set the password of an ordinary account below the scope OUs and enable an invited one, within the ceilings | act on a privileged account or outside the scope OUs; raise the ceilings or change the scope OUs |
| revoke tokens | read a stored token or recover a password |
| create disabled Google-first accounts, change their Google-owned attributes, enable, disable and move them within their scope OU, within the ceilings | act on an account without its marker, change a marker, delete, write a group membership, a security descriptor or any other attribute |

A stolen password of the delegated account, used directly against AD,
gives what the delegation grants on the managed OUs and nothing else; the
account's credential deserves the same care as a helpdesk
administrator's. Design and protocol:
<https://github.com/openbasalt/samba-conductor-provisioner/blob/main/docs/design.md>;
the delegation:
<https://github.com/openbasalt/samba-conductor-provisioner/blob/main/docs/delegation.md>.

### E-mail, invitations and password reset

conductor sends e-mail through one SMTP submission relay (STARTTLS or
implicit TLS with the certificate verified; plain SMTP only to a relay on
loopback). Messages wait in a queue in conductor's database, sealed with
the key that seals the second-factor secrets, with retries, an expiry per
message (a link's message expires with the link) and an hourly ceiling;
the log next to the queue keeps the recipient's domain, never the address
or the content. Templates (plain text and HTML, branded, overridable) are
listed in conductor's
[mail.md](https://github.com/openbasalt/samba-conductor/blob/main/docs/mail.md).

On top of the mail queue and conductor-provisioner, conductor offers
invitations (an administrator or helpdesk sends a one-time link; the
person sets the first password and enrolls a second factor, and the
account is enabled), password reset by e-mail from a public form, an
optional recovery address and a notice after every password change. How
the three parts share the work, and what each one refuses:
[passwords by e-mail](passwords.md).

### conductor-idp

- OpenID Connect provider: Authorization Code with PKCE (S256) only, also
  for confidential clients; discovery, JWKS with ES256 keys sealed at rest
  and rotated with an overlap, token, userinfo, revocation and end
  session. Codes are single use; refresh tokens are stored hashed and
  rotated, and reuse revokes the chain; every refresh re-checks the
  account and its groups in AD.
- SAML 2.0 identity provider: SP- and IdP-initiated SSO, signed responses
  and assertions, optional encryption, per-SP NameID and attribute
  mapping, staged signing-key rotation. This is how Google Workspace and
  similar services sign users in with their AD password.
- Clients and service providers are registered by administrators, with
  exact redirect URIs, allowed AD groups by SID, scopes, a consent screen
  for third-party clients and optional mandatory second factor per
  client. `sub` is the user's objectGUID; the `groups` claim carries group
  names or SIDs, nested membership included.
- Users authenticate against AD with the same Kerberos path as conductor.
  On the same host, conductor-idp uses conductor's second factor through a
  local socket: one enrollment (authenticator app, recovery codes,
  security keys) and conductor's role-based policy for both; a security
  key registered in conductor works at the IdP when the IdP's origin is
  one of conductor's WebAuthn origins (a shared parent domain as RP ID, or
  WebAuthn related origins). Elsewhere it keeps its own TOTP enrollments.
  Same web hardening; the only script is the WebAuthn one, on the
  second-factor page, under a per-response CSP nonce.
- SAML single logout: a LogoutRequest signed with the HTTP-Redirect
  binding's query signature by the SP's registered certificate ends the
  session at once (anything else asks the user first, since the IdP
  verifies no XML signature); every other SP of the session then receives
  a signed LogoutRequest in turn. A logout started at the IdP or by an
  OpenID Connect client logs the SAML participants out too.
- Managed from conductor's "Single sign-on" section through a local
  management API (Unix socket, `SO_PEERCRED`, typed operations): clients
  and SPs with guided presets (Google Workspace, Grafana, Nextcloud,
  GitLab, generic), metadata import, a claims and assertion preview for a
  real user, signing keys and staged rotation, session lifetimes and the
  consent note, and sign-in activity per application. Every change is
  confirmed in conductor with a fresh second factor and audited on both
  sides; a client secret is shown once and stored only as a hash.

### conductor-sync

- One-way provisioning from AD to Google Workspace through the Admin SDK
  Directory API (a service account with domain-wide delegation): users,
  groups, memberships, suspension, org unit placement. The connector
  interface is generic; Google Workspace is the connector it ships.
- Scope by OU and by include and exclude groups (nested, by DN or SID);
  configurable address templates and attribute mapping.
- Plan, then apply: every run computes a plan against the target and the
  recorded links (AD objectGUID to target ID). A new configuration starts
  in dry-run mode, the first apply is manual, and scheduled runs apply
  only plans inside configured safety limits (creates, suspends, renames,
  share of accounts touched, source shrinkage); otherwise the run stops,
  records the plan as blocked and alerts.
- Never deletes: an account that leaves the scope is suspended. Deletion
  is a separate, manual, typed confirmation of one long-suspended account.
  Only accounts carrying the sync's ownership marker are touched.
- Every operation is journaled before and after it is sent, so a crashed
  run is resumed safely by the next one.
- Passwords are not synchronized (Samba keeps hashes Google cannot use);
  users sign in to Google through conductor-idp's SAML provider.
- conductor manages it through a local management API (Unix socket,
  `SO_PEERCRED`, typed operations); applies from the browser are bound to
  the digest of the reviewed plan, and secrets are write-only.

### Google-first mode

The Google-first mode is the other direction, opt-in and off by default:
for the people of chosen scopes, Google Workspace is the source of truth
and AD follows. A scope pairs a managed OU (with a groups OU and a
quarantine OU below it) with Google org units, optionally restricted to
members of Google groups. It never shares an OU tree or an org unit with
the AD to Google direction.

- conductor-sync reads Google with read-only scopes and AD with its
  read-only account and returns a plan of typed AD operations (updates,
  primary address changes, re-enables, creates, disables) with a digest
  and safety limits. It writes neither to Google nor to AD.
- conductor shows the plan; an administrator applies it with the digest
  typed and a fresh second factor. conductor applies each operation
  through conductor-provisioner, sends an invitation to each account
  created, and reports the results to conductor-sync, which records the
  links.
- conductor-provisioner re-checks every operation: scope, privilege, the
  marker of the Google account, and the values the plan read.
- Privileged access is granted only in AD: nothing from Google creates or
  widens a privileged membership, Google administrator roles map to
  nothing, and a privileged AD account is never changed by a plan (it is
  listed as skipped so a person acts in AD). Accounts suspended,
  deleted or no longer selected in Google are disabled and moved to the
  quarantine OU, never deleted.
- Google keeps its own sign-in in this mode: conductor refuses to turn it
  on while a conductor-idp application targets Google, and the reverse.
  The Google password is never written to AD or read from it; people set
  their AD password through an invitation.

Details: conductor-sync's
[google-first.md](https://github.com/openbasalt/samba-conductor-sync/blob/main/docs/google-first.md)
(reads, plan, limits, field ownership) and conductor's
[google-first.md](https://github.com/openbasalt/samba-conductor/blob/main/docs/google-first.md)
(settings, plan review, apply, fields shown as managed by Google).

### conductor-files

- An agent on domain-member file servers (Samba with winbind), never on a
  DC. conductor's administration pages manage file servers through it.
- Transport: TLS 1.3 with client certificates and no CA. Each side
  generates its own key and pins the other's (SHA-256 of the public key).
  Enrollment starts on the file server: root runs `conductor-files
  enroll-code`, which prints a one-time code carrying a token and the
  agent's key pin; the administrator pastes it into conductor, which
  connects pinning that key. Revocation is removing a pin on either side.
- Shares are written through Samba's registry configuration (`net conf
  import`, one section replaced in a transaction); `smb.conf` is never
  edited, and only shares the agent created are changed or removed.
- Access is granted by AD group, matched by SID and resolved through
  winbind, at three levels (read, modify, full), applied as an NT ACL on
  the share folder (`samba-tool ntacl set --use-s3fs`) and mirrored in
  the share permissions (`sharesec`).
- Paths stay strictly below configured roots; every component is opened
  with `O_NOFOLLOW`, and the root and every parent must be root-owned and
  not writable by others.
- Plan, then apply bound to a digest: the plan shows the exact section,
  ACLs before and after, and the commands in order; the apply re-plans and
  runs only if the digest still matches. conductor checks roles and
  requires a fresh second factor for every write; each request carries the
  acting user, and both sides audit it.
- Live sessions and open files come from `smbstatus --json`.

### conductor-backup

- Online backups with `samba-tool domain backup online`, run by
  conductor-helper over loopback with a dedicated account that holds only
  the three replication rights, not an administrator. Its password is a
  systemd credential of the helper only.
- The archive (the Samba backup with SYSVOL, conductor's state with
  sessions removed and second-factor secrets still sealed, Samba and
  component configuration) is encrypted with age (X25519) by the helper
  before it is written anywhere, to recipients listed in a root-owned
  file. Private keys are never on a DC: operators keep theirs offline and
  the drill host holds its own.
- Destinations: local directories and S3-compatible object storage, each
  archive with a manifest signed by the DC's Ed25519 key (sizes, SHA-256,
  versions, recipients). Every copy is read back and checked. Optional
  S3 object lock.
- Schedule and retention (daily, weekly, monthly) are editable by
  administrators in conductor, previewed and re-authenticated.
  Destinations, credentials and recipients are host configuration only, so
  a compromised web process cannot redirect future backups.
- Restore: `conductor-backup restore` checks the download against the
  signed manifest and runs `samba-tool domain backup restore`. The restore
  runbook is in the conductor repository.
- Restore drills run on a drill host that is not a DC and has no route to
  the DCs: the newest backup is restored into a sandbox with its own
  network, mount, PID, UTS and IPC namespaces and no outside network, then
  checked (LDAP, SRV records, a Kerberos sign-in of a probe account, the
  user count, sample SIDs). The measured restore time and every check go
  into a signed report that conductor shows and audits.
- Alerts by e-mail and signed webhook when a backup or drill fails, a
  destination misses a backup, or the last good backup is too old;
  conductor shows a dashboard banner.

## Packaging and operation

- One .deb and one .rpm per component from the same static Go binaries,
  plus a SELinux policy package per component for Fedora and Basalt OS
  ([packaging](packaging.md)). The `ad` library is not packaged.
- A fresh installation enables and starts nothing: each component needs
  configuration and secrets first.
- `conductor setup`: interactive first run. It detects the realm and the
  local DC, pins the domain CA (from a file, or by confirming the
  fingerprint read from the DC), resolves the role groups to SIDs with a
  one-time sign-in, generates the TOTP key, writes
  `/etc/conductor/conductor.toml` and can issue the first administrator's
  enrollment link.
- TLS: conductor serves HTTPS with a certificate the administrator
  installs in `/etc/conductor/tls/`, or sits behind a TLS reverse proxy on
  loopback.
- Upgrades: database migrations are embedded in the binaries,
  configuration lives in `/etc/<component>/*.toml`, state in
  `/var/lib/<component>`, and no state is inside a package. Running
  services are restarted after an upgrade; enabled state is kept.

## The ad library

The AD access layer is a standalone Go module,
`github.com/openbasalt/samba-conductor-ad` (package `ad`), used by
conductor, conductor-provisioner, conductor-idp, conductor-sync and
conductor-backup. It has no global state, takes a `context.Context` on
every network call and returns typed errors. It provides:

- connections: LDAPS with CA pinning, DC discovery from SRV records,
  preferred DCs and failover;
- Kerberos: its own AS exchange (to read AD's error details), an
  in-memory credential cache, GSSAPI binds with channel bindings, password
  change through kpasswd (RFC 3244), and classification of every AD bind
  sub-code (expired, must change, locked, disabled, expired account), with
  "password verified" reported only when AD has proven the password right;
- paged and sorted searches, the RFC 4515 filter builder and RFC 4514 DN
  escaping (`ad/escape`), SID and GUID handling (`ad/sid`);
- typed models and operations with Preview, then Apply: users, groups,
  OUs, computers, DNS zones and records, Group Policy links, domain and
  fine-grained password policies, with optimistic concurrency where AD
  allows it;
- typed `samba-tool` operations (`ad/sambatool`) with an exact command
  preview, and the helper protocol (`ad/helper`).

The library is independent of the web interface: a terminal tool can reuse
the ad library. Design details:
<https://github.com/openbasalt/samba-conductor-ad/blob/main/docs/design.md>.
