# Install

How an Operator stands up an Installation with `factory admin install`.
Have the [Prerequisites](prerequisites.md) ready. The installer runs one
convergent sequence on every run: it creates what is missing, verifies
what is pending and changes nothing that is already right, so re-running
it after a failure, an interruption or a Handoff is always safe. Every
run is the same command; `--check` is the same command changing nothing.

## 1. Get `factory`

Download the archive for your machine, the checksums file and its
signature from the release, then verify them. Replace `darwin_arm64` with
your OS and architecture (`darwin_amd64`, `linux_amd64`, `linux_arm64`,
`windows_amd64.zip`, `windows_arm64.zip`).

```sh
gh release download v0.1.2 --repo Identivize/relay-factory --pattern 'factory_*_darwin_arm64.tar.gz' --pattern checksums.txt --pattern checksums.txt.sigstore.json
```

The checksums file is signed by the release workflow (cosign keyless): the
certificate identity names the workflow at this tag. Verify the signature,
then the archive against the checksums.

```sh
cosign verify-blob \
  --bundle checksums.txt.sigstore.json \
  --certificate-identity 'https://github.com/Identivize/relay-artifactory/.github/workflows/release.yml@refs/tags/v0.1.2' \
  --certificate-oidc-issuer https://token.actions.githubusercontent.com \
  checksums.txt
```

```sh
sha256sum --ignore-missing -c checksums.txt
```

(`shasum -a 256 --ignore-missing -c checksums.txt` on macOS.) Then unpack
and put `factory` on your `PATH`:

```sh
tar xzf factory_*_darwin_arm64.tar.gz
```

The archive holds `factory`, `LICENSE` and `THIRD_PARTY_NOTICES`. The
binary is not notarized for macOS: a browser download is quarantined,
while `gh`, `curl` and Homebrew downloads are not. Use the release's own
`factory` for the whole install: a development build is refused by the
`release` check. The installer installs the release it was built from,
and nothing else.

## 2. Check before changing anything

```sh
factory admin install --check --workspace-domain example.com --billing-account <billing-account> --region <region> --company-host artifacts.example.com --admin-group <admin-group>
```

