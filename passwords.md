# Passwords by e-mail

Invitations, password reset by e-mail and password-change notices are
features of conductor, but no single component makes them work. This page
describes how they are split between conductor, conductor-provisioner and
the mail relay, and what each part refuses on its own. The pages and
settings are described in conductor's
[passwords.md](https://github.com/openbasalt/samba-conductor/blob/main/docs/passwords.md),
the relay and the queue in its
[mail.md](https://github.com/openbasalt/samba-conductor/blob/main/docs/mail.md),
and the delegated account in conductor-provisioner's
[delegation.md](https://github.com/openbasalt/samba-conductor-provisioner/blob/main/docs/delegation.md).

## Who does what

| Part | Does | Never does |
|---|---|---|
| conductor | the user pages (send, list and revoke links), the public reset form, the link pages, the second factor check and enrollment, the mail queue and its templates, its own audit entries | set a password or enable an account itself; keep a token or a password |
| conductor-provisioner | issues, checks and consumes the one-time tokens; sets the password and clears a lockout; enables an invited account; refuses accounts outside its scope OUs and privileged accounts; applies its own ceilings; audits every change and refusal | send e-mail; act on an account conductor did not name; keep the raw token or the password |
| the SMTP relay | delivers the messages; signs them (DKIM) and is named by the domain's SPF and DMARC records | talk to AD or to conductor-provisioner: it only receives finished messages from conductor |

conductor reaches conductor-provisioner on a local Unix socket that admits
only conductor's user, and the relay over SMTP with TLS (plain SMTP only
to a relay on loopback). Without `[mail]`, or without
conductor-provisioner enabled in `[provisioner]`, the features are off and
the pages say what is missing. Links are built from `server.public_url`,
never from a request's Host header.

## An invitation

1. An administrator or helpdesk user opens a user's page. conductor asks
   conductor-provisioner whether the account is below a managed OU and not
   privileged, and shows the reason when it is not.
2. On confirmation, conductor-provisioner issues a token bound to the
   account's SID, objectGUID and `pwdLastSet`, stores its hash and returns
   the raw token once. conductor puts it in a link and queues the message,
   sealed in its database, to the account's `mail` address.
3. The relay delivers the message. The queue entry is deleted as soon as
   the relay accepts it, or when the link expires.
4. The person opens the link. Opening only checks the token, so a mail
   scanner that fetches the link uses nothing. The button moves the token
   into a short-lived cookie, and the person chooses a password; the
   domain's password policy decides, and conductor-provisioner sets it.
5. When conductor's second factor policy asks for it, the person enrolls a
   second factor in conductor.
6. conductor-provisioner enables the account and consumes the token. It
   enables only an account that was disabled when the invitation was
   issued, so an account an administrator disabled later stays disabled.

Accounts created with "Send an invitation instead of setting a password",
and the accounts the Google-first mode creates, start disabled with a
random password nobody sees and follow the same steps.

## A reset by e-mail

1. The public form takes a username or an address and always answers with
   the same page at once. The lookup runs in the background: the answer
   does not tell whether the account exists or can be reset.
2. conductor applies its own limits per client address and per account,
   then asks conductor-provisioner to find the account and issue a token.
   Privileged accounts, accounts outside the managed OUs and disabled
   accounts get no link.
3. The link goes to the account's `mail` address and to its verified
   recovery address, if any.
4. On the link page, a user with a second factor must pass it (five wrong
   answers revoke the link), then sets the new password. A lockout is
   cleared when the setting says so, and every conductor session of the
   user ends.

## Notices

After an invitation is completed, a reset, a self-service change or a
reset by an administrator or helpdesk, conductor sends the user a message
with the time and the client address (or "an administrator"), to the
account's address and the recovery address. Notices carry no link.

## Where the secrets are

| Secret | Where it exists |
|---|---|
| the raw token | in the message (sealed in the queue until the relay takes it), in the person's mailbox, and for 10 minutes in a cookie and in conductor's memory during the link steps. conductor-provisioner keeps only its hash |
| the new password | typed in the browser, sent once over the socket to conductor-provisioner, which sends it to AD and forgets it; never logged, audited, stored or returned |
| the delegated account's password | a systemd credential of conductor-provisioner only |
| the relay password | a file or a systemd credential of conductor only |

The mailbox is part of the trust path: whoever reads the account's mail
while a link is valid can use it. That is why links are single use and
short-lived (a reset defaults to 30 minutes), why a reset of a user with a
second factor asks for it, why a new recovery address receives reset
links only 72 hours after it was set, and why the previous address is told
about the change.

## Limits that hold if conductor is compromised

conductor-provisioner does not trust conductor. A compromised web process
can issue and use tokens, and so set passwords and enable invited
accounts, but only for ordinary user accounts below the managed OUs and
within the ceilings of conductor-provisioner's own root-owned
configuration (per hour for each kind of operation, and tokens per account
per day). It cannot act on a privileged account (built-in administrative
groups, conductor's role groups listed in `extra_group_sids`, `adminCount`,
holders of more than read on the domain head, an OU, AdminSDHolder or a Group
Policy object), outside the scope OUs, or on the delegated account itself,
and it cannot read a stored token. Both conductor and conductor-provisioner
audit every step; conductor-provisioner's log is a keyed hash chain.

List conductor's role group SIDs in conductor-provisioner's
`extra_group_sids`: it knows the built-in administrative groups on its own,
but a custom helpdesk or auditor group only from that list.

## Setting it up

1. Configure the relay in conductor's `[mail]` and send a test message
   ([mail.md](https://github.com/openbasalt/samba-conductor/blob/main/docs/mail.md)).
2. Install conductor-provisioner on conductor's host, create its delegated
   account and delegate the managed OUs
   ([install.md](https://github.com/openbasalt/samba-conductor-provisioner/blob/main/docs/install.md),
   [delegation.md](https://github.com/openbasalt/samba-conductor-provisioner/blob/main/docs/delegation.md)).
3. Enable `[provisioner]` in conductor and set `server.public_url`
   ([config.md](https://github.com/openbasalt/samba-conductor/blob/main/docs/config.md)).
4. Choose the link lifetimes, the public reset form and the second factor
   rule in Settings > Passwords
   ([passwords.md](https://github.com/openbasalt/samba-conductor/blob/main/docs/passwords.md)).

The packages are described in [packaging](packaging.md); the architecture
of conductor-provisioner in [architecture](architecture.md#conductor-provisioner).
