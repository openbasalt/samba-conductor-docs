# Contributing to the Samba Conductor documentation

Thank you for helping. This repository holds the documentation that spans
the Samba Conductor components: architecture, packaging, release
verification and testing. Each component documents its own installation,
configuration and design in its own repository:

| Repository | Component |
|---|---|
| [samba-conductor](https://github.com/openbasalt/samba-conductor) | conductor: web administration, self-service, conductor-helper |
| [samba-conductor-idp](https://github.com/openbasalt/samba-conductor-idp) | conductor-idp: OpenID Connect and SAML identity provider |
| [samba-conductor-sync](https://github.com/openbasalt/samba-conductor-sync) | conductor-sync: provisioning to Google Workspace |
| [samba-conductor-backup](https://github.com/openbasalt/samba-conductor-backup) | conductor-backup: encrypted backups and restore drills |
| [samba-conductor-files](https://github.com/openbasalt/samba-conductor-files) | conductor-files: agent on domain-member file servers |
| [samba-conductor-ad](https://github.com/openbasalt/samba-conductor-ad) | the `ad` Go library shared by the components |

A change to how a component behaves, or to its own documents, belongs in
that component's repository; follow its CONTRIBUTING.md.

Security problems: do not open an issue, follow [SECURITY.md](SECURITY.md).

## Proposing a change

- Typos, broken links and small corrections: open a pull request
  directly.
- A document that is wrong about how a component behaves: open an issue
  here or in the component's repository, with what the code or a test run
  shows.
- New documents or a new structure: open an issue first to agree on it.

## Style

- English, plain Markdown, lines wrapped near 76 columns.
- Describe the system as it is. Plans and ideas go into issues.
- Relative links between files of this repository; absolute GitHub links
  (`https://github.com/openbasalt/<repository>/blob/main/<path>`) to the
  component repositories.
- No secrets, real host names, addresses or customer data in examples: use
  the reserved `.test` and `example.com` names and documentation addresses
  (192.0.2.0/24).
- Commit messages: a short summary line, then what changed and why.

By submitting a contribution you agree that it is licensed under the
Apache License, Version 2.0, this repository's license (see
[LICENSE](LICENSE) and [NOTICE](NOTICE)), as section 5 of that license
provides.

## Code of conduct

Be respectful and constructive. The OpenBasalt code of conduct applies to
every repository of the organization.