`--check` runs the preflight checks, plans the run and changes nothing.
The report opens with the plan: the projects the run would create (three,
or four with `--source-built`: [Source-built apps](#source-built-apps)),
named with a `<suffix>` placeholder until the run chooses one, or adopt
([Pre-created projects](#pre-created-projects) below), the state
bucket, the two Terraform phases, the six images to verify and copy, the
registry's migrations and policy fields, what it would do with the
[Spend cap](#spend-cap), and the Handoffs to issue. Then
each check as `ok`, `fail`, `handoff` or `unknown`. Before the first run
the Handoffs are all still to come, so the only two exits are:

- **26** (`handoff_pending`): no check failed. Go on to the first run,
  which issues the Handoffs. Read the report first: a check that is
  `unknown` could not be made yet, which is usual before any project
  exists (there is no project to count API calls against, so organization
  policy and the Admin group are read against your own gcloud project and
  are unknown when it lacks the API); the first run reads them again on
  the new project. `invoker-iam` as `handoff` means your organization
  enforces `run.managed.requireInvokerIam` and an Organization Policy
  Administrator will be needed ([Invoker IAM](handoffs/invoker-iam.md)).
- **25** (`preflight_failed`): a check failed; the message says which. Fix
  it and check again. [Troubleshooting](troubleshooting.md#exit-25-preflight_failed).

On an existing Installation (`--check --installation <handle>`) the same
command also re-verifies the Handoffs from what the ledger holds, plans
the Terraform module against the Installation's state (what the apply
would add, change or destroy, resource by resource) and reads the Front
Door, and exits 0 when no check failed and no Handoff other than
`artifact-capacity` is pending, 26 when one is, and 27
(`preflight_unknown`) when nothing failed or is pending but a check could
not be made. A plan that cannot be made, because a read failed or
Terraform could not plan, exits 28 with the error; nothing was changed.

## 3. First run

The same command without `--check`:

```sh
factory admin install --workspace-domain example.com --billing-account <billing-account> --region <region> --company-host artifacts.example.com --admin-group <admin-group>
```

The run prints its plan first, the same summary `--check` gave, now
naming the projects (created, or adopted when you named them), and applies it with no confirmation. On an
existing Installation the plan refreshes Terraform's view of it, which
takes a minute or two before anything starts. The
control-plane project's ID is in that plan and printed again as the run
begins: **write it down.** It is the Installation's handle,
`<prefix>-cp-<suffix>`, and every later run takes it as
`--installation`. The run then creates the other two projects,
the state bucket, creates the [Spend cap](#spend-cap)'s budget when you
gave one, applies the Terraform module's first phase, verifies and
copies the release's images into your own registries, brings the registry
to this release's schema, seeds the policy, applies the second phase and
issues the Handoffs. It prints progress on stderr and ends with a report.

A first run ends with **exit 26**: the Installation exists, but the
Handoffs are pending. The report lists each one with the instructions to
forward and a link to its page. Four are always issued, a fifth with
`--email-sender`, and a sixth only when the policy requires it:

| Handoff | Who completes it | Blocks ready |
|---|---|---|
| `dns` | The DNS administrator: two `_acme-challenge` CNAMEs and two A records. [DNS](handoffs/dns.md) | yes |
| `oauth-client` | Usually you: a Desktop app OAuth client in the control-plane project. [OAuth client](handoffs/oauth-client.md) | yes |
| `groups-reader` | A Google Workspace super admin: the Groups Reader role on the Installation's two service accounts. [Groups Reader](handoffs/groups-reader.md) | no, but the run ends pending (exit 26) until it is done |
| `email` | A Google Workspace super admin, when `--email-sender` names a mailbox: domain-wide delegation of the sender account, by the client ID in the report. [Email](handoffs/email.md) | no, but the run ends pending (exit 26) until the test mail has gone out |
| `artifact-capacity` | Usually you: a Cloud Run quota request in the Artifacts project. [Artifact capacity](handoffs/artifact-capacity.md) | no |
| `invoker-iam` | An Organization Policy Administrator, when `run.managed.requireInvokerIam` is enforced. [Invoker IAM](handoffs/invoker-iam.md) | yes, the second phase waits for it |

Forward each page with the values from your report. The installer does
not trust a reply: it verifies each Handoff against the world on every
run, and a Handoff that stops verifying goes back to pending.

If the first run dies before it has written its config (right after the
first project), re-run it with the same inputs plus `--installation <the
printed ID>` to adopt that project instead of creating another one:
[Troubleshooting](troubleshooting.md#adopting-a-half-created-project).

## 4. Complete the OAuth client Handoff

Once the Desktop app OAuth client exists
([OAuth client](handoffs/oauth-client.md)), hand the installer its ID and
its secret, the secret in a file (mode 0600; `-` reads it from stdin):

```sh
factory admin install --installation <handle> --oauth-client-id <client-id> --oauth-client-secret-file <secret-file>
```

Google is asked before anything changes: a client Google rejects is a
`usage` failure (exit 2) and nothing is stored. An accepted secret is
stored as a version of the Installation's `oauth-desktop-client` secret,
the ID in the config, and the second phase is applied again with it. The
secret is never written to the ledger, the state or any output.

## 5. Resume and wait

Every later run is the handle alone:

```sh
factory admin install --installation <handle>
```

It re-verifies every Handoff and reports. The DNS Handoff takes the
longest: Google issues the two certificates about 20 to 25 minutes after
the CNAMEs resolve in public DNS, and TLS answers a few minutes after the
certificates are active. Rather than re-running by hand, let one run wait:

```sh
factory admin install --installation <handle> --wait 40m
```

`--wait` re-verifies every 30 seconds for up to that long while the DNS
Handoff or the Front Door's readiness is outstanding, then reports. Each
pass verifies every Handoff, but only those two keep the wait going: the
others change only when someone acts, so a run that waits on them alone
ends at once.

## 6. The ready report

The run exits **0** when the Installation is ready: the `dns` and
`oauth-client` Handoffs are verified and the Front Door's discovery
document, `https://<company-host>/.well-known/relay-factory.json`, answers
with the Control API's URL and the OAuth client. The report then says
`Installation <handle> (release v0.1.2) is ready.` with every check `ok`,
every step `done` and each Handoff `done`, except that
`artifact-capacity` may still be pending: it waits on Google's review of
the quota request and never holds the Installation up. Until it is
granted, the Installation holds as many Artifacts as the current quota
allows.

The `groups-reader` Handoff does not gate readiness either, since the
Installation serves Publishers without an Admin, but it does hold the
exit code: while it is pending the run reports `Installation <handle> is
ready; the groups-reader Handoff is pending.` and exits 26. Forward
[Groups Reader](handoffs/groups-reader.md) to a Workspace super admin,
then run again five minutes after they are done: the Control API and the
Hub each check their role at start-up and every five minutes
([Troubleshooting](troubleshooting.md#the-groups-reader-handoff-stays-pending)).
The `email` Handoff, issued only with `--email-sender`, works the same way:
the Installation serves without mail, the run exits 26 while it is
pending, and once the super admin has authorized the delegation the sweep
sends a test mail to the Admin group within five minutes and the next run
reports it done ([Email](handoffs/email.md),
[Troubleshooting](troubleshooting.md#the-email-handoff-stays-pending)).
`factory admin status` says whether email is on.

Publishers can now sign in with `factory login <company-host>`
([Publisher](publisher.md)), and every member of the Workspace domain can
open the Hub at `https://<company-host>/app/` with their Google account
(`https://<company-host>/` leads there too). Tell your Admins the Front Door host, and
read [Admin](admin.md) for what the Admin group is for in this release.

Re-running the command on a ready Installation changes nothing and exits
0: its plan says `nothing to change`, no new project, no new secret
version, images already present are neither re-verified nor re-copied,
and Terraform plans no changes. `--check --installation <handle>` reports
the same from a live read without touching anything.

If the plan of a ready Installation does change something, with no new
input and the same release, that is **drift**: a resource of the module
was changed or removed outside the installer, an image is gone from a
registry, or the Spend cap's budget is gone or no longer at its amount.
The run prints a `Warning:` line naming what no longer matches, puts it
back, and still exits 0; the report's `plan.drift` lists the same. Run
`--check` first to see it without changing anything.

## Pre-created projects

An Operator who is Owner of existing projects but not a Project Creator
names them instead of letting the run create three
([Prerequisites](prerequisites.md#pre-created-projects)):

```sh
factory admin install --check --workspace-domain example.com --billing-account <billing-account> --region <region> --company-host artifacts.example.com --admin-group <admin-group> --control-plane-project <project> --artifacts-project <project> --notifier-project <project>
```

The three go together and `--project-prefix` is not given with them. The
check's `project-owner` and `project-parent` lines say whether each
exists with you as its Owner under the organization, and `project-create`
passes without the Project Creator role; `firestore-location` is unknown
until the run has enabled the control-plane project's APIs, and the run
reads it again then, before it applies anything. The plan says `adopt`:
the run creates no project, links the three to billing, enables their
APIs and applies the module into them. The handle is the control-plane
project you named, and the three names are permanent. A re-run naming
one of them differently is refused (exit 37). With `--source-built` a
fourth pre-created project, `--build-project`, is adopted the same way
([Source-built apps](#source-built-apps)); without it the switch is
refused as `usage`, since the run has no project to build in.

## Source-built apps

Source-built apps (`kind: streamlit` or `kind: python` in a folder's
`factory.yaml`) are off by default: an Installation that never turns
them on carries nothing of them. `--source-built` installs the
capability, on the first run or on any later one:

```sh
factory admin install --check --installation <handle> --source-built
factory admin install --installation <handle> --source-built
```

The run creates a fourth project, `<prefix>-bld-<suffix>` (the build
project; with pre-created projects, the one you name with
`--build-project`), links it to billing and enables Cloud Build, Artifact
Registry and its scanning, and applies the module's source-built
resources: in the build project, a builder account with no rights
outside it, the repository the builds push to, a mirror of PyPI the
builds install packages through and a bucket the builds read their code
from; in the Artifacts project, a second runtime account with no
permissions that every source-built app runs as, and a no-egress network
(a VPC with no Cloud NAT and a rule denying all egress) for the policy's
`sourceBuiltEgress` of `none`. The plan lists the project and the
resources first, as for any run; no Handoff is needed, and the Operator
account needs no role beyond the install's (the Project Creator role
creates the fourth project as it created the three).

Installing the capability does not yet let anyone publish such an app:
an Admin turns the policy's `sourceBuiltAllowed` on
([Admin](admin.md#source-built-apps)), and until both are done a
`streamlit` or `python` publish is refused with `source_built_disabled`,
whose message says which of the two is missing. Once the pipeline is in
place, every image a build pushes is scanned by Artifact Analysis in the
build project, which Google charges per image; the capability enables
that scanning now.

`--source-built=false` on
a later run removes the module's source-built resources (new source-built
publishes are refused again) and keeps the build project, stored in the
config, so that a run that turns the switch on again reuses it and
[`factory admin uninstall`](uninstall.md) deletes it; delete it by hand
before then if you want it gone. Turn it off only while no source-built
Artifact is live: the network those services attach to and the account
they run as go with the resources, and the run fails part way while one
is attached (an Admin ends their Lets first; running again resumes). The
run does not check this for you yet.

## Spend cap

**An Installation without a Spend cap has no money ceiling.** Every
Artifact is its own Cloud Run service, and a coding agent can mint and
republish Artifacts without pause, so the bill is bounded by the live
Artifacts alone. The preflight check `spend-cap`, the plan and
`factory admin status` all say so while no cap is set.

The Spend cap is yours, the Operator's, in money: a monthly amount in
the billing account's currency that caps Cloud Run spend in the
Artifacts project. The other two projects get no cap: their cost is the
fixed load-balancer charge plus near-zero compute. Set it on the first
run or on any later run with `--spend-cap <amount>` (a whole number;
`spendCap` in a `--config` file), and the run creates one Google budget
on the Artifacts project with the cap configured, before anything that
can cost money exists:

```sh
factory admin install --installation <handle> --spend-cap 500
```

A later run with another amount updates the budget; a run without the
flag keeps the stored cap. The budget's name is stored with the
Installation's config (`factory admin config get` prints it), since a
project Owner can read a budget by name but cannot list the billing
account's budgets. The run needs
`billing.resourcebudgets.configureSpendCap` on the Artifacts project,
which the Owner and Editor roles hold; the Operator who created the
project is its Owner, and the `spend-cap` check fails before anything
changes when the permission is missing.

**What Google sends.** The alerts are Google's and cannot be changed: a
mail at 50 %, 80 % and 100 % of the cap, on gross estimated spend, and
one more when the cap is enforced, to the billing account's
administrators and the Artifacts project's Owners. Nothing is sent for a
lift.

**What enforcement does.** When spend reaches the cap, Google pauses
Cloud Run in the Artifacts project for the rest of the month: every
Artifact stops answering. Enforcement follows the crossing by minutes,
not at once, so some spend past the cap is billed. The cap lifts itself
on the first of the next month. Relay Factory reads the budget's state
on every sweep run and records the Installation as Paused within a sweep
interval of the enforcement: the Front Door then answers every Artifact
link with a paused page, the Control API refuses publishes and Restores
with `installation_paused` (Renewals still work), and the Admins are
mailed once each time the state flips, when the Installation has email.
After a lift it records the Installation serving again once two
consecutive reads find the cap no longer enforced, since the Artifacts
answer a few minutes after the lift. `factory admin status` shows both
the budget's state, `enforced: the Installation is Paused`, read from
Google, and the Installation's own, `Serving: Paused since <time>`.

**Raising it mid-month.** An enforced cap cannot be changed until it is
lifted, and Relay Factory never lifts or removes a cap, since a lift
means agreeing to spend more: lift it in Google's billing console, or
by the Budget API, then re-run with the new amount, which re-arms the
cap. A run that gives a new amount while the cap is enforced fails
(exit 28) and says so. `--spend-cap 0` is no cap: it leaves an existing
budget exactly as it is, and the run says so; delete the budget in the
console if you want the ceiling gone.

`factory admin status --installation <handle>` shows the Spend cap as
`Spend cap: 500 USD, configured (budget <name>).`, as `enforced` or
`lifted for the rest of the month`, or as `unset; the Installation has
no money ceiling.` ([Admin](admin.md#what-an-admin-can-see)).

## Viewing records

**Google logs every view, in every Installation.** The install turns on
the Identity-Aware Proxy's audit log in the Artifacts project. Google then
writes one entry for every request to an Artifact, admitted or refused:
the member (nobody, for a request that was not signed in), the Artifact,
whether it was admitted, and also the full address requested, the IP
address and the browser's user agent. The entries stay in the Artifacts
project's `_Default` log bucket for 30 days. Only an account with the
Private Logs Viewer role (`roles/logging.privateLogViewer`) on the
Artifacts project can read them; the Logs Viewer role alone reads
nothing. Relay Factory does not read this log. Members' use of the Hub
is not logged this way.

**Viewing records are your option, off by default.** `--viewing-records`
keeps a thin record of every request for longer, in BigQuery in the
Artifacts project: the time, the Artifact, the member the proxy named
(the Viewer, when admitted) and whether they were admitted, and nothing
else.

```sh
factory admin install --check --installation <handle> --viewing-records
factory admin install --installation <handle> --viewing-records
```

The run enables BigQuery and the BigQuery Data Transfer Service in the
Artifacts project and creates, in the Installation's region, two
datasets: `viewing_routed`, where a log sink (`relay-viewing-records`)
copies the proxy's entries whole, kept two days, and `viewing_records`,
where an hourly scheduled query ("Relay Factory viewing records") copies
the four fields of each entry into the table `records`. The address, IP
address and user agent stay in `viewing_routed` alone, for those two
days. A refused request is a row with `admitted` false; a request that
was not signed in has no Viewer. The query runs as its own service
account, `viewing-records@<artifacts project>`, which can read the one
dataset and write the other and nothing more. A missed run loses
nothing, since each run copies every entry of the last two days not yet
copied. The query's first run is in its first hourly slot at least ten
minutes after it is created (not at once, since its permission takes a
moment to apply), and the `records` table appears with that run, within
about an hour and ten minutes.

**Entries in the sink's first minute or so can be lost.** A new sink, or
its permission to write, takes a short while to take effect, and entries
written in that window can miss BigQuery with no error. The 30-day log
above still has them.

**Retention.** `--viewing-records-retention-days <days>` sets how long
the rows are kept, a whole number of days, 400 when not given. A later
run with another number changes it: the dataset's default at once, and
the table's at the query's next run. A shorter retention deletes the
rows older than it at that run.

**Who can read the rows.** The rows name members, so no part of Relay
Factory reads them: not the Control API, not the Hub and not the deploy
worker. BigQuery gives each new dataset its default sharing: the account
that ran the install that created it is an owner, and so are the
Artifacts project's Owners, while its Editors and Viewers can write and
read. Anyone else reads them only once you grant it. To let someone
read them, open the `viewing_records` dataset in the BigQuery console, choose
**Sharing**, then **Permissions**, and add them with the BigQuery Data
Viewer role (they also need the BigQuery Job User role on a project to
run a query). A BigQuery or basic role on the Artifacts project, such as
Owner, reads them too. Publishers see nothing about viewing.

**What the Hub shows Admins: counts, never who.** With the records on,
the Hub's All Artifacts and Installation pages show an Admin each
Artifact's views, distinct Viewers and last view, and the Installation's
totals ([The Hub's Admin pages](admin.md#the-hubs-admin-pages)). A view
is one Viewer on one Artifact in one UTC day, and only an admitted
request that names a Viewer counts. The run creates a third dataset for
them, `viewing_aggregates`, where a second hourly scheduled query
("Relay Factory viewing aggregates"), run as `viewing-records@`, replaces
two tables from `records` each run: `artifacts`, one row per Artifact,
and `installation`, the totals. They name nobody, and of Relay Factory
the Control API alone reads them, with a read of that one dataset; it
still has no right to the rows. Until the `records` table appears, the query does nothing,
and the counts trail the records by up to an hour.

**Turning them off.** `--viewing-records=false` on a later run removes
the sink, the scheduled queries and their account. The two datasets stay,
with what they hold, until you delete them, and their old rows go on
expiring by the retention; the BigQuery APIs stay enabled. The
`viewing_aggregates` dataset does not stay: it holds only counts derived
from the rows, so it is deleted with its tables, and the aggregates
query's first run rebuilds them when the records are on again. Turning the
records on again picks up the datasets that are still there, by the
label Relay Factory gave them (`relay-factory: viewing-records`); a
dataset of the same name without it is not taken, and the run fails
until you rename or delete it.
[`factory admin uninstall`](uninstall.md) deletes them with the
Installation, except, in pre-created projects, datasets left by turning
the records off: those are yours to delete.

**Cost.** BigQuery storage of the routed entries (about 1.9 KB each, kept
two days) and of the rows, and two small queries an hour, the records'
and the aggregates' (whose few rows of counts cost next to nothing to
store): for an
Installation of a few hundred members, an estimate (not a measurement)
puts it well inside BigQuery's free tier of 10 GiB stored and 1 TiB
queried a month.

## The command

<!-- cliref:begin flags admin install -->

```
factory admin install [--check] [--installation <control-plane-project>]
                      [--config <file>] [input flags]
                      [--oauth-client-id <id> --oauth-client-secret-file <path>]
                      [--wait <duration>] [--json]
```

**Flags**

| Flag | Meaning |
|---|---|
| `--check` | plan the run and change nothing: the preflight checks, the Handoffs, and what the run would create, copy, migrate and apply |
| `--installation <project>` | the control-plane project ID of an Installation to resume or check |
| `--config <file>` | a JSON object of inputs, keyed workspaceDomain, billingAccount, region, companyHost, artifactDomain, adminGroup, artifactCapacity, projectPrefix, controlPlaneProject, artifactsProject, notifierProject, buildProject, spendCap, viewingRecords, viewingRecordsRetentionDays, sourceBuilt, emailSender |
| `--wait <duration>` | keep verifying a pending DNS Handoff for up to this long (default 0: verify once) |
| `--json` | print the report as one JSON object on stdout |

**Input flags**

| Flag | Meaning |
|---|---|
| `--workspace-domain <domain>` | the Google Workspace domain; permanent (required) |
| `--billing-account <id>` | the billing account to link; mutable (required) |
| `--region <region>` | the Installation's region; permanent (required) |
| `--company-host <host>` | the Front Door's host; permanent (required) |
| `--admin-group <email>` | the Admin Google Group; mutable (required) |
| `--artifact-domain <domain>` | parent of each Artifact's host; permanent (default a.&lt;company-host&gt;) |
| `--artifact-capacity <n>` | how many Artifacts to request quota for; mutable (default 4000) |
| `--project-prefix <prefix>` | prefix of the project IDs the run creates; permanent (default relay-factory); not with the pre-created projects |
| `--control-plane-project <id>` | a pre-created project to adopt as the control plane, with --artifacts-project and --notifier-project; permanent |
| `--artifacts-project <id>` | a pre-created project to adopt as the Artifacts project; permanent |
| `--notifier-project <id>` | a pre-created project to adopt as the notifier project; permanent |
| `--build-project <id>` | a pre-created project to adopt as the build project of source-built apps, with --source-built; permanent |
| `--spend-cap <amount>` | the monthly Spend cap on Cloud Run in the Artifacts project, a whole number of the billing account's currency units; mutable (0 is no cap: an existing cap is left as it is) |
| `--source-built` | install source-built apps: the build project and its resources, the no-egress network; mutable (default off; --source-built=false removes them and keeps the project) |
| `--email-sender <address>` | the mailbox the Installation sends mail as; mutable (default none: no email) |

**Viewing records**

| Flag | Meaning |
|---|---|
| `--viewing-records` | install the viewing records: IAP's entries routed to BigQuery, an hourly thin record and its counts; mutable (default off; --viewing-records=false removes them and keeps the two record datasets) |
| `--viewing-records-retention-days <days>` | days the viewing records are kept; mutable (default 400) |

**Completing the oauth-client Handoff**

| Flag | Meaning |
|---|---|
| `--oauth-client-id <id>` | the Desktop app OAuth client's ID |
| `--oauth-client-secret-file <path>` | a file holding its secret; - is stdin |

<!-- cliref:end -->

Every command also prints the result as one JSON object on stdout with
`--json`, on success or failure; the report's shape and every exit code
are in [Troubleshooting](troubleshooting.md). Inputs can come from a file
instead of flags, `--config <file>`, a JSON object keyed by the input
names in [Prerequisites](prerequisites.md#names-and-a-region); a flag wins
over the file.
