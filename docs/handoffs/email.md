# Handoff: Email

**Who does it:** a Google Workspace super administrator of the
Installation's Workspace account; a Workspace admin creates the mailbox.

**What it is for:** an Installation sends its mail (the Share mail, the
expiry mails, the Admin-change mails) as one dedicated mailbox, such as
`relay-factory@example.com`, through the Gmail API. It does so without
any password or key: a service account of its own, the **email sender
account** in the Installation's notifier project, is authorized by the
Workspace account to send mail as that mailbox, and only to send
(domain-wide delegation with the single scope
`https://www.googleapis.com/auth/gmail.send`). Three of the
Installation's services may use that account, and nothing else may. Only a
super admin can authorize a delegation, and only Google's Admin console
offers it, so the installer cannot do it. This Handoff exists only on an
Installation whose Operator gave `--email-sender`; without one the
Installation runs with no email, and every command that would mail says so.

## Steps

The installer's instructions name the sender account
(`email-sender@<notifier project>.iam.gserviceaccount.com`), its **client
ID** (a number, also `clientId` in the report's `--json`), the scope and
the mailbox.

1. **The mailbox.** If it does not exist yet, create the user the
   Operator named as `--email-sender` in the Admin console (Directory,
   Users, Add new user) and give it a license that includes Gmail. A
   normal user mailbox is what the delegation needs; it is never signed
   into. One mailbox can serve several Installations (each has its own
   sender account to authorize).
2. In the Google Admin console (`https://admin.google.com`), signed in as
   a super administrator: Menu, Security, Access and data control, API
   controls, then **Manage Domain Wide Delegation**.
3. **Add new.** In Client ID, enter the client ID from the installer's
   instructions (the number, not the account's email address). In OAuth
   scopes, enter exactly
   `https://www.googleapis.com/auth/gmail.send` and nothing else.
4. **Authorize.** Google says the change can take up to 24 hours, and
   typically much less.
5. Tell the Operator it is done.

## Done when

The Installation has sent a test mail to its Admin group as the mailbox.
Its sweep service tries every five minutes while the Handoff is pending:
until the delegation is authorized Google refuses the token and no mail
leaves; once it is, the next try sends one mail, subject `Relay Factory at
<company host> can send email`, to the Admin group, and records that email
is on. The installer reads that record: the Operator re-runs
`factory admin install --installation <handle>` once the super admin is
done, or waits five minutes first so that the sweep has run. `factory
admin status` shows `Email: on, sent as <mailbox>` from then on.

Until then the report shows the Handoff as pending with a `Now:` line
saying what the last try found: not tried yet (wait five minutes), Google
refusing the delegation (the client ID or the scope is not authorized, or
the mailbox is no user of the account: check both against the
instructions), or the sweep unable to reach Google (it tries again)
([Troubleshooting](../troubleshooting.md#the-email-handoff-stays-pending)).

The Installation serves Publishers while this Handoff is pending, so the
run reports it ready with this Handoff pending and exits 26
(`handoff_pending`) until it is done; it exits 0 once it is
([Install](../install.md#6-the-ready-report)). Once email is on the sweep
leaves it on: a delegation revoked later is noticed when a mail is refused,
which turns email off and the Handoff back to pending: the first mail
Google refuses (an Admin-change mail, a Share mail or an expiry mail)
notices it, and `factory admin status` then says so. Changing the mailbox
(`--email-sender` on a re-run) needs no new console step unless the new
mailbox does not exist: the delegation is the account's, not the
mailbox's ([Upgrade](../upgrade.md#changing-a-mutable-input)).
