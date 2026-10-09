# Container images

Samba Conductor publishes five container images, built from the same
released and signed packages as the .deb and .rpm files, so a container
and a package of the same version run the same binaries. The images, the
compose files and the release tooling live in
[samba-conductor-containers](https://github.com/openbasalt/samba-conductor-containers).

## When to use the images

| Image | Contents | Runs as | Support level |
|---|---|---|---|
| `samba-conductor-dc` | Samba AD DC (Debian 13 packages), chronyd, conductor-helper, conductor-backup (restores only), `sc-dc-init` | root in the container, nine capabilities | preview |
| `samba-conductor` | conductor and Samba's `samba-tool` (GPO create and delete) | UID 2093 | same as the packages |
| `samba-conductor-idp` | conductor-idp (distroless) | UID 2095 | same as the packages |
| `samba-conductor-sync` | conductor-sync (distroless) | UID 2094 | same as the packages |
| `samba-conductor-backup` | conductor-backup (distroless) | UID 2097 | same as the packages |

The DC image is a preview: it is meant for labs, evaluation and small
single-site domains. For production domain controllers the packages remain
the recommended installation (see the
[install guide](https://github.com/openbasalt/samba-conductor/blob/main/docs/install.md)).
The four service images are supported like the packages and can also run
next to domain controllers installed from packages.

There is no image for the member file server agent (conductor-files needs
a joined smbd and winbindd host) or for restore drills (a drill builds
namespaces, which needs `CAP_SYS_ADMIN`); both stay package installs.

## Quick start (lab)

On a Linux host with Docker Engine and Compose v2 and a synchronized
clock:

```sh
git clone https://github.com/openbasalt/samba-conductor-containers
cd samba-conductor-containers/compose
cp .env.example .env          # domain, NetBIOS name, host name, this host's address
./make-secrets.sh             # random passwords in ./secrets (0600)
docker compose up -d dc       # provisions the domain (about a minute)
docker compose run --rm conductor-setup
docker compose up -d
```

`conductor-setup` prints a one-time enrollment link for the first
administrator; open it in a browser (conductor listens on port 8443 of
`SC_HOST_IP`). The DC creates a certificate authority limited to the
domain and to `SC_HOST_IP`; its certificate is on the `dc-public` volume
(`ca.pem`) for clients that should trust it.

Add-ons are listed in `COMPOSE_FILE` in `.env`; each file starts with the
steps it needs:

| Add-on | What it adds |
|---|---|
| `addons/idp.yaml` | conductor-idp (OpenID Connect and SAML 2.0), with conductor's second factor and passkeys |
| `addons/sync.yaml` | conductor-sync (Google Workspace provisioning, starts in dry-run) |
| `addons/backup.yaml` | conductor-backup: scheduled, encrypted, signed domain backups |
| `addons/host-network.yaml` | the DC on the host's network |
| `addons/provided-tls.yaml`, `addons/idp-provided-tls.yaml` | certificates from your own CA |
| `addons/join.yaml` | an additional DC for an existing domain |
| `addons/restore.yaml` | a full-forest recovery from a backup |

## Images, tags and verification

Registries, with the same digests in both:

- `docker.io/openbasalt/<image>`
- `ghcr.io/openbasalt/<image>`

Every image is built for `linux/amd64` and `linux/arm64`.

| Tag | Example | Meaning |
|---|---|---|
| immutable | `samba-conductor-dc:0.1.0-samba4.22.11-r1`, `samba-conductor:0.1.0-r1` | one build; never moved or overwritten. `-rN` counts rebuilds of the same component versions (Debian security updates, a new base image) |
| `testing` | | the newest release under test |
| `X.Y.Z`, `X.Y`, `X`, `latest` | `0.1.0`, `0.1`, `0`, `latest` | a release that passed its tests; the version is the containers release, the same for the five images |
| `sambaX.Y` (DC only) | `samba4.22` | the newest tested DC image of a Samba line |

The compose files use `SC_IMAGE_PREFIX` (the registry) and `SC_TAG`, whose
default is the release of the checkout, never `latest`. In production,
pin a digest: every release lists its image references with their digests
in `IMAGES.txt`.

Each published digest is signed with cosign (keyless, by the release
workflow of the containers repository) and carries CycloneDX SBOMs and
build provenance attestations. Verify before you run an image:

```sh
cosign verify docker.io/openbasalt/samba-conductor-dc@sha256:<digest> \
  --certificate-oidc-issuer https://token.actions.githubusercontent.com \
  --certificate-identity-regexp '^https://github.com/openbasalt/samba-conductor-containers/\.github/workflows/release\.yml@refs/tags/v'
gh attestation verify oci://docker.io/openbasalt/samba-conductor-dc@sha256:<digest> --owner openbasalt
```

The same commands work with `ghcr.io/openbasalt/<image>@sha256:<digest>`. A digest that has
been promoted to the version tags and `latest` also carries a second
signature, made by the promote workflow of the same repository; the
release signature above is the one to require.

Without the Sigstore tools, use the OpenBasalt release key
([verifying releases](verifying-releases.md)): check `SHA256SUMS.asc` of
the containers release, check `IMAGES.txt` against `SHA256SUMS`, then pull
by the digests listed there. A digest is the hash of the content, so no
registry can change what it names.

Podman and other engines that check signatures themselves can be
configured to require the release signature; the verification above, done
once per digest before the digest is pinned, gives the same guarantee.

## Networking

A DC uses 53 (TCP and UDP), 88 (TCP and UDP), 123/udp, 135, 389 (TCP and
UDP), 445, 464 (TCP and UDP), 636, 3268, 3269 and an RPC range (default
49152-49159, `SC_RPC_PORTS`). NetBIOS (137 to 139) is off.

| Mode | When | How | Limits |
|---|---|---|---|
| bridge (default compose) | labs, evaluation, hosts whose firewall you cannot change | the ports are published on `SC_HOST_IP` only; the DC registers `SC_HOST_IP` in DNS, not the container's address | one DC per host address; the RPC range must be published exactly |
| host (`addons/host-network.yaml`) | production-like setups, several DCs | `network_mode: host`; Samba binds only `SC_HOST_IP` and loopback, so the host's own resolver and services keep their addresses | the host firewall must admit the ports (firewalld: the `samba-dc` service); nothing else may use port 445 on that address |
| macvlan or ipvlan (`SC_NETWORK_MODE=macvlan`) | a DC with its own LAN address | the container's LAN address is `SC_HOST_IP`, no NAT | the host cannot reach the container without a shim interface; macvlan needs a switch that accepts extra MAC addresses |

Never publish the DC's ports on `0.0.0.0`: port 53 collides with local
resolvers such as systemd-resolved.

DNS: the DC is its own resolver. `SC_DNS_FORWARDERS` lists the upstream
resolvers for names outside the domain (never Docker's embedded
127.0.0.11, which exists only inside a container). Clients and member
servers must use the DC, or a resolver that forwards the domain to it, as
their DNS server. The other containers of the stack use the DC as their
only resolver and know its name through an `extra_hosts` entry. In bridge
mode IPv6 is off; in host mode Samba registers the addresses of its
interfaces, and `SC_HOST_IP6` names the advertised IPv6 address.

conductor (8443) and conductor-idp (9443) are published on `SC_HOST_IP`,
or can sit behind your reverse proxy on a private network; keep TLS
between the proxy and the service.

## Persistence

| Volume | Holds | Losing it means |
|---|---|---|
| `dc-data`, `dc-config` | the domain database, sysvol, TLS files, the state file; `smb.conf` | this DC is gone: join a new one (other DCs alive) or restore from a backup |
| `dc-public` | the CA certificate, the domain's `krb5.conf` and client `smb.conf` for the other containers | recreated by the DC at its next start |
| `dc-issued` | the web certificates the DC's self-signed CA issues for conductor and conductor-idp (read by the setup one-shots) | reissued at the next start |
| `conductor-etc`, `conductor-state`, `conductor-cred` | conductor's configuration, database (second factors, audit log) and TOTP key | second-factor enrollments and the audit log; keep an offline copy of the TOTP key |
| `idp-etc`, `idp-state`, `idp-cred` | registered applications, signing keys, IdP settings and branding | applications must be registered again |
| `sync-etc`, `sync-state`, `sync-cred` | sync configuration, state and state key | the next full run rebuilds the state |
| `backup-etc`, `backup-state`, `backup-cred` | backup configuration, spool, the DC's signing key | publish a new public signing key to the drill host and restore configuration |
| `backup-local` | archives of a local backup destination | those archives; keep a remote destination too |
| `helper-run`, `idp-run`, `sync-run`, `mfa-run` | sockets between the containers | nothing (recreated) |

File systems: the DC stores NT ACLs in the `user.NTACL` extended
attribute, so the volume's file system must support user extended
attributes: ext4, xfs and btrfs (and the engine's named volumes on them)
work; NFS volumes are not supported.

## Time

A container shares the host's kernel clock and the images never ask for
`CAP_SYS_TIME`, so:

- Keep the host's clock synchronized (chrony, systemd-timesyncd or the
  hypervisor's clock). Kerberos fails beyond five minutes of skew.
- The DC's chronyd runs with `-x`: it never adjusts the clock, and serves
  the host's time on 123/udp, signed through Samba's `ntp_signd` socket,
  for Windows members that use the DC as their time source.
- `sc-dc-init` reads the kernel's synchronization status: an unsynchronized
  clock refuses `provision` and `join` (unless `SC_ALLOW_UNSYNCED_CLOCK=1`)
  and is a warning in the health output while the DC runs.
- `SC_NTP=off` turns the DC's time service off, for sites with another
  signed time source for their Windows members.

## Permissions and security

| Container | User | Capabilities | Root file system |
|---|---|---|---|
| DC | root | `CHOWN`, `DAC_OVERRIDE`, `DAC_READ_SEARCH`, `FOWNER`, `FSETID` (ownership and ACLs of sysvol and state), `SETUID`, `SETGID` (Samba's and chronyd's privilege changes), `KILL` (signals to its children), `NET_BIND_SERVICE` (ports below 1024) | read-only |
| conductor, conductor-idp, conductor-sync, conductor-backup | 2093, 2095, 2094, 2097 | none | read-only |
| setup one-shots | root | `CHOWN`, `DAC_OVERRIDE`, `FOWNER`, `SETUID`, `SETGID` | read-only |

Every container runs with `no-new-privileges`, never in privileged mode,
and the DC never gets `CAP_SYS_ADMIN` or `CAP_SYS_TIME`. The user IDs are
fixed because the services check each other's UID on their sockets
(`SO_PEERCRED`): they must be the same numbers in every container, so do
not give the containers separate user namespaces.

NT ACLs: writing the `security.NTACL` attribute that Samba uses by default
needs `CAP_SYS_ADMIN`, so the DC image keeps NT ACLs in `user.NTACL`. The
consequence: anything that can write the volume's files as root on the
host can change sysvol ACLs. Do not share the DC's volumes with anything
else.

## Secrets

Passwords and keys reach the containers only as files, never as
environment variables, command lines, image layers or compose files. The
DC refuses to start when a variable that looks like a secret
(`*PASSWORD*`, `*SECRET*`, `*_KEY`) is set, and names the secret file to
use instead.

| Secret (`compose/secrets/`) | Used by | Created by |
|---|---|---|
| `admin-password` | the DC (first boot), the setup one-shots | `make-secrets.sh` |
| `idp-ad-password`, `sync-ad-password`, `backup-account` | the service accounts of the IdP, sync and backups | `make-secrets.sh` |
| `join-password` | an additional DC (first boot) | you |
| `s3` | backups and restores to S3-compatible storage | you |
| `age-identity` | a restore (first boot) | you (an operator's offline key) |
| `dc-tls-cert`, `dc-tls-key`, `dc-tls-ca`, `conductor-tls-cert`, `conductor-tls-key`, `idp-tls-cert`, `idp-tls-key` | provided TLS | your CA |

Docker Compose mounts file secrets without honouring owner or mode, so the
setup one-shots copy each credential the services need into a
per-service volume (`conductor-cred`, `idp-cred`, `sync-cred`,
`backup-cred`), mode 0400 and owned by the service's UID, mounted
read-only at `/run/credentials/<service>`; the services read them through
`CREDENTIALS_DIRECTORY`, exactly as with systemd's `LoadCredential=` in the
packages.

After the first boot, remove `admin-password`, `join-password` and
`age-identity` from the stack; the DC logs a warning while they stay
mounted. Without `admin-password` the DC generates the Administrator
password, keeps it on the volume (0400) and logs only where it is:
`docker compose exec dc sc-dc-init show-initial-password` prints it and
`forget-initial-password` deletes it.

## Additional DCs and Windows clients

`addons/join.yaml` joins this host's DC to an existing domain
(`SC_MODE=join`, host networking): set `SC_JOIN_DC` to an existing DC's
address and put the join account's password in `secrets/join-password`.
Before joining, the DC checks the clock against the target and that its
host name is not already a DC object of the domain. After the join, check
replication with `samba-tool drs showrepl` on both sides.

Samba does not replicate SYSVOL. Edit Group Policy on one DC, copy its
sysvol to the others, then run `samba-tool ntacl sysvolreset` there.

Windows clients and member servers join as with any Samba DC; they must
use the DC (or a resolver that forwards the domain to it) as their DNS
server and take their time from it.

To move an existing Samba domain controller (installed from packages, or
run from the images of the earlier, archived Samba Conductor) into the DC
image, join a new container
DC to the domain, move the roles to it, then demote the old DC. Adopting
an existing Samba data directory is not supported.

## Backups and restore

With `addons/backup.yaml`, backups work as with the packages:
conductor-helper in the DC container takes an online backup over loopback
with an account that holds only the replication rights, encrypts it with
age to your offline recipients and signs its manifest; the
`samba-conductor-backup` container (no capabilities) uploads it to S3 or a
local volume, verifies it, applies retention and sends alerts, in place of
the systemd timer and path unit. The archive also holds conductor's
database. Restore drills run on a drill host installed from the packages,
never in these containers
([conductor-backup](https://github.com/openbasalt/samba-conductor-backup/blob/main/README.md)).

conductor-idp's and conductor-sync's state is not in the archive: back up
their volumes (`idp-*`, `sync-*`) with the containers stopped, for example
with `tar` from a throwaway container.

Two rules:

- A copy of `dc-data` taken while the DC runs is not a backup.
- Never restore a volume snapshot of a DC into a domain that has other
  DCs: it causes USN rollback and silently breaks replication. A copy of
  stopped volumes is only acceptable for a single-DC lab.

Full-forest recovery: `addons/restore.yaml` (`SC_MODE=restore`) restores
the latest (or a chosen) backup into a new DC container under a new host
name. It requires `SC_RESTORE_CONFIRM=<REALM>`; never run it while another
DC of the domain is running. Afterwards remove `age-identity` from the
stack and follow the next steps the restore prints, as in the
[restore runbook](https://github.com/openbasalt/samba-conductor/blob/main/docs/restore.md).

## Upgrades

Take a backup run first. Then change `SC_TAG` (or the digest) and run
`docker compose pull` and `docker compose up -d`. In a domain with several
DCs, upgrade one DC at a time and check `samba-tool drs showrepl` between
them.

At every start the DC compares the image's Samba version with the version
that last ran the domain:

| Change | What happens |
|---|---|
| same Samba | starts |
| newer Samba patch release | starts, then runs a read-only `samba-tool dbcheck --cross-ncs`; the result is in the log and the health output |
| newer Samba minor release | refused unless `SC_ALLOW_SAMBA_UPGRADE=<new minor>` is set; then an offline copy of `private/` and `sysvol/` is written to `dc-data/pre-upgrade/` before the start |
| older Samba | refused: downgrades are not supported |

The DC image uses Debian stable's Samba. A Samba minor change is a new
`sambaX.Y` line and is never applied without the acknowledgement above,
even through `latest`.

conductor, conductor-idp and conductor-sync migrate their databases
forward when they start and refuse a database newer than themselves, so
rolling back an image after an upgrade fails at start instead of damaging
data: restore the volumes from before the upgrade to roll back.
conductor-helper (in the DC image) and conductor must come from the same
release; keep the DC and conductor images on the same tag.

## Podman, rootless and SELinux

| Engine | DC | Service images |
|---|---|---|
| Docker Engine (rootful) with Compose v2 | tested | tested |
| Podman rootful | expected to work, not in the test matrix | expected to work, not in the test matrix |
| Podman rootless | not recommended for a DC | expected to work |
| Docker rootless | not tested | not tested |

Rootless Podman:

- The DC needs ports below 1024: set
  `net.ipv4.ip_unprivileged_port_start=53` on the host. Publishing on high
  ports does not help, since clients need the standard ports.
- Use pasta networking (the default of Podman 5) so the DC sees the
  clients' source addresses.
- Run every container of the stack under the same user namespace mapping
  (one user, never `--userns=auto`); otherwise the UID checks on the
  sockets fail. A single pod is the simplest.
- The volume files belong to sub-UIDs of your user, which matters for
  backups of the volumes taken from the host.

SELinux (Fedora, Basalt OS and other SELinux hosts): containers that share
a socket on a volume must share one MCS level, or the policy refuses the
connection. The compose files set one level for the whole stack,
`SC_MCS_LEVEL` (default `s0:c93,c94`); two stacks on the same host need
two levels. Named volumes are labeled by the engine; bind mounts need `:z`
for the shared directories and `:Z` for private ones. On AppArmor hosts
the engine's default profile applies.

Docker's `userns-remap` is fine: it maps every container the same way.

## Troubleshooting

| Symptom | Likely cause |
|---|---|
| Kerberos errors, `clock skew too great` | the host's clock is not synchronized; see Time |
| DNS records name the container's address | `SC_HOST_IP` was not the host's address at provision time; in bridge mode records must carry `SC_HOST_IP` |
| the DC cannot bind port 53 | another resolver on the host listens on that address; publish on `SC_HOST_IP` only, or use host networking (Samba binds only `SC_HOST_IP` and loopback) |
| `permission denied` on a socket between containers, AVC denials | the containers do not share one MCS level (`SC_MCS_LEVEL`) |
| a service refuses a peer on its socket | the containers run under different user namespace mappings |
| the DC refuses to start | read the first log line: a different `SC_REALM` or `SC_HOSTNAME` than the volume's domain, an unfinished first boot (`SC_RETRY_FIRST_BOOT=1` starts it over), an unknown `SC_*` variable or a secret in the environment |

`docker compose exec dc sc-dc-init health` prints the DC's checks and
warnings.
