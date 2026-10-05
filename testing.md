# Testing

Samba Conductor is tested at four levels: unit tests and static checks on
every change, an integration lab with real Samba domain controllers, a
package lab that installs, upgrades and removes the packages on every
supported distribution, and restore exercises that rebuild a domain from
its backups.

The lab tooling is not published yet. The usage reports in each component
repository (`docs/usage-*.md`) are transcripts of lab runs: what was set
up, what was run and what was checked.

## Unit tests and gates

Every component repository runs the same gates, locally with `make check`
and in CI on every push and pull request:

| Gate | What it checks |
|---|---|
| `gofmt` | formatting |
| `go vet` | suspicious constructs |
| `staticcheck` | static analysis (pinned version) |
| `govulncheck` | known vulnerabilities in the code paths actually reached (pinned version) |
| `go test -race` | unit and integration tests with the race detector |

Packaging changes add `make package`, `make lintian` (Debian 13's lintian,
clean at warning level) and `make rpmlint` (Fedora 44's
`rpmlint --strict`), and a rebuild that must produce the same SHA-256 sums.

Notable test practices:

- The `ad` library fuzzes its parsers and builders (filter and DN
  escaping, SID decoding, samba-tool argument building, the helper
  protocol) with `make fuzz`.
- conductor-sync runs every test against a fake Google Directory API; no
  test ever writes to a real Google Workspace.
- Wire formats that differ between implementations are tested in every
  known layout (for example the SASL wrap tokens of Heimdal and MIT
  Kerberos), along with tampered and malformed input.
- Tests use the reserved `.test` top-level domain, documentation address
  ranges (192.0.2.0/24) and `example.com`, never real names.

## Integration lab

The integration lab is a set of virtual machines on an isolated libvirt
network:

- two Samba AD domain controllers (Debian 13) in the realm
  `lab.conductor.test`, on the reserved `.test` top-level domain,
  functional level 2016, replicating with each other;
- a domain-member file server (winbind, registry configuration, NT ACLs)
  for conductor-files;
- for backups, an S3-compatible object store and a mail catcher on the
  lab network, and a drill host on a separate network with no route to
  the DCs.

The lab network has no Internet access: packages reach the VMs only while
they are provisioned, and time comes from the host clock. DC certificates
come from a lab CA with proper subject alternative names, which the
tests pin. Secrets are generated for each lab and never leave the lab
host.

The domain is seeded with data that exercises the edge cases: 2,500 users
in several OUs, nested groups several levels deep, a group with more
members than one LDAP range retrieval returns, an account for each AD
bind sub-code (expired password, must change, locked, disabled, expired
account), a name that needs DN escaping, a delegated helpdesk OU,
fine-grained password policies, Group Policy objects with enforced,
disabled and blocked links, and DNS zones with more records than one page.

Snapshots of every VM let a run start from a known state: a reset takes
seconds, and the end-to-end suites reset the lab before each run. Some
components use their own copy of the lab (their own network and domain),
so they never disturb another component's runs.

Integration tests:

- the `ad` library runs its tests against both DCs, including
  `samba-tool` operations as root on a DC;
- conductor-files runs its tests on the member file server against the
  installed agent and its sandboxed unit, including SMB access as domain
  users;
- conductor-sync runs the real binary against the lab domain and a
  loopback fake of the Google Directory API.

### End-to-end suites

conductor has a Playwright suite that drives the real application on a DC
of the lab, once with a desktop viewport and once with a mobile one, on a
freshly reset lab each time: refused sign-ins and their messages, the
per-account rate limit, expired passwords, second factor enrollment (TOTP,
and WebAuthn through the browser's virtual authenticator), administration
of users, groups, OUs, DNS, Group Policy and password policies, lockouts
across both DCs, bulk operations, backups and drills, the Google
Workspace sync section, file servers with SMB access checked as domain
users, self-service, and what each role (administrator, helpdesk,
auditor) may and may not see. The browser trusts exactly conductor's
certificate.

conductor-idp has its own suite, with an independent OpenID Connect
client, a real application signing in through it and a SAML service
provider, also on desktop and mobile. conductor's suite also runs
conductor-idp next to conductor: applications registered from the Single
sign-on section, a passkey registered in conductor used at the IdP, and
SAML single logout started by a service provider. The OpenID Foundation
conformance suite (basic and config plans) is run locally against a test
deployment; its results and conductor-idp's deliberate deviations are in
conductor-idp's decisions (D13).

The lab has found real defects that unit tests could not: Samba requiring
TLS channel bindings for GSSAPI over LDAPS, Samba's KDC reporting an
expired password before checking it, a ticket request reaching a DC that
had not yet replicated a password change, and MIT Kerberos sending SASL
wrap tokens in a different layout from Heimdal.

## Package lab

The package lab installs the real packages from a signed repository built
for the run, on fresh virtual machines of each supported distribution:

| Distribution | Samba | Machines |
|---|---|---|
| Debian 13 | 4.22 | a DC and a member file server |
| Ubuntu 26.04 | 4.23 | a DC and a member file server |
| Ubuntu 24.04 (best effort) | 4.19 | a DC |
| Basalt OS | 4.24, MIT Kerberos | a DC and a member file server, UEFI Secure Boot and an emulated TPM |

For each distribution it:

1. builds two versions of every package (vN and vN+1) and checks that a
   rebuild is byte-identical;
2. installs vN and checks that nothing was enabled or started;
3. configures every component as its install document says (conductor
   setup and the first administrator, the helper, the identity provider,
   the sync in dry-run with its API socket, backups to a local destination
   (on Basalt OS also to S3) with a backup made and verified, the file
   server agent installed and checked);
4. checks that every unit is active and answers (HTTPS, OpenID Connect
   discovery, sockets, a backup in the spool);
5. upgrades to vN+1 and checks that edited configuration, credentials,
   databases and the enabled and running state are kept;
6. removes the packages, then purges them (.deb), and checks that no
   package file, directory, unit or statoverride is left.

On Basalt OS the run also covers the browser flows (second factor
enrollment and sign-in, user and OU changes, domain information through
conductor-helper, Group Policy changes with samba-tool, self-service
password change, an OpenID Connect code flow with PKCE through
conductor-idp) and conductor-files' integration tests, all with SELinux
enforcing. The run passes only with zero AVC denials after each phase, and
on removal the policy modules, file labels and units must be gone.
Policy development uses a permissive run to collect every denial at once;
only an enforcing run with no denial counts.

arm64 packages are installed from the same repository in arm64 containers
(Debian 13 and Ubuntu 26.04) under QEMU user emulation, and every binary is
run; the release workflow includes the same smoke test for the arm64 .rpm on
Fedora 44.

## Restore exercises

Backups are only as good as the last restore, so restores are tested in
three ways:

- Restore drills, part of the product, run on a schedule: the newest
  backup is restored into an isolated sandbox and checked (LDAP answers,
  SRV records exist, a probe account gets a Kerberos ticket, the user
  count matches the source, sample SIDs are unchanged), with the restore
  time measured.
- Failure tests: a wrong decryption key, a tampered object in the bucket,
  the bucket unavailable (an alert is sent and the backup retried), pruning
  that must keep the last good backup, and a missed schedule that must
  raise an alert.
- A full-forest restore exercise: both DCs are switched off, a DC is
  rebuilt on a fresh machine from the latest backup following the restore
  runbook, conductor is restored with its state, a fresh second DC is
  joined, and the directory's facts are compared with those taken before
  the loss, with the recovery time measured.
