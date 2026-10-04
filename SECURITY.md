# Security policy

Samba Conductor manages Active Directory domains, so a vulnerability in it
can mean control over a whole domain. Please report security problems
privately and give us time to fix them before they are disclosed.

## Reporting a vulnerability

This repository holds documentation only. Report a problem in the
component it concerns, through that repository's private vulnerability
reporting (the "Report a vulnerability" button under the Security tab);
each component's SECURITY.md describes what to include:

| Repository | Component |
|---|---|
| [samba-conductor](https://github.com/openbasalt/samba-conductor/blob/main/SECURITY.md) | conductor: web administration, self-service, conductor-helper |
| [samba-conductor-idp](https://github.com/openbasalt/samba-conductor-idp/blob/main/SECURITY.md) | conductor-idp: OpenID Connect and SAML identity provider |
| [samba-conductor-sync](https://github.com/openbasalt/samba-conductor-sync/blob/main/SECURITY.md) | conductor-sync: provisioning to Google Workspace |
| [samba-conductor-backup](https://github.com/openbasalt/samba-conductor-backup/blob/main/SECURITY.md) | conductor-backup: encrypted backups and restore drills |
| [samba-conductor-files](https://github.com/openbasalt/samba-conductor-files/blob/main/SECURITY.md) | conductor-files: agent on domain-member file servers |
| [samba-conductor-ad](https://github.com/openbasalt/samba-conductor-ad/blob/main/SECURITY.md) | the `ad` Go library shared by the components |

If you are not sure which component is affected, or the problem is in
this documentation (for example an instruction that would leave a system
insecure), report it to conductor:
<https://github.com/openbasalt/samba-conductor/security/advisories/new>.
We move reports between repositories.

Do not open a public issue, pull request or discussion for a security
problem.

## Verifying releases

How to check that a package or repository is signed by the project:
[verifying-releases.md](verifying-releases.md).
