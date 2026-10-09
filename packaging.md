# Packaging

How Samba Conductor's packages are built and what they contain. Every
component repository has a `packaging/` directory with the same build
script, and `make package` builds every package of that component.

## Packages

| Package | Contents | System user | Configuration files |
|---|---|---|---|
| `conductor` | `conductor`, `conductor-helper`, their units | `conductor` | `/etc/conductor/helper.toml` |
| `conductor-idp` | `conductor-idp`, its unit | `conductor-idp` | `/etc/conductor-idp/idp.toml` |
| `conductor-sync` | `conductor-sync`, service, timer, management API service and socket | `conductor-sync` | `/etc/conductor-sync/conductor-sync.toml` |
| `conductor-backup` | `conductor-backup`, DC units (service, timer, path), drill units, the helper drop-in (inactive) | `conductor-backup` | `/etc/conductor-backup/conductor-backup.toml`, `drill.toml` |
| `conductor-files` | `conductor-files`, its unit | none (root, narrow unit) | `/etc/conductor-files/agent.toml` |

Each package is built as a .deb and as an .rpm. On RPM, each component
also has a `<pkg>-selinux` package with its SELinux policy module
([SELinux policy packages](#selinux-policy-packages)).

conductor-helper ships inside the `conductor` package: it is built from
the same module, speaks a protocol versioned with conductor and needs the
`conductor` user, so one package avoids version skew. `conductor-backup`
only suggests `conductor`, because a drill host must not pull conductor
in.

The shipped configuration files are safe defaults that start nothing by
themselves: backups off in `helper.toml`, `mode = "dry-run"` in
`conductor-sync.toml`. `/etc/conductor/conductor.toml` is not a packaged
file: `conductor setup` writes it and refuses to overwrite it.

## One configuration, two formats

Packages are built with [nfpm](https://nfpm.goreleaser.com/) from one
configuration per component (`packaging/nfpm.yaml`), not with
`dpkg-buildpackage` or `rpmbuild`. The binaries are static Go builds that
are identical for every distribution, so a Debian build chroot or an RPM
spec would add nothing; nfpm builds both formats and both architectures
from one file and honours `SOURCE_DATE_EPOCH`. Entries that exist in one
format only are marked `packager: deb` or `packager: rpm`. The results are
still checked with each distribution's linter: lintian (Debian 13) and
`rpmlint --strict` (Fedora 44), with every override or filter justified in
the override file itself (`packaging/lintian-overrides`,
`packaging/rpmlintrc`).

## Binaries

- Static Go binaries, `CGO_ENABLED=0`, built with
  `-trimpath -ldflags "-s -w -buildid="` and the version stamped in.
- One build per architecture: amd64 and arm64 (x86_64 and aarch64 in RPM
  names). arm64 is cross-compiled; no arm64 build host is needed.
- The bytes inside the .deb and the .rpm are the same, and one .deb serves
  Debian 13, Ubuntu 26.04 and Ubuntu 24.04.
- The Go toolchain is the one `go.mod` names; with `GOTOOLCHAIN=auto` the
  go command fetches it.

## Reproducible builds

A rebuild from the same commit produces byte-identical packages and SBOMs:

- `SOURCE_DATE_EPOCH` is the commit time; nfpm, gzip (`gzip -n`), the RPM
  build time and the changelog dates use it;
- `-trimpath` and an empty build ID remove paths and random IDs from the
  binaries;
- the RPM header's build host is fixed and the payload is zstd;
- the SBOM's timestamp and serial number are derived from the package;
- nfpm, cyclonedx-gomod and the SELinux toolchain are pinned (the Fedora
  container that compiles policy modules is pinned by digest).

Builds are checked by building twice and comparing SHA-256 sums.

## Versions

Each repository is versioned and tagged on its own.

| Git | .deb version | .rpm version |
|---|---|---|
| tag `v1.2.3` on HEAD | `1.2.3-1` | `1.2.3-1` |
| tag `v1.2.3-rc.1` on HEAD | `1.2.3~rc.1-1` | `1.2.3~rc.1-1` |
| commits after `v1.2.3` | `1.2.3+git<time>.<sha12>-1` | `1.2.3^git<time>.<sha12>-1` |
| no tag yet | `0.0.0+git<time>.<sha12>-1` | `0.0.0^git<time>.<sha12>-1` |

`<time>` is the UTC commit time and `<sha12>` the abbreviated commit. A
release candidate sorts before its release (`~`); a snapshot sorts after
the last tag (`+` in Debian, `^` in RPM, following Fedora's versioning
guidelines). Builds from a tree with uncommitted changes get a `.dirty`
suffix and are never released. RPMs carry release `1` and no distribution
tag: the same packages serve every Fedora-based release the policy
supports. The binaries report the same upstream version
(`<binary> version`).

## Contents

Every package ships:

- binaries in `/usr/bin`;
- systemd units in `/usr/lib/systemd/system`: the repository's
  `deploy/systemd/` files with `/usr/local/bin` rewritten to `/usr/bin`,
  nothing else changed, hardening included;
- a `sysusers.d` entry for the system user (every package except
  conductor-files, which has none);
- one man page per binary, section 8;
- example configuration in `/usr/share/doc/<pkg>/examples`;
- license files (below).

| | .deb | .rpm |
|---|---|---|
| File name | `<pkg>_<version>-1_amd64.deb` | `<pkg>-<version>-1.x86_64.rpm`, `<pkg>-selinux-<version>-1.noarch.rpm` |
| Dependencies | `systemd`; Recommends the Samba packages the component uses | `systemd`, `(<pkg>-selinux = <version>-<release> if selinux-policy-targeted)`; Recommends the Samba packages |
| System user | postinst runs `systemd-sysusers` | `%pre` runs `systemd-sysusers` with the same line, so the payload can be owned by the user |
| Directories and modes | set by postinst; shipped paths get a `dpkg-statoverride`, so an administrator's own override wins | owned by the package with their final owner and mode; rpm restores them on upgrade |
| Configuration | conffiles | `%config(noreplace)`: an edited file is kept and the new one lands as `.rpmnew` |
| Licenses | `copyright` (DEP-5), `NOTICE`, `THIRD-PARTY-LICENSES.gz` in `/usr/share/doc/<pkg>` | `LICENSE`, `NOTICE`, `THIRD-PARTY-LICENSES` in `/usr/share/licenses/<pkg>` |
| Documentation | `README.Debian`, `changelog.Debian.gz` | `README.Fedora`, the RPM `%changelog` |

### Directories and secrets

Each component owns `/etc/<pkg>` (0750, root and the component's group),
a `credentials/` directory (0700, root) and `/var/lib/<pkg>` (0700, the
component's user). Secrets are files the administrator or a setup command
creates in `credentials/`; packages never ship or generate them. systemd
hands them to the service with `LoadCredential=`; the service reads
`$CREDENTIALS_DIRECTORY/<name>`, and the credentials directory is
inaccessible inside the sandbox. A credential can optionally be encrypted
with the host key or a TPM (`systemd-creds encrypt`) and loaded with
`LoadCredentialEncrypted=` from a drop-in; the service code does not
change.

conductor-backup's drop-in for conductor-helper ships inactive in
`/usr/share/conductor-backup/systemd/`: active, its `LoadCredential=`
would stop conductor-helper from starting until the backup account's
credential exists. Enabling backups on a DC links it.

## systemd units and sandboxing

Every long-running program is a systemd service with its own user (except
conductor-files, which runs as root with a narrow capability set),
`NoNewPrivileges=yes`, `ProtectSystem=strict`, `ProtectHome`,
`PrivateTmp`, `PrivateDevices`, `RestrictAddressFamilies`,
`RestrictNamespaces`, `SystemCallFilter=@system-service`, kernel, clock
and control group protections, `StateDirectory=` with mode 0700 and
`UMask=0077`. The capability bounding set is empty for conductor,
conductor-idp, conductor-sync and conductor-backup; conductor-helper keeps
CHOWN, FOWNER, DAC_OVERRIDE and DAC_READ_SEARCH and may reach only
localhost; conductor-files keeps the same plus SYS_ADMIN (for Samba's
`security.NTACL` attribute). Scheduled work uses timers and path units;
conductor-sync's management API uses socket activation.

Installation and upgrades follow the same rules in both formats:

| Event | What happens |
|---|---|
| install | the system user and directories are created and systemd reloaded. Nothing is enabled or started: every component needs configuration and secrets first, and a half-configured identity or backup service must not run |
| upgrade | running long-lived services (conductor, conductor-helper, conductor-idp, conductor-sync-api, conductor-files) are restarted; timers, sockets and paths keep their state; enabled units stay enabled |
| remove | every unit is stopped and disabled; configuration and state stay |
| purge (.deb) | `/etc/<pkg>`, `/var/lib/<pkg>`, drop-ins of the package's units and statoverrides are removed |

Never removed by a package: system users (files they own may exist
elsewhere), backups (bucket or local destination), and shares and their
folders on file servers.

The .deb maintainer scripts mirror what
`dh_installsystemd --no-enable --no-start --restart-after-upgrade`
generates. On Debian and Ubuntu no AppArmor profile is shipped: each
unit's systemd sandbox is the confinement.

## RPM packages for Fedora and Basalt OS

The RPMs carry the same binaries, units, man pages, `sysusers.d` entries
and configuration as the .debs, and follow Fedora's conventions where they
differ:

- scriptlets are what Fedora's systemd macros expand to
  (`systemd-update-helper`, with a `systemctl` fallback): on install the
  distribution's preset decides, and Fedora's and Basalt OS's default
  preset disables the units. The packages ship no preset file, so nothing
  is enabled or started;
- on upgrade the long-running services that run are restarted at the end
  of the transaction;
- on removal every unit is disabled and stopped; rpm removes packaged
  files and empty directories. What the administrator created
  (`conductor.toml`, credentials, TLS files, state, local backups, shares)
  is kept, and edited configuration is kept as `.rpmsave`. There is no
  purge: delete kept data by hand;
- licenses are installed with `%license` in `/usr/share/licenses/<pkg>`.

Fedora and Basalt OS build the Samba AD DC with MIT Kerberos (Debian and
Ubuntu with Heimdal). The units order themselves after both distributions'
Samba units (`samba-ad-dc.service` and `samba.service` for a DC,
`smbd.service` and `smb.service` for a file server), and paths that only
exist on one distribution (such as `/var/cache/samba`) are optional in the
sandbox.

## SELinux policy packages

Each component has a SELinux policy module for the targeted policy, in
`packaging/selinux/` of its repository (`.te`, `.fc`, `.if`, written with
the reference policy's macros), shipped as `<pkg>-selinux`, a noarch RPM.

| Module | Domains |
|---|---|
| `conductor` | `conductor_t` (conductor), `conductor_helper_t` (conductor-helper) |
| `conductor_idp` | `conductor_idp_t` |
| `conductor_sync` | `conductor_sync_t` (scheduled runs and the management API) |
| `conductor_backup` | `conductor_backup_t` (DC backups), `conductor_backup_drill_t` (restore drills) |
| `conductor_files`, `conductor_files_port` | `conductor_files_t`, the agent's TCP port type |

Each module also defines types for the component's configuration, its
credentials directory, its state and its sockets.

- The main package requires `(<pkg>-selinux = <version>-<release> if
  selinux-policy-targeted)`: wherever the targeted policy is installed,
  the policy package is installed first and upgraded together with the
  program.
- The policy mirrors the units' systemd sandbox: the same paths, sockets
  and ports, and nothing the sandbox forbids. Each credentials directory
  has its own type that only systemd reads; the services read their copy
  in `$CREDENTIALS_DIRECTORY`.
- Root work is bounded by its domain: `conductor_helper_t` may run
  `samba-tool` as root against the local Samba database and write
  conductor-backup's spool; `conductor_files_t` runs Samba's tools in its
  own domain, so the agent's rights bound theirs.
- Cross-component rules live in the module of the component that needs
  them, in `optional_policy` blocks, so each package installs on its own.
- Restore drills run in `conductor_backup_drill_t`, which is unconfined:
  a drill builds namespaces and mounts and runs Samba inside its
  network-less sandbox, on a dedicated host.
- The modules are compiled with `selinux-policy-devel` in a Fedora
  container pinned by digest, with the policy version pinned; the
  compiled module is byte-reproducible. The package requires
  `selinux-policy-targeted` at least as new as the policy it was compiled
  against.
- The scriptlets are Fedora's `%selinux_relabel_pre`,
  `%selinux_modules_install` (priority 200), `%selinux_modules_uninstall`
  and `%selinux_relabel_post`. On removal the remaining files of the
  component are relabeled to the system defaults.
- The policy is tested with SELinux enforcing ([testing](testing.md)):
  every rule answers a denial found in a real run, no blanket
  `audit2allow`.

### License of the policy packages

The module sources are Apache-2.0, like the rest of each repository. A
compiled module also contains expansions of the reference policy's macros
from `selinux-policy-devel`, which is GPL-2.0-or-later. The policy
packages are therefore licensed `Apache-2.0 AND GPL-2.0-or-later`, and each
ships the reference policy's `COPYING` in its `THIRD-PARTY-LICENSES`
(`/usr/share/licenses/<pkg>-selinux/`).

The corresponding source of a policy package is:

- the component repository at the release commit (`packaging/selinux/`
  and its build script), and
- the Fedora `selinux-policy` source package of the version the module was
  compiled against (named in the package's `THIRD-PARTY-LICENSES`
  file; the package also requires `selinux-policy-targeted` at least that
  new).

## Licenses inside the packages

- The package license is `Apache-2.0`. The .deb has a DEP-5 `copyright`
  file pointing at `/usr/share/common-licenses/Apache-2.0`; the RPM ships
  `LICENSE`. Both ship the repository's `NOTICE`.
- `THIRD-PARTY-LICENSES` holds the license texts of everything linked into
  the binaries. `packaging/third-party-licenses.py` reads the module list
  from the built binaries' own build information (`go version -m`, so
  exactly what was linked) and copies each module's LICENSE, COPYING,
  NOTICE and PATENTS files from the module cache, the Go standard
  library's from GOROOT, sorted and without timestamps.
- The build fails on a module without a license file, and on a module
  with a NOTICE file that the repository's `NOTICE` does not name
  (Apache-2.0, section 4(d)).

## SBOMs

Every package has a CycloneDX 1.6 SBOM (`<package>.cdx.json`), generated
with `cyclonedx-gomod app -licenses -std` for each binary and merged per
package: modules, versions, hashes, detected licenses and the Go standard
library. The package itself is the SBOM's subject with license
`Apache-2.0` and a package URL (`pkg:deb/openbasalt/<pkg>@<version>` or
`pkg:rpm/openbasalt/<pkg>@<version>`, with the architecture). Licenses the
tool does not detect are set explicitly. Each policy package has its own
SBOM, recording the policy toolchain. Release SBOMs carry the real hashes
of the sibling Samba Conductor modules.

## Module paths and sibling pins

Each component is a Go module whose path is its repository path:

| Module | Imports |
|---|---|
| `github.com/openbasalt/samba-conductor-ad` | |
| `github.com/openbasalt/samba-conductor` | `ad`, `syncapi` (from conductor-sync), `filesapi` (from conductor-files) |
| `github.com/openbasalt/samba-conductor-idp` | `ad` |
| `github.com/openbasalt/samba-conductor-sync` | `ad` |
| `github.com/openbasalt/samba-conductor-backup` | `ad` |
| `github.com/openbasalt/samba-conductor-files` | |

Because the module path is the repository path, `go get` and
`go install` resolve every module directly. There are no `replace`
directives: each `go.mod` pins its sibling modules by commit
(pseudo-versions such as `v0.0.0-<time>-<sha12>`), and `go.sum` holds
their hashes. Every repository therefore builds on its own: the Makefiles
and the packaging script set `GOWORK=off`, so a package is always built
against the pinned siblings, which is also what a release SBOM records.

To work across repositories, clone them side by side and use a Go
workspace; `make check GOWORK=<path to go.work>` checks a repository
against the local copies. A pin must name a commit that is on GitHub:
push the sibling first, then update the consumer with
`GOWORK=off go get <module>@<commit>` and `GOWORK=off go mod tidy`. Each
repository's CONTRIBUTING.md describes the workflow.

## Building

In a component repository:

```sh
make package          # dist/: .deb (amd64, arm64), .rpm (x86_64, aarch64),
                      # <pkg>-selinux .noarch.rpm, one .cdx.json per package
make package ARCHES=arm64 FORMATS=deb
make lintian          # Debian 13's lintian on the .deb files
make rpmlint          # Fedora 44's rpmlint --strict on the .rpm files
```

`make package` first compiles the SELinux modules
(`packaging/selinux/build.sh`, in a Fedora container run with docker or
podman, or with a local `selinux-policy-devel` when `SELINUX_LOCAL=1`),
then runs `packaging/build.sh`: static binaries, units, man pages and
changelogs staged, nfpm, third-party licenses and SBOMs. A release also
produces a `SHA256SUMS` file over every package and SBOM, signed with the
release key ([verifying releases](verifying-releases.md)).

The Debian and Ubuntu packages are published in an APT repository with one
suite for every supported distribution (`stable`, plus `testing` for
release candidates), component `main`, architectures amd64 and arm64: the
packages are identical across distributions, so per-codename suites would
only multiply metadata. The RPMs are published in Basalt OS's
`basalt-tools` repository.

## Container images

The container images are built from these packages, not from a separate
compile: the image build downloads the released .deb files from the APT
repository, verifies them through its signed `InRelease` and extracts the
binaries, so an image and a package of the same version contain the same
binaries. How to run and verify them: [container images](containers.md).
