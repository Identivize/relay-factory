---
name: factory-admin
description: "Install, check or administer a Relay Factory Installation with the factory CLI: run factory admin install --check, read the installer's report and its Handoffs, hand the apply to the Operator, change a mutable input (billing account, Admin group, capacity, Spend cap) by a re-run, read the stored config with factory admin config get, read and change the Installation's policy with factory admin policy get and set, read its state with factory admin status, end any Artifact's Let early with factory admin end, and hand the Operator the uninstall (factory admin uninstall). Use when asked to install, check, upgrade, administer or uninstall a Relay Factory Installation, to change its inputs, to change its policy, or to take an Artifact down."
license: Apache-2.0
metadata:
  version: "v0.1.4"
---

# factory-admin: install and administer an Installation

A Relay Factory Installation is one organization's own: three Google Cloud
projects, installed and upgraded by its **Operator** with
`factory admin install`, and governed by its **Admins**, the members of
the Admin Google Group. This skill is for both. Publishing is every
member's, with the `factory` skill. The CLI never prompts: inputs come from
flags or a `--config` file, progress goes to stderr, the result to stdout
(one JSON object with `--json`), and every failure has a stable code and
exit code.

This skill describes the `factory` of its own release: the version in this
file's frontmatter is the one the CLI's `version` command prints. If they
differ, refresh the skill with `factory skills install --admin`.

## 1. What this release has

