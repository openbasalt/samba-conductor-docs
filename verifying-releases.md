# Verifying releases

Samba Conductor is signed with the OpenBasalt release key, the single
OpenPGP identity of the OpenBasalt organization. It has one signing subkey
per use; Samba Conductor uses the packages subkey.

## The release key

| | |
|---|---|
| User ID | `OpenBasalt release key <openbasalt@openbasalt.org>` |
| Primary key (certify only, RSA 4096) | `3601 7348 42BD 4E48 2D19  DE4A E4EE D5EC A395 B302` |
| Packages signing subkey (RSA 4096) | `3024 61D2 6520 E077 D07F  FCA9 AA27 C62C 36CC FC4B` |
| Public key | <https://obpkg.org/keys/openbasalt-release-key.asc> |

The primary key is kept offline and only certifies subkeys. Check the
fingerprints above against the key you download before you trust it; they
are also published in each component repository's SECURITY.md.

## What signs what

| Artifact | Signature | Key |
|---|---|---|
| `SHA256SUMS` of a release (covers every .deb, .rpm, policy package and SBOM) | detached OpenPGP signature, `SHA256SUMS.asc` | packages subkey |
| Samba Conductor's APT repository metadata (`InRelease`, `Release.gpg`) | OpenPGP | packages subkey |
| RPMs of the `basalt-tools` repository | OpenPGP signature in each package (`rpmsign`) | packages subkey |
| Repository metadata of `basalt-tools` (`repomd.xml.asc`) | OpenPGP | packages subkey |
| Git tags | OpenPGP or SSH signature | the key of the person who tags |

Packages are not signed by the build itself: signing is a release-time
step. Individual .deb files are not signed; apt verifies them through the
signed `InRelease`, which lists their hashes. RPMs are signed when they
are published to `basalt-tools`, where dnf checks both the package
signatures and the repository metadata.

## Get and check the key

```sh
curl -fsSLO https://obpkg.org/keys/openbasalt-release-key.asc
gpg --show-keys --with-subkey-fingerprints openbasalt-release-key.asc
```

The output must show the primary key
`3601734842BD4E482D19DE4AE4EED5ECA395B302` and the subkey
`302461D26520E077D07FFCA9AA27C62C36CCFC4B`. Then import it:

```sh
gpg --import openbasalt-release-key.asc
```

## Verify downloaded release files

Download the packages you need together with `SHA256SUMS` and
`SHA256SUMS.asc` from the same release, then:

```sh
gpg --verify SHA256SUMS.asc SHA256SUMS
sha256sum -c --ignore-missing SHA256SUMS
```

`gpg --verify` must report a good signature from the OpenBasalt release
key (made by the packages subkey); `sha256sum` must report `OK` for every
file you downloaded. Only then install them, for example
`sudo apt install ./conductor_<version>-1_amd64.deb` or
`sudo dnf install ./conductor-<version>-1.x86_64.rpm`.

## Verify RPMs

To check RPM signatures by hand, trust the key in rpm, then check each
file:

```sh
sudo rpm --import openbasalt-release-key.asc
rpm -K conductor-*.rpm
```

`rpm -K` must report `digests signatures OK`. RPMs taken from a release's
assets (rather than from `basalt-tools`) carry no embedded signature;
check them with `SHA256SUMS` as above.

## The basalt-tools repository (Basalt OS, Fedora 44)

On Basalt OS the `basalt-tools` repository is configured by default and
its key is installed by `basalt-release` as
`/etc/pki/rpm-gpg/RPM-GPG-KEY-basalt`. On Fedora 44, add it with a file
`/etc/yum.repos.d/basalt-tools.repo`:

```ini
[basalt-tools]
name=Basalt OS tools $releasever - $basearch
baseurl=https://obpkg.org/basalt-tools/$releasever/$basearch/
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://obpkg.org/keys/openbasalt-release-key.asc
```

On Basalt OS the same definition uses
`gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-basalt`.

`gpgcheck=1` makes dnf refuse a package whose signature does not verify;
`repo_gpgcheck=1` makes it refuse repository metadata whose signature does
not verify. The first time dnf uses the key it shows the fingerprint and
asks to import it: compare it with the primary key above. Then:

```sh
sudo dnf install conductor
```

The `<pkg>-selinux` policy package comes with each component wherever the
targeted policy is installed.

## The APT repository (Debian, Ubuntu)

The APT repository is signed with the same packages subkey (`InRelease`,
`Release.gpg`), and apt checks it against the key named in the source's
`Signed-By=`. It is not published yet; until it is, install the .deb
files of a release after checking them with `SHA256SUMS` as above. Each
component's install document shows the APT source once it is available.

## Key rotation and compromise

Subkeys expire and are extended or replaced with the offline primary key;
the updated public key is published at the same URL well before the old
expiry, and a new subkey is used only once clients can have it. A
compromised subkey is revoked with the primary key and the repositories
are re-signed with a new subkey; the fingerprint of the primary key stays
the same, so the trust you placed in it carries over.
