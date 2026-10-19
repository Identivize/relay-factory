# Troubleshooting

What the installer's exit codes mean and what to do, then the situations
an install runs into, then every exit code of the `factory` CLI.

Every failure prints `factory: <message>` on stderr, and with `--json` one
JSON object on stdout whose `error` carries the stable code and the
message. Re-running `factory admin install --installation <handle>` is
always safe: it resumes where the last run stopped.

## The installer's exit codes

<!-- cliref:begin installer-exit-codes -->

| Code | Exit | From | When |
|---|---|---|---|
| `preflight_failed` | 25 | CLI | `factory admin install`: a preflight check is FAIL. Nothing was changed, or at most the control-plane project was created. |
| `handoff_pending` | 26 | CLI | `factory admin install`: the run did its work and a Handoff is still pending; with `--check`, no check is FAIL but one needs a Handoff. |
| `preflight_unknown` | 27 | CLI | `factory admin install --check`: no check is FAIL or needs a Handoff, but one could not be checked. |
| `install_failed` | 28 | CLI | `factory admin install`: a step failed (the rebase pass included), or the plan could not be made (then nothing was changed); the message names it. Re-running resumes. `factory admin rebase`: the pass stopped part way (re-running resumes), or the Installation was applied by a release before the pass existed: run `factory admin install` from this release first. `factory admin status` and `factory admin config get`: the Installation's config or ledger could not be read from the state bucket. `factory admin uninstall`: a deletion step failed (the message names it); running it again finishes, and what is deleted stays deleted. |
| `signature_invalid` | 29 | CLI | `factory admin install`: a release image's signature did not verify. |
| `older_release` | 30 | CLI | `factory admin install` and `factory admin rebase`: this factory is an older release than the one the Installation runs; an Installation never moves back. Nothing was changed; use that release or a newer one. |
| `permanent_input` | 37 | CLI | `factory admin install`: a permanent input (`workspaceDomain`, `region`, `companyHost`, `artifactDomain`, `projectPrefix` or a pre-created project's name) was given with a value other than the one the Installation was installed with; the message names it. They name the Installation and cannot be changed by a re-run. Nothing was changed; drop the input to resume. |
| `concurrent_run` | 39 | CLI | `factory admin install`: another run of the installer wrote the Installation's config or ledger while this run was working (the state bucket's generation precondition refused this run's write); the message names the object and, when it records one, the other writer. This run wrote nothing there and changed nothing more. Wait for that run to finish, then re-run. |

<!-- cliref:end -->

### Exit 25: `preflight_failed`

A preflight check is `fail`, and the report says which. Nothing was
changed, with one exception: on a first run, when an organization policy
could only be read from the new project, the control-plane project alone
may have been created; the message then says so, and the next run adopts
it (below). Fix what the check names and run again:

- `release`: this is a development build (`go build`), or the Installation
  runs a newer release than this `factory` (that case is exit 30, below).
  Use the release's own `factory` from [Install](install.md#1-get-factory).
- `gcloud`: gcloud is missing or signed out. `gcloud auth login` as the
  Operator.
- `organization`, `project-create`, `billing`: the Workspace domain has no
  Google Cloud organization, or the Operator lacks Project Creator on it or
  Billing Account User on the billing account, or the account is closed
  ([Prerequisites](prerequisites.md#the-organization-and-the-account)).
  Project Creator is not needed with
  [pre-created projects](install.md#pre-created-projects).
- `project-owner`, `project-parent`, `firestore-location`: a pre-created
  project does not exist, the Operator is not its Owner, it is not under
  the Workspace domain's organization, or the control-plane project
  already has a `(default)` Firestore database (the install creates the
  Installation's) or has fixed its location of default Google Cloud
  resources somewhere other than the region; the message names the
  project. Name another project, or install in that location
  ([Prerequisites](prerequisites.md#pre-created-projects)).
- `region`, `cross-project-lb`, `global-lb`, `ingress`: an organization
  policy denies what the Installation needs
  ([Prerequisites](prerequisites.md#organization-policies-the-preflight-checks)).
- `spend-cap`: a Spend cap is given and the Operator lacks
  `billing.resourcebudgets.configureSpendCap` on the Artifacts project,
  which the Owner and Editor roles hold. Have one of those roles granted
  on the project, or drop `--spend-cap` ([Install](install.md#spend-cap)).
  With no cap the check passes and says the Installation has no money
  ceiling.

### Exit 26: `handoff_pending`

The run did its work and a Handoff is pending: one that gates readiness
(`dns`, `oauth-client` or `invoker-iam`), or `groups-reader` or `email`,
which do not gate readiness (the message then says the Installation is
ready) but still need a Workspace super admin to act; or every Handoff is done and
the Front Door is not answering yet, which takes
a few minutes after the certificates become active. The report lists each
pending Handoff with the instructions to forward, a link to its page and a
`Now:` line saying what verification last found missing. Forward what is
pending, then run again, or let a run wait with `--wait`
([Install](install.md#5-resume-and-wait)). A pending `artifact-capacity`
alone never causes this exit. With `--check`, this exit means no check
failed but one needs a Handoff; before the first run that is always so,
since the Handoffs are still to come, so a first `--check` exits 26 or 25
and nothing else.

### Exit 27: `preflight_unknown`

`--check` on an existing Installation only: no check failed or needs a
Handoff, but one could not be made; the message says what could not be
read. (Before the first run a `--check` exits 26, whatever is unknown,
because the Handoffs are still to come; the report lists the unknown
checks, usually organization policy and the Admin group read against the
Operator's own gcloud project before any project exists, and the first
run reads them again on the new project.) An unknown check never stops a
run without `--check`: the install goes on and the report lists what could
not be checked.

### Exit 28: `install_failed`

A step failed; the message names the step and the error. Run again: the
sequence resumes at the step. The same exit, with or without `--check`,
when the plan could not be made before the run began (`No plan could be
made …`): a registry or image read failed, or Terraform could not plan
the module; the message carries the error, and nothing was changed. A failed Terraform apply is already retried
twice, 30 seconds apart, within the run, because a new project's IAM
grants take a moment to reach every Google backend; a persistent failure
has its cause in the message. Two of them have their own sections below:
a stale Terraform lock, and state Terraform could not save. The
`spend-cap` step fails this way on purpose in two cases, and the message
says which: the cap is enforced and you gave another amount (lift the
cap in Google's billing console or by the Budget API first, then re-run
with the new amount; Relay Factory never lifts a cap), or a budget for
the Artifacts project and Cloud Run already exists that the
Installation's config does not name (a Billing Account Administrator
finds it with `gcloud billing budgets list --billing-account=<id>` and
deletes it, then re-run) ([Install](install.md#spend-cap)).
`factory admin uninstall` exits 28 too when one of its deletion steps
fails (the budget, the projects, the Artifacts' services, the module's
destroy or the state bucket); the message names the step. Run it again
with the same flags: what is deleted stays deleted, and the run finishes
the rest ([Uninstall](uninstall.md)).

### Exit 29: `signature_invalid`

A release image's signature did not verify against the signer stamped
into this `factory`: the image in the public registry is not the one the
release workflow signed. The image is not copied and no service is
deployed. Do not work around it; report it on the public repository's
issues.

### Exit 30: `older_release`

This `factory` is an older release than the one the Installation runs; an
Installation never moves back. Nothing was changed. The message names the
Installation's release: use that one or a newer one
([Upgrade](upgrade.md#what-is-refused)).

### Exit 37: `permanent_input`

A re-run gave a permanent input with a value other than the one the
Installation was installed with: `workspaceDomain`, `region`,
`companyHost`, `artifactDomain`, `projectPrefix` or a pre-created
project's name (`controlPlaneProject`, `artifactsProject`,
`notifierProject`, `buildProject`). They name the
Installation, so a re-run cannot change them; the message names the input
and both values. Nothing was changed, not even preflight ran. Drop the
input (or the key in the `--config` file) to resume. To have another
domain, host or region, install a new Installation
([Upgrade](upgrade.md#what-is-refused)). The one exception is
`buildProject` on an Installation that stored none: the re-run that
turns `--source-built` on may name it, once
([Install](install.md#source-built-apps)). The mutable inputs
(`billingAccount`, `adminGroup`, `artifactCapacity`, `spendCap`,
`sourceBuilt`, `emailSender`) are changed by a re-run with the flag
([Upgrade](upgrade.md#changing-a-mutable-input)).

### Exit 39: `concurrent_run`

Another run of the installer wrote the Installation's config or ledger
while this run was working: every write of those two objects is
conditioned on the generation the run read them at, and the state bucket
refused this one. The message names the object, the generations and, when
the object records one, who made the other run and when. This run wrote
nothing there and changed nothing more; the bucket holds what the other
run wrote. Wait for that run to finish (ask the Operator it names), then
re-run `factory admin install --installation <handle>`: it reads the state
the other run left and resumes from there. The Terraform apply itself is
guarded by Terraform's state lock
([A stale Terraform lock](#a-stale-terraform-lock)).

## Situations

### A stale Terraform lock

If the installer is killed outright while Terraform is applying (a
`kill -9`, a crashed machine; a Ctrl-C or a dropped SSH session is passed
on to Terraform, which unlocks before it stops), Terraform's state lock
stays in the state bucket. The next run fails with exit 28 and a message
ending:

> If another install is running, wait for it. If none is, an earlier one
> was killed before it could unlock: remove the lock object
> `gs://<handle>-installation/terraform/state/default.tflock` and run the
> install again

Make sure no other install of this Installation is running, remove that
one object (`gcloud storage rm gs://<handle>-installation/terraform/state/default.tflock`),
and run again.

### State Terraform could not save

If Terraform could not write its state back to the bucket after an apply,
the run fails with exit 28 and a message naming a local `.tfstate` file
the installer kept under its cache directory (`terraform/errored/`). Keep
that file: it is the only record of what that apply created. Push it to
the state bucket with `terraform state push` before the next run, as the
message says, or the next install does not know about those resources.

### Adopting a half-created project

A first run creates the control-plane project and prints its ID before it
writes the Installation's config. If the run dies in that window (or
preflight fails on a policy it could only read from the new project), the
project exists but no later run knows it is an Installation. Run the first
command again with the same inputs plus `--installation <the printed ID>`:
the installer adopts that project instead of creating another. The ID must
have the form `<prefix>-cp-<suffix>` and the project must exist; an
unknown ID is a `usage` failure, never three new projects. If creating the
control-plane project itself failed there is nothing to adopt: run the
first install again. Once the config is written, every later interruption
simply resumes with `--installation <handle>`. With
[pre-created projects](install.md#pre-created-projects) the handle is the
named control-plane project: re-run with
`--installation <controlPlaneProject>` and the same inputs.

### Certificate timing

Google issues the two certificates about 20 to 25 minutes after the two
`_acme-challenge` CNAMEs resolve in public DNS, and documents no bound
beyond "several minutes": allow up to an hour before suspecting the
records. The DNS Handoff's `Now:` line says which stage it is at: a CNAME
not resolving yet, a certificate not active yet, or an A record answering
with another address. While a certificate is pending, check the CNAMEs
through a public resolver (`dig +short CNAME _acme-challenge.<company-host>`),
which is what the installer and Google see, not your own resolver. After
the certificates are active, TLS answers a few minutes later, and the
Installation becomes ready on the next verification; `--wait 40m` covers
the whole span ([DNS](handoffs/dns.md)).

### The groups-reader Handoff stays pending

The installer verifies this Handoff through the Installation itself: the
Control API and the Hub each read the Admin group as their own service
account, with the Groups Reader role, at start-up and every five minutes,
and the installer reads both outcomes from the Front Door's discovery
document ([Groups Reader](handoffs/groups-reader.md)). The Handoff's
`Now:` line names the service it found wanting (the Control API first)
and which of three things it found:

- **Not reported yet.** That service has not made its check since the
  role was assigned, or the Front Door could not be read yet (the DNS
  Handoff comes first). Wait five minutes and run again; `--wait` helps
  only while the Front Door is the reason, since the role assignment needs
  a person.
- **Google answered 403.** The role is missing on that service account,
  or its scope is narrower than all groups; Google does not say which.
  Have the super admin open the Groups Reader role's assignments in the
  Admin console, confirm both addresses from the report are listed, and
  set the scope to all groups. Then run again five minutes later.
- **Google answered 404: the Admin group was not found.** The group
  given as `--admin-group` does not exist in the Workspace account, or
  was renamed or deleted after the install. Create or restore it under
  that address, or re-run the installer with `--admin-group` naming
  another group: the input is mutable
  ([Upgrade](upgrade.md#changing-a-mutable-input)).

A message that the service could not reach the Admin SDK is none of these:
the service tries again in five minutes on its own.

### The email Handoff stays pending

Issued only on an Installation with `--email-sender`. The installer never
sends the test mail itself: the sweep does, every five minutes while the
Handoff is pending, as the sender mailbox through the delegation of the
sender account, and records what Google answered; the installer reads
that record ([Email](handoffs/email.md)). The Handoff's `Now:` line says
which it found:

- **The sweep has not tried the test mail yet.** The sweep runs every five
  minutes; wait five minutes and run again. If it still says so after
  that, the sweep is not running: check the `sweep` Cloud Scheduler job
  and the service's logs in the control-plane project (the log lines start
  `sweep: email:`).
- **Google refused the delegation.** `unauthorized_client` means the
  client ID is not authorized in the Admin console, or not for the
  `https://www.googleapis.com/auth/gmail.send` scope, or was entered by
  email rather than by its numeric ID; have the super admin open Manage
  Domain Wide Delegation and compare the client ID and the scope with the
  report. `invalid_grant` with "Invalid email or User ID" means the
  mailbox given as `--email-sender` is no user of the Workspace account:
  create it, or re-run the installer with `--email-sender` naming an
  existing licensed mailbox ([Upgrade](upgrade.md#changing-a-mutable-input)).
  Google says a new delegation can take up to 24 hours to apply, and
  typically much less.
- **The sweep could not send the test mail.** A 5xx, a rate limit or the
  network: the sweep tries again in five minutes on its own.
- **The email state could not be read from the registry.** The Operator's
  account could not read the control-plane project's Firestore; the
  installer's other reads would have failed too. Check the account and
  run again.

`factory admin status` shows the same state as `Email: …`
([Admin](admin.md#what-an-admin-can-see)).

### A Handoff that was done goes back to pending

The installer verifies every Handoff against the world on every run and
trusts no reply. A DNS record removed, a quota lowered, an exemption
withdrawn or a replaced OAuth client makes its Handoff pending again, with
a `Now:` line saying what changed. Restore it, or complete the Handoff
again. A changed `--email-sender` does the same to `email` until the sweep
has sent the test mail as the new mailbox; a revoked delegation is noticed
when a mail is refused: the first Admin-change, Share or expiry mail
Google refuses ([Email](handoffs/email.md), [Admin](admin.md)).

### The Installation is Paused

What you see: every Artifact link answers a paused page, and `factory
publish` and `factory restore` exit 19, `installation_paused`.
`factory renew` still works. Why: the Operator's
[Spend cap](install.md#spend-cap) is enforced, so Google has stopped
Cloud Run in the Artifacts project until the cap is lifted or the month
ends, and the
sweep has recorded the Installation as Paused. Nothing in a
bundle is wrong, and publishing again does not help. Who acts: the
Operator, who lifts the cap in Google's billing console or by the Budget
API and then re-runs `factory admin install --spend-cap <amount>` with
the new amount; otherwise the cap lifts itself on the first of next
month. Relay Factory records the Installation serving again once two
sweep runs in a row find the cap lifted, a few minutes after the lift.
`factory admin status` shows the state as `Serving: …`
([Admin](admin.md#what-an-admin-can-see)).

## Every exit code

The `factory` CLI's exit codes, all commands. "From" says who reports it:
"API" codes are the Installation's Control API's, passed through with its
message; "CLI" codes the CLI decides itself. Codes are stable: never renamed
or renumbered. Publisher commands' codes are explained on the
[Publisher](publisher.md#exit-codes) page.

<!-- cliref:begin exit-codes -->

| Code | Exit | From | When |
|---|---|---|---|
| `internal` | 1 | API, CLI | Anything else, including a credential store that cannot be read or written, and any code this CLI does not know. |
| `usage` | 2 | CLI | Bad command line, a host that is not an Installation host, or a malformed link file. `factory admin install`: also a missing or invalid input, an unknown Installation, or an OAuth client Google rejects. `factory admin rebase`, `factory admin status` and `factory admin config get`: an unknown Installation, or one whose install never wrote its config. `factory admin uninstall`: those too, a missing or wrong `--confirm`, or an Installation whose projects are already pending deletion; nothing was changed. |
| `unauthenticated` | 3 | API, CLI | Not signed in to the Installation, a sign-in whose OAuth client was replaced, a refresh Google refuses, or an ID token the Control API rejects. Sign in again with `factory login <host>`. |
| `wrong_domain` | 4 | API, CLI | The Google account has no Workspace domain, or not the Installation's. |
| `not_publisher` | 5 | API | The Artifact belongs to another Publisher. |
| `not_found` | 6 | API | No such Artifact or Revision on this Installation. |
| `invalid_request` | 7 | API | The Control API refused the request's body. |
| `unsupported_kind` | 8 | API | The kind sent is not `static`, `streamlit` or `python`. |
| `too_large` | 9 | API, CLI | The bundle is above 100 MiB compressed. |
| `deploy_in_progress` | 10 | API | Another publish of the Artifact holds its publish lock. Do not publish again, or end the Artifact, until it finishes; the message says when the lock lapses. |
| `installation_at_capacity` | 11 | API | The Installation has no room for another Artifact. |
| `superseded` | 12 | API | The Revision being committed is no longer the Artifact's pending one. |
| `upload_missing` | 13 | API | The commit found no uploaded bundle for the Revision. |
| `upload_mismatch` | 14 | API | The uploaded bundle's size or checksum differs from what was declared. |
| `not_configured` | 15 | API, CLI | The Installation's desktop OAuth client is not set up yet (a pending install Handoff). |
| `unavailable` | 16 | API | The Installation's discovery document was still unavailable after the CLI's retries. |
| `publisher_at_limit` | 17 | API | `factory publish` and `factory restore`: the Publisher holds the live Artifacts the Installation allows per Publisher (50 by default), so a publish that needs a new one is refused; a republish of a live Artifact never is. `factory transfer`: the receiving Publisher would go over. The message gives the count and the way out: let one expire, `factory end` one, or ask an Admin to raise `liveArtifactsPerPublisher`. |
| `publish_rate_limited` | 18 | API | `factory publish` and `factory restore`: the Publisher has made the publishes the Installation allows per Publisher per UTC day (100 by default; Restores count). The message names when the count resets (the next UTC midnight); wait, or ask an Admin to raise `publishesPerPublisherPerDay`. |
| `installation_paused` | 19 | API | `factory publish` (at create and at issue) and `factory restore`: the Installation is Paused because the Operator's Spend cap is enforced; Google has stopped its Artifacts until the cap is lifted. Nothing to fix in the bundle: the Operator lifts the cap, or it lifts on the first of next month; `factory renew` still works. |
| `network` | 20 | CLI | The Installation, Google or the upload host could not be reached, did not answer in time, or answered with something that is not the contract. |
| `invalid_bundle` | 21 | CLI | The folder cannot be published as it is: its `factory.yaml` is invalid or its kind could not be detected, a `static` app has no root `index.html` or holds Python, a `streamlit` or `python` app's `requirements.txt` or `pyproject.toml` names a package source other than the Installation's mirror (`--index-url`, `--extra-index-url`, `--find-links`, a direct URL or `git+` requirement, `-r`/`-c`), or the folder has a symlink or special file, a whiteout name, an unreadable file, or no regular file. The message names the rule and the way out. |
| `revision_failed` | 22 | CLI | The Revision was uploaded and its publish failed (`publish`, or `status --wait` on it); the message is its failure reason: a bundle the deploy worker refused (its manifest, kind, static rule or dependency sources), or a source-built build, scan or start that failed (`build failed`, `build timed out`, `scan failed`, `scan unavailable`, `app did not start`), whose log tail `factory status` shows. |
| `wait_timeout` | 23 | CLI | The Revision was not live or failed when `--timeout` passed (`publish`, or `status --wait` on it); or the Access Policy was not yet on the service when `share`'s or `unshare`'s `--timeout` passed (the change stands and is applied when the deploy worker catches up). |
| `login_failed` | 24 | CLI | The sign-in was refused, abandoned, timed out, or Google's answer lacked what is needed. |
| `preflight_failed` | 25 | CLI | `factory admin install`: a preflight check is FAIL. Nothing was changed, or at most the control-plane project was created. |
| `handoff_pending` | 26 | CLI | `factory admin install`: the run did its work and a Handoff is still pending; with `--check`, no check is FAIL but one needs a Handoff. |
| `preflight_unknown` | 27 | CLI | `factory admin install --check`: no check is FAIL or needs a Handoff, but one could not be checked. |
| `install_failed` | 28 | CLI | `factory admin install`: a step failed (the rebase pass included), or the plan could not be made (then nothing was changed); the message names it. Re-running resumes. `factory admin rebase`: the pass stopped part way (re-running resumes), or the Installation was applied by a release before the pass existed: run `factory admin install` from this release first. `factory admin status` and `factory admin config get`: the Installation's config or ledger could not be read from the state bucket. `factory admin uninstall`: a deletion step failed (the message names it); running it again finishes, and what is deleted stays deleted. |
| `signature_invalid` | 29 | CLI | `factory admin install`: a release image's signature did not verify. |
| `older_release` | 30 | CLI | `factory admin install` and `factory admin rebase`: this factory is an older release than the one the Installation runs; an Installation never moves back. Nothing was changed; use that release or a newer one. |
| `not_expired` | 31 | API | `factory restore`: the Artifact is live, so there is nothing to restore; publish or renew it instead. |
| `restore_disabled` | 32 | API | `factory restore`: this Installation does not let Publishers restore their own Artifacts; ask an Admin to restore it. |
| `renewals_exhausted` | 33 | API | `factory renew`: the Artifact has used the idle Renewals the Installation allows per Let; publish again to start a fresh Let. |
| `not_admin` | 34 | API | `factory admin policy`, `factory admin status`, `factory admin end`, `factory transfer` and `factory restore --publisher`: the member signed in is not a member of the Admin group (`RELAY_ADMIN_GROUP`). Sign in as an Admin, or ask one. |
| `not_live` | 35 | API | `factory end` and `factory renew`: the Artifact has no running Let (never live, or Expired already). Its link is already down; publish again to bring it back. |
| `source_built_disabled` | 36 | API | The `factory.yaml` declares `streamlit` or `python` and this Installation does not build apps from source: the Operator has not installed the capability (`factory admin install --source-built`), or an Admin has the policy's `sourceBuiltAllowed` off; the message says which. Only `static` publishes until then. |
| `permanent_input` | 37 | CLI | `factory admin install`: a permanent input (`workspaceDomain`, `region`, `companyHost`, `artifactDomain`, `projectPrefix` or a pre-created project's name) was given with a value other than the one the Installation was installed with; the message names it. They name the Installation and cannot be changed by a re-run. Nothing was changed; drop the input to resume. |
| `nothing_retained` | 38 | API | `factory restore`: the Artifact has no retained bundle (never live, its Retention has passed, or the bundle is gone); publish it again. |
| `concurrent_run` | 39 | CLI | `factory admin install`: another run of the installer wrote the Installation's config or ledger while this run was working (the state bucket's generation precondition refused this run's write); the message names the object and, when it records one, the other writer. This run wrote nothing there and changed nothing more. Wait for that run to finish, then re-run. |
| `unknown_name` | 40 | API | `factory share` and `factory publish --share`: a name is not a member or group Google recognises in the Workspace account (a deleted or mistyped address, or a group's address given as a member); the message names it. The whole command was refused and nothing changed; fix the name and share again. |
| `not_all_ended` | 41 | CLI | `factory admin end`: with `--yes`, at least one Artifact was skipped (a publish in flight; the line says when its lock lapses) or failed, after every one was tried; without it, an ID named is not on the Installation. Each Artifact's outcome is in the output; run it again for the ones that need it. |
| `not_whole_domain` | 42 | API | `factory share listed`: the Artifact is not open to the whole Workspace domain (its domain switch is off), so it cannot be shown in the Hub's Across the domain. Nothing was changed; `factory share domain` first, then list it. |
| `interrupted` | 130 | CLI | SIGINT or SIGTERM stopped the command. A Revision already committed may still go live. |

<!-- cliref:end -->