Under `factory admin` this release has `install`, `config get`, `rebase`
and `uninstall` (the Operator's: sections 2 to 4, 6, 7 and 8), `policy get` and
`policy set` (the Admin's read and write of the Installation's policy:
section 5), `status` (the Installation's state, the Spend cap
included: section 6) and `end` (the Admin's end of any Artifact's Let, by
ID or by a filter: section 10). The Admin's
other commands are top-level: `factory transfer` gives an Artifact to
another Publisher, and `factory restore` with `--publisher` restores an
Expired Artifact as another member's (section 9). The rest (re-sharing
another Publisher's Artifact, reading the Installation activity record)
come in later releases, each as a section of this skill.

## 2. First move: the check

```sh
factory version
factory admin install --check --installation <handle> --json
```

`--check` runs the preflight checks and changes nothing. The handle is the
Installation's control-plane project ID, which the first install printed.
Read the exit code: 0 means every check passed; 25 `preflight_failed` means
one is FAIL and the report says which; 26 `handoff_pending` means a
Handoff is still to be done (nothing is wrong); 27 `preflight_unknown`
means one check could not be made. Before the first install there is no
Installation to name: `--check` with the inputs instead checks that the
inputs and the organization are ready for one, and exits 26 (go on to the
first run: the Handoffs are all still to come) or 25, never 0; 0 and 27
are for an existing Installation.

A development build (`factory version` prints `dev`) is refused by the
installer; use a release.

## 3. The install is the Operator's

`factory admin install` without `--check` is a Terraform run against the
organization's Google Cloud with the Operator's own `gcloud` login. **Run
the check yourself; hand the apply to the Operator** with the exact
command line, and never run it on your own. It never prompts and re-running
it is always safe: after a failure, an interruption or a Handoff it resumes
where it stopped.

The first run takes the inputs (`--workspace-domain`, `--billing-account`,
`--region`, `--company-host`, `--admin-group`, and the optional
`--artifact-domain`, `--artifact-capacity`, `--project-prefix`), from the
flags, a `--config` JSON file, or both; it prints the handle. Later runs
take `--installation <handle>` alone, or with the flag of a **mutable**
input to change it: `--billing-account`, `--admin-group`,
`--artifact-capacity` and `--spend-cap` are applied by the run and then
stored, and the plan's `Input <key> (<flag>): from <stored> to <given>;
applied by this run, then stored.` line says so first, so run `--check`
with the flag before handing the Operator the run. A **permanent** input
(`--workspace-domain`, `--region`, `--company-host`, `--artifact-domain`,
`--project-prefix`, and the pre-created projects' names) given with
another value is refused with
`permanent_input` (37) and nothing changes: drop it, or it is a new
Installation. An Operator who is Owner of existing projects but not a
Project Creator names them on the first run, `--control-plane-project`,
`--artifacts-project` and `--notifier-project` together (no
`--project-prefix`; `--build-project` is a fourth, the build project of
source-built apps, adopted with `--source-built`): preflight checks each for the
Operator's Owner permissions and its organization (`project-owner`,
`project-parent`), passes `project-create` without the Project Creator
role, and reads the control-plane project's Firestore location
(`firestore-location`, unknown on a `--check` until the run has enabled
the project's APIs, read again then); the plan says `adopt`, the run
creates no project, and the handle is the named control-plane project.
A new Admin group is stored only once the second-phase
apply has succeeded; a run held by the `invoker-iam` Handoff says it is
not stored yet, and the re-run needs the flag again. `--email-sender <address>` names the
mailbox the Installation sends mail as (a licensed user the Workspace admin
creates): mutable, applied by the second-phase apply like the Admin group,
and the reason the `email` Handoff is issued; without it the Installation
has no email. `--viewing-records` (default off) routes IAP's entries to BigQuery, kept two days,
and an hourly scheduled query keeps each request's time, Artifact, member and outcome for
`--viewing-records-retention-days` (default 400); entries in the sink's first minute or so can be
lost. The rows are read through the datasets' sharing, the Operator's (basic roles included); `--viewing-records=false`
removes the sink and queries and keeps the record datasets. A second query counts views into a dataset naming nobody, which the Hub's Admin pages show (All Artifacts, Installation). Google's 30-day log of every view is on in every Installation.

**Source-built apps** (`--source-built`, ADR-0009) are off by default,
and `--source-built` on a first run or a re-run installs the capability:
a fourth, build project (`<prefix>-bld-<suffix>`, or the pre-created one
named with `--build-project`, which the switch then requires) with Cloud
Build, a builder account with no rights outside it, the PyPI mirror the
builds install through and the source bucket they read; in the Artifacts
project a second runtime account with no permissions and the no-egress
network (a VPC with no Cloud NAT and a deny-all rule). The plan names
the project and the resources; no Handoff is needed. The switch is
stored once the project exists; `--source-built=false` removes the
resources and keeps the project (uninstall deletes it). Installing it
lets nobody publish yet: an Admin turns the policy's `sourceBuiltAllowed`
on (section 5), and until both are done a `streamlit` or `python`
publish is refused with `source_built_disabled` (36), whose message says
which is missing. It is the Operator's decision, like the Spend cap: the
flag goes on the command line you hand them.

**The Spend cap** (`--spend-cap <amount>`, a whole number of the billing
account's currency units) is the Operator's money ceiling: the run's
`spend-cap` step makes it one Google budget capping Cloud Run in the
Artifacts project, with Google's alerts at 50, 80 and 100 % mailed to
the billing account's administrators and the project's Owners. The plan's
`Spend cap:` line says what the run would do (create, update, current,
leave, blocked, or `none; the Installation has no money ceiling`); a
blocked plan (the cap is enforced and another amount is given) is
refused with exit 28 before anything changes, `--check` included. Without a cap the
Installation has no money ceiling, and the `spend-cap` check says so.
Relay Factory never lifts or removes a cap: `--spend-cap 0` leaves an
existing budget as it is, and a cap Google has enforced must be lifted by
the Operator in the billing console or by the Budget API before a re-run
with a new amount (the run fails with `install_failed` otherwise and says
so).

**Handoffs.** The report lists what someone else must do: the DNS records,
the OAuth client (its ID and secret come back as
`--oauth-client-id` and `--oauth-client-secret-file`), the Groups Reader
role (a Google Workspace super admin assigns it, with scope all groups, to
the two service-account addresses the instructions name; it does not
block ready, but the run exits 26 until it is done, so re-run five
minutes after the super admin is finished), the email delegation (only
with `--email-sender`: a super admin authorizes the sender account's
client ID, which the instructions and the report's `clientId` give, for
the one `gmail.send` scope in the Admin console's Manage Domain Wide
Delegation; the sweep then sends a test mail to the Admin group within
five minutes, and the next run reports the Handoff done; like
`groups-reader` it does not block ready but holds exit 26), the Artifact
capacity quota, the invoker IAM grant. Each Handoff's instructions end with a `Guide:`
link to its own page; forward them to the person who does it, then re-run
the installer, or let it keep verifying DNS with `--wait <duration>`.
Exit 26 means the run did its work and a Handoff is still pending (the
message says whether the Installation is ready meanwhile); exit 0 means
the Installation is ready and only `artifact-capacity` may be pending.

## 4. Upgrade

An Installation moves to a newer release by running that release's
`factory admin install --installation <handle>`, never back: an older
`factory` is refused with `older_release` (30). Check first with the new
release's `--check`. When the release changed the base image live
Artifacts are served from, the plan shows a `Rebase:` line (how many live
Artifacts the run would redeploy onto it) and the run's steps show
`rebase`: the upgrade redeploys every live Artifact onto the new base
after the apply, as a Revision of the same bundle, one a second (the run
waits about a second per live Artifact), with no new Let and nothing for
Publishers to do (section 7).

## 5. The policy

The policy is the Installation's one document of Admin settings: the
Let's bounds, the Retention, the expired page and the Restore Requests
([restore-requests.md](references/restore-requests.md)), the branding
([branding.md](references/branding.md)), the usage limits, and the source-built
switch and its limits. Read it first, then change only the fields asked for:

```sh
factory admin policy get --installation <host> --json
factory admin policy set letMaxSeconds=30d retentionDays=30 --installation <host> --json
```

`<host>` is the Front Door's host (`artifacts.example.com`), the one `factory
login` signed in to; with no `--installation` the Installation signed in to
last is used. No folder is read: the policy is the Installation's. `get` prints
every field by its wire name, its value then its default (`--json`: `policy`
and `defaults`, each the whole document). `set` takes `<field>=<value>`
arguments, one PATCH for all of them: a field in seconds takes a count of
seconds or a duration (`15d`, `36h`, `1d12h`); a number a whole number
(`retentionDays=-1` is indefinite); a switch `true`/`false` (`on`/`off`,
`yes`/`no`); text as given; `logoPng` the path of a PNG file, sent as base64,
or an empty value to clear it. Every field, with its default and what it takes:
`factory help admin policy set`, and [commands.md](references/commands.md).

Every change is one write the Control API validates as a whole (an invalid
value is `invalid_request`, 7, naming the field, and nothing is written) and
records in the Installation activity record (field, from, to). The answer lists
what changed, as `Changed <field> from <from> to <to>.`; a value the policy
already has writes nothing. Changing `letMaxSeconds`, `maxIdleRenewals` or
`retentionDays` rewrites the Revisions bucket's lifecycle rule, and the result
says what it now deletes, or that it could not be rewritten (a warning on
stderr; the policy stands). `expiryWarningLeadSeconds` (2d) is how early the
Publisher is mailed that a Let ends; a shorter Let is not warned. A non-Admin
gets `not_admin` (34).

Nine fields are source-built apps' (ADR-0009; the spec's section 5.12).
`sourceBuiltAllowed` (false) allows or pauses publishing `streamlit` and
`python` apps once the Operator has installed the capability
(`--source-built`, section 3): on, such publishes are admitted; off, they
are refused with `source_built_disabled` (36) and the live ones stay. It
installs nothing: while the capability is not installed the refusal holds
whatever the switch says, and the message says so. The limits
`sourceBuiltVcpu` (1, 1 to 8), `sourceBuiltMemoryMiB` (1024, 128 to 32768),
`sourceBuiltMaxInstances` (1), `sourceBuiltMinInstances` (0, up to the max),
`sourceBuiltRequestTimeoutSeconds` (3600, at most 3600) and
`sourceBuiltBuildTimeoutSeconds` (600, up to 86400) bound every source-built
app's service and build; `sourceBuiltEgress` (`none`, or `internet`) is one
Installation-wide setting, `none` being the no-egress network;
`scanThreshold` (`CRITICAL`, or `HIGH` to tighten it) is the scan gate's: a
fixable finding at or above it in the app's own packages, or a
known-malicious package, fails the publish (`scan failed`); the scan cannot
be turned off. Base-image findings are Google's to patch and never fail a
publish; the full report, base image included, is an Admin's to read through
the Control API's `GET /api/v1/artifacts/{id}/revisions/{rev}/scan` (no CLI
command yet). A change applies to the Revisions published afterwards, never
to a running service: closing egress for what is live means ending those
Lets. Change them on the Admin's say-so with the values they gave.

Three fields are the Usage limits (ADR-0011): `staticMaxInstances` (3),
the max instances of a static Artifact's Cloud Run service;
`liveArtifactsPerPublisher` (50), the live Artifacts one Publisher may
hold; and `publishesPerPublisherPerDay` (100), the publishes one Publisher
may make per UTC day, Restores included. A changed `staticMaxInstances`
applies to the Revisions deployed after it, never to a running service.
Raising `liveArtifactsPerPublisher` or `publishesPerPublisherPerDay` is
the Admin's way out of `publisher_at_limit` (17) or `publish_rate_limited`
(18): the messages name the field, and the new limit holds from the next
publish, Restore or Transfer. Raise one on the Admin's say-so, with their
number; `status`'s usage line (section 6) shows who is near or at a limit.
`installation_paused` (19) is no limit of the Admin's: the Operator's
Spend cap is enforced until the Operator lifts it or the month ends.

## 6. The Installation's status and stored config

```sh
factory admin status --installation <handle> --json
factory admin config get --installation <handle> --json
```

`config get` prints the config the install stored, changing nothing: every
input as stored (permanent, then mutable; `unset` for one not stored, with the module's
default for `artifactDomain` and the retention), the organization, the project IDs (the build project's once
`--source-built` has been on), the OAuth client's ID and, once the install
created it, the Spend cap's budget (`Spend cap budget: <name>`;
`spendCapBudget` in `--json`); with `--json` the one object the state
bucket holds, with no secret. It needs only the Operator's `gcloud` login,
like `install`, so an agent may run it. There is no `config set`: to
change a mutable input, hand the Operator the `factory admin install
--installation <handle> <flag>` run (section 3).

`status` reports, changing nothing: the release the installer's ledger
records, whether the Installation is ready, each Handoff's status, the
services' readiness, the capacity against the quota read live from Google
(`warning` at 80 % and up, `warnedAt` once the group is mailed), and the
Spend cap: `Spend cap: 500 USD, configured (budget <name>).` (the state
is `configured`, `enforced: the Installation is Paused` or `lifted for
the rest of the month`), or `Spend cap: unset; the Installation has no
money ceiling.`, or `unset;` with why the stored budget cannot be shown
(gone, or not created yet: re-run the install). In `--json`, `spendCap`
is `{"set", "amount", "currencyCode", "budget", "state"}` or `{"set":
false}` with an optional `detail`. It reads two places, so it needs both:
the ledger (and the budget, by its stored name) in the state bucket, with
the Operator's `gcloud` login as `install` reads it (the handle is the
control-plane project ID), and the Control API's status call as the
member signed in to the Installation's host, who must be an Admin
(`factory login <host>` first; `unauthenticated` without a Sign-in,
`not_admin` for a member outside the Admin group). It also says whether
email is on: `Email: on, sent as <mailbox>` (the `email` object in
`--json`), or `Email: off` with the reason (no sender configured, `email`
null; or the `email` Handoff pending: the test mail not tried yet, Google
refused the delegation, or the sweep could not send); until it is on, tell
the Admin that nothing is mailed. And the usage per Publisher (ADR-0011),
in counts and states, never money: one line, `Usage per Publisher:
<email> <n> live, <n> today, <state>; … (limits <n> live Artifacts, <n>
publishes per UTC day; today's counts reset <local time>).`, the state
`ok`, `near limit` (within 10 % of either limit) or `at limit`, or `Usage
per Publisher: nobody has published yet (limits …).`; in `--json`,
`usage` is `{"limits": {"liveArtifactsPerPublisher",
"publishesPerPublisherPerDay"}, "resetsAt", "publishers": [{"publisher",
"liveArtifacts", "publishesToday", "state"}]}` with the state as `ok`,
`near_limit` or `at_limit` and `publishers` `[]` when nobody has
published. A Publisher at a limit is refused their next publish (section
5 says how to raise it). And `Serving: yes.`, or `Serving: Paused since
<time>: the Spend cap is enforced (budget read …)` (`serving` in `--json`:
`paused`, `since`, `budgetState`, `budgetReadAt`; `notAvailable` is `[]`).

## 7. Rebase

`factory admin rebase --installation <handle> [--json]` runs the upgrade's
rebase pass again. It is the Operator's, like the install: it copies
bundles and enqueues deploys with the Operator's own `gcloud` login, so
hand them the command line. Run it when an upgrade stopped during its
`rebase` step (exit 28, `install_failed`; the re-run resumes, skipping
the Artifacts already on the base), or after an upgrade that skipped a
release whose base image changed. Read the counts it prints (`result` in
`--json`): `live`, `rebased` (redeploying now), `resumed` (still
deploying from an earlier pass), `current` (on the base already),
`inProgress` (a publish in progress, or a stale lock the sweep times out;
left alone) and `notLive`. The command waits a second between deploys, so
it runs for about as many seconds as there are live Artifacts; Ctrl-C is
safe and the re-run resumes. An
`install_failed` (28) whose message says to run `factory admin install`
first means the Installation was last applied by a release before the
pass existed; the install from this release makes the pass itself when
the base image changed. An older `factory` is refused with
`older_release` (30), as the install refuses it.

## 8. Uninstall

```sh
factory admin uninstall --installation <handle> --confirm <workspace-domain>
```

Removes the Installation: the Operator's, run by them. It deletes the
projects the install created, the three and the build project when
`--source-built` made one (Google keeps each recoverable for about 30
days, `gcloud projects undelete <id>`, and the command prints the
commands; nothing is exported first), or, when the Operator pre-created
the projects, empties them of the Installation's resources and leaves them
in place (the report says so, and there is nothing to undelete); the Spend
cap's budget is deleted either way. It never prompts: `--confirm` must be
the Installation's Workspace domain exactly as `config get` prints
`workspaceDomain`, and a missing or wrong value, an unknown Installation
or one already uninstalled is `usage` (2) with nothing changed. The
report ends with two cleanups only a person can do, each saying whether
there is anything to do: revoke the email sender's domain-wide delegation
in the Admin console (named by its client ID) and remove the four DNS
records (listed). Forward both. A step that fails is `install_failed`
(28) naming it; the same command again finishes, and what is deleted
stays deleted.

## 9. Transfer and restore for another Publisher

Both are an Admin's actions through the Installation's Control API, which
checks the caller against the Admin group itself: a caller who is not a
member gets `not_admin` (34), whatever they own. Run them only on an
Admin's say-so, with the exact Artifact ID and address they gave; the
address must be a member of the Workspace domain.

- **`factory transfer <id> <email>`** makes that member the Artifact's
  Publisher, in whatever state the Artifact is (live, Expired or never
  live). Nothing else moves: the address, the Let, the Viewers, the
  Revisions. `invalid_request` (7) for an address outside the domain or
  the current Publisher's; `deploy_in_progress` (10) while a publish of it
  is in flight; `publisher_at_limit` (17) when the Artifact is live and
  the receiver already holds the Installation's limit of live Artifacts
  (ADR-0011): the message names their count and the way out, `factory
  admin policy set liveArtifactsPerPublisher=<n>`, or one of theirs
  ending or expiring first; an Expired or never-live Artifact is never
  refused so. Output: `Transferred Artifact <id> to <email>; <url>.`, or
  with `--json` `{"artifactId", "url", "state", "publisher": {"email",
  "name"}}`.
- **`factory restore <id> --publisher <email>`** restores an Expired
  Artifact and gives it to that member in one transaction: it comes back
  live as theirs, from the bundle the Installation retained, and the
  command waits and reports exactly as `factory publish` does. Without
  `--publisher`, an Admin's `factory restore <id>` restores any member's
  Artifact for its Publisher, even when the policy does not let
  Publishers restore their own. The Restore's refusals: `not_expired`
  (31), `nothing_retained` (38); `deploy_in_progress` (10) and
  `installation_at_capacity` (11) as for a publish; and the Usage limits
  of the Publisher who receives it (the named one, else the Artifact's),
  since a Restore counts as their publish and their live Artifact:
  `publisher_at_limit` (17) and `publish_rate_limited` (18), which an
  Admin lifts by raising the limit (section 5); `installation_paused` (19).

When the Installation's email is on, both commands mail the Publisher in
the Admin's name (a Transfer mails both Publishers; never the Admin about
their own action) and print one stderr line naming who was owed a mail
and what became of it (`The Installation emailed <a> and <b> in your
name.`, `… could not email … (the failures are logged there); tell them
yourself.`, or `Email is off on this Installation, so … not emailed; tell
them yourself.`; `mail` and `mailTo` in `--json`). Report it as it is: a
Publisher not mailed is the Admin's to tell. No prompt, nothing local.

## 10. Ending another Publisher's Artifact

```sh
factory admin end --publisher <email> --published-before 30d --json
factory admin end --publisher <email> --published-before 30d --yes --reason "<why>" --json
factory admin end <id>... --yes --reason "<why>" --json
```

`factory admin end` ends the Let of any Artifact, as an Admin: the link goes
down at once, the Artifact stays its Publisher's, keeps its address and may
be restored or published again by them, and your name and the reason go on
its activity record, where the Publisher reads them; with email on each
Publisher is emailed in your name with the reason and what they may do next:
each `ended` row carries `mail` (`sent`, `failed`, `off`) and the report's
last line sums them up (each emailed; email off, so none; or how many could
not be). Report that line as it is: a Publisher not emailed is the Admin's
to tell. Only an Admin may run it; anyone else gets `not_admin` (34) before
anything is ended. Run it only on an Admin's say-so, and **always list
first**: without `--yes` it ends nothing and prints one `listed` line per
Artifact (`"artifacts": [{"artifactId", "title", "publisher", "kind", "url",
"letStartedAt", "outcome": "listed"}]`, `"applied": false`). Show that list
and get the Admin's yes before the run with `--yes`.

- **Name IDs** (`factory admin end <id>...`) or **filter**, never both:
  `--kind static|streamlit|python|source-built`, `--publisher <email>`
  and `--published-before <when>` combine (`<when>` is a date,
  `2026-10-01`, an RFC 3339 instant, or an age, `7d`; it compares with
  the start of the Artifact's current Let), or `--all` alone is every
  live Artifact. `--reason "<why>"` is optional; give the Admin's words.
- **With `--yes`** every Artifact is tried and the object reports each
  `outcome`: `ended` (with `expiredAt`), `skipped` (a publish of it is in
  flight, `deploy_in_progress`; its `message` says when the lock lapses:
  run the command again for it after that) or `failed` (any other
  refusal, with `code` and `message`; `not_live` means the named
  Artifact is not live, or expired on its own meanwhile, `not_found`
  that the ID is not on this Installation; the listing shows both too).
  `counts` has the totals. Any skip or failure exits `not_all_ended`
  (41) with the same object beside `error`: report every line, and retry
  only the skipped ones, once.
- To keep an Artifact down, the Admin transfers it to themselves first
  (section 9), then ends it; the command's last line says so. Never do
  that unasked.

## 11. Reading a failure

Every failure is `{"error": {"code": "…", "message": "…"}}` with `--json`
(exit code = the code's number), or `factory: <message>` on stderr. The
installer's own codes are 25 to 30: `preflight_failed`, `handoff_pending`,
`preflight_unknown`, `install_failed` (a step failed; re-running resumes),
`signature_invalid` (a release image's signature did not verify; stop and
report it) and `older_release`, and `permanent_input` (37) is its refusal
of a permanent input given with another value (section 3);
`concurrent_run` (39) is a run that another run of the installer
overtook: a config or ledger write was refused on its generation, the
message names the object and the other writer, nothing more was changed,
and the Operator waits for that run before re-running; `factory admin
rebase` exits with `install_failed` and `older_release` too (section 7),
and `factory admin uninstall` with `install_failed` when a deletion step
fails (section 8). `usage` (2) is a missing or invalid input, an unknown
Installation, an uninstall's missing or wrong `--confirm`, or an
`admin end` with both IDs and a filter, or neither. `not_admin` (34) is
the Admin commands' refusal of a member outside the Admin group.
`not_all_ended` (41) is `factory admin end` reporting that at least one
Artifact was skipped or failed, every one tried, the outcomes in the
object (section 10). `publisher_at_limit` (17) and `publish_rate_limited`
(18) are the Usage limits refusing a `transfer` or a `restore` on the
receiving Publisher's behalf (section 9); the Admin raises the limit
(section 5). `installation_paused` (19) is the Paused refusal (section 6).
Every code, who reports it and when:
[exit-codes.md](references/exit-codes.md).

## 12. Traps

1. **Never run the apply on your own.** It creates projects and spends the
   organization's money; the Operator runs it, from the command line you
   hand them. The same for `factory admin rebase`: it redeploys every live
   Artifact.
2. **A FAIL preflight is the organization's to fix**, not the installer's:
   the report names the check; the Operator or the organization's
   administrators act on it.
3. **No secret ever appears in output.** The OAuth client secret is read
   from a file (or stdin as `-`) and never printed; do not paste it on a
   command line or into a message.
4. **`policy set` writes at once**, with no confirmation and for the whole
   Installation: run `policy get` first, change only the fields asked
   for, and report the `Changed` lines back.
5. **Never set, lower, lift or remove a Spend cap on your own.** The
   amount is the Operator's decision, since it is their money (ADR-0011):
   `--spend-cap` goes on the command line you hand them, never on a run
   you make, and lifting an enforced cap or deleting the budget is theirs
   to do in Google's console. Report the `Spend cap:` line as it is.
6. **Never run `factory admin uninstall` on your own.** It deletes the
   Installation's projects and every Artifact in them, with no export and
   a 30-day undelete as the only way back. Hand the Operator the command
   line with the Workspace domain filled in, and forward the two cleanups
   its report ends with.
7. **Never run `factory admin end --yes` unasked, and never widen a
   filter to make it match.** It takes other people's Artifacts down at
   once. List first (no `--yes`), show the Admin exactly what would end,
   and run with `--yes` only on their word, with their reason.

## Reference

- [commands.md](references/commands.md): `factory admin install`,
  `factory admin config get`, `factory admin end`, `factory admin policy
  get`, `factory admin policy set`, `factory admin rebase`, `factory admin
  status`, `factory admin uninstall`, `factory transfer`, `factory
  restore` and the other commands an Admin or Operator uses, with their
  flags.
- [exit-codes.md](references/exit-codes.md): every exit code.
- `factory help admin install`: its last line is the Operator
  documentation's root at this release, whose Install, Upgrade,
  Uninstall, Admin and Troubleshooting pages and one page per Handoff
  cover the rest.
