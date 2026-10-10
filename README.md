# Samba Conductor documentation

Samba Conductor is a web administrator and self-service portal for Samba
Active Directory, written in Go. Administrators manage users, groups, OUs,
computers, DNS, Group Policy links, password policies and lockouts from a
browser; every user of the domain gets a self-service portal for their
profile, password and second factors. Every directory operation runs with
the signed-in user's own identity, so the domain's own access rules apply,
and every change is previewed before it is applied and recorded in an
append-only audit log (see [architecture](architecture.md) for what its
hash chain does and does not protect against).

Optional components, each deployed on its own, extend it:

- an OpenID Connect and SAML 2.0 identity provider backed by the domain;
- provisioning from the domain to Google Workspace;
- encrypted domain backups, restore tooling and automated restore drills;
- an agent that manages file shares on domain-member file servers.

The components run as native systemd services on Debian, Ubuntu, Fedora and
Basalt OS, each under its own user and sandbox, or as hardened container
images ([containers](containers.md)).

This repository holds the documentation that spans the components. Each
component repository documents its own installation, configuration and
design.

## Repositories

| Repository | Contents |
|---|---|
| [samba-conductor](https://github.com/openbasalt/samba-conductor) | conductor: web administration, self-service and conductor-helper, the narrow privileged helper |
| [samba-conductor-idp](https://github.com/openbasalt/samba-conductor-idp) | conductor-idp: OpenID Connect and SAML 2.0 identity provider backed by AD |
| [samba-conductor-sync](https://github.com/openbasalt/samba-conductor-sync) | conductor-sync: provisioning from AD to Google Workspace |
| [samba-conductor-backup](https://github.com/openbasalt/samba-conductor-backup) | conductor-backup: encrypted domain backups, restore and restore drills |
| [samba-conductor-files](https://github.com/openbasalt/samba-conductor-files) | conductor-files: agent for shares and NT ACLs on domain-member file servers |
| [samba-conductor-ad](https://github.com/openbasalt/samba-conductor-ad) | the `ad` Go library for Samba AD access, shared by the components |
| [samba-conductor-docs](https://github.com/openbasalt/samba-conductor-docs) | this repository: architecture, packaging, release verification, testing |

## Documentation

In this repository:

- [Architecture](architecture.md): goals, platforms, components, how AD
  is accessed, the security model, the integration components and the
  `ad` library.
- [Packaging](packaging.md): how the .deb and .rpm packages and the
  SELinux policy packages are built, what they contain, versions,
  reproducible builds, SBOMs and licenses.
- [Container images](containers.md): the images on Docker Hub and GHCR,
  compose files, networking, persistence, time, permissions, secrets,
  backups, upgrades, Podman and SELinux, and how to verify an image.
- [Verifying releases](verifying-releases.md): the release key, what it
  signs and how to check a download or a repository.
- [Testing](testing.md): unit tests and gates, the integration lab, the
  package lab and restore exercises.

In the component repositories:

- conductor: [install on Debian and Ubuntu](https://github.com/openbasalt/samba-conductor/blob/main/docs/install.md),
  [install on Basalt OS and Fedora](https://github.com/openbasalt/samba-conductor/blob/main/docs/install-fedora.md),
  [configuration reference](https://github.com/openbasalt/samba-conductor/blob/main/docs/config.md),
  [restore runbook](https://github.com/openbasalt/samba-conductor/blob/main/docs/restore.md),
  [design](https://github.com/openbasalt/samba-conductor/blob/main/docs/design.md).
- conductor-idp: [install](https://github.com/openbasalt/samba-conductor-idp/blob/main/docs/install.md),
  [install on Basalt OS and Fedora](https://github.com/openbasalt/samba-conductor-idp/blob/main/docs/install-fedora.md),
  [design](https://github.com/openbasalt/samba-conductor-idp/blob/main/docs/design.md).
- conductor-sync: [README and commands](https://github.com/openbasalt/samba-conductor-sync/blob/main/README.md),
  [mapping reference](https://github.com/openbasalt/samba-conductor-sync/blob/main/docs/mapping.md),
  [install on Basalt OS and Fedora](https://github.com/openbasalt/samba-conductor-sync/blob/main/docs/install-fedora.md),
  [design](https://github.com/openbasalt/samba-conductor-sync/blob/main/docs/design.md).
- conductor-backup: [README and install](https://github.com/openbasalt/samba-conductor-backup/blob/main/README.md),
  [install on Basalt OS and Fedora](https://github.com/openbasalt/samba-conductor-backup/blob/main/docs/install-fedora.md),
  [design](https://github.com/openbasalt/samba-conductor-backup/blob/main/docs/design.md).
- conductor-files: [install](https://github.com/openbasalt/samba-conductor-files/blob/main/docs/install.md),
  [install on Basalt OS and Fedora](https://github.com/openbasalt/samba-conductor-files/blob/main/docs/install-fedora.md),
  [design](https://github.com/openbasalt/samba-conductor-files/blob/main/docs/design.md).
- ad: [README and usage](https://github.com/openbasalt/samba-conductor-ad/blob/main/README.md),
  [design](https://github.com/openbasalt/samba-conductor-ad/blob/main/docs/design.md).

## Status

Released: conductor and conductor-idp 0.1.1, conductor-sync and
conductor-backup 0.1.0. Each has a signed GitHub release (tag `v0.1.1` or
`v0.1.0`) and Debian and Ubuntu packages (`0.1.1-1` or `0.1.0-1`) in the APT
repository at <https://obpkg.org/apt>. The container images of containers
release 0.1.1 carry these versions and are tagged `0.1.1` and `latest` on Docker Hub
(`docker.io/openbasalt`) and GHCR (`ghcr.io/openbasalt`), with the same
digests in both ([container images](containers.md)). Packages for Fedora
and Basalt OS are published in the basalt-tools repository at
<https://obpkg.org/basalt-tools> (see the [install guide](https://github.com/openbasalt/samba-conductor/blob/main/docs/install.md)).
conductor-files and the ad library have no tagged release yet.

## Contributing and security

Documentation changes: see [CONTRIBUTING.md](CONTRIBUTING.md). Security
problems: see [SECURITY.md](SECURITY.md); do not open a public issue.

## License

Apache License, Version 2.0 ([LICENSE](LICENSE), [NOTICE](NOTICE)), like
every Samba Conductor repository.
