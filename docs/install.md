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
gh release download v0.1.0-rc.6 --repo Identivize/relay-factory --pattern 'factory_*_darwin_arm64.tar.gz' --pattern checksums.txt --pattern checksums.txt.sigstore.json
```

The checksums file is signed by the release workflow (cosign keyless): the
certificate identity names the workflow at this tag. Verify the signature,
then the archive against the checksums.

```sh
cosign verify-blob \
  --bundle checksums.txt.sigstore.json \
  --certificate-identity 'https://github.com/Identivize/relay-artifactory/.github/workflows/release.yml@refs/tags/v0.1.0-rc.6' \
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
The report opens with the plan: the three projects the run would create
(named with a `<suffix>` placeholder until the run chooses one), the state
bucket, the two Terraform phases, the six images to verify and copy, the
registry's migrations and policy fields, and the Handoffs to issue. Then
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
naming the three projects, and applies it with no confirmation. On an
existing Installation the plan refreshes Terraform's view of it, which
takes a minute or two before anything starts. The
control-plane project's ID is in that plan and printed again as the run
begins: **write it down.** It is the Installation's handle,
`<prefix>-cp-<suffix>`, and every later run takes it as
`--installation`. The run then creates the other two projects,
the state bucket, applies the Terraform module's first phase, verifies and
copies the release's images into your own registries, brings the registry
to this release's schema, seeds the policy, applies the second phase and
issues the Handoffs. It prints progress on stderr and ends with a report.

A first run ends with **exit 26**: the Installation exists, but the
Handoffs are pending. The report lists each one with the instructions to
forward and a link to its page. Four are always issued, and a fifth
only when the policy requires it:

| Handoff | Who completes it | Blocks ready |
|---|---|---|
| `dns` | The DNS administrator: two `_acme-challenge` CNAMEs and two A records. [DNS](handoffs/dns.md) | yes |
| `oauth-client` | Usually you: a Desktop app OAuth client in the control-plane project. [OAuth client](handoffs/oauth-client.md) | yes |
| `groups-reader` | A Google Workspace super admin: the Groups Reader role on the Installation's two service accounts. [Groups Reader](handoffs/groups-reader.md) | no, but the run ends pending (exit 26) until it is done |
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
`Installation <handle> (release v0.1.0-rc.6) is ready.` with every check `ok`,
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
was changed or removed outside the installer, or an image is gone from a
registry. The run prints a `Warning:` line naming what no longer matches,
puts it back, and still exits 0; the report's `plan.drift` lists the same.
Run `--check` first to see it without changing anything.

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
| `--config <file>` | a JSON object of inputs, keyed workspaceDomain, billingAccount, region, companyHost, artifactDomain, adminGroup, artifactCapacity, projectPrefix, spendCap, viewingRecords, viewingRecordsRetentionDays, sourceBuilt, emailSender |
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
| `--project-prefix <prefix>` | prefix of the three project IDs; permanent (default relay-factory) |

**Inputs of later releases (mutable; stored, not yet effective)**

| Flag | Meaning |
|---|---|
| `--spend-cap <amount>` | the monthly Spend cap, a whole number of the billing account's currency units; 0 is no cap |
| `--viewing-records` | keep viewing records (--viewing-records=false turns them off) |
| `--viewing-records-retention-days <days>` | how long viewing records are kept |
| `--source-built` | install source-built apps (--source-built=false turns them off) |
| `--email-sender <address>` | the mailbox the Installation sends mail as |

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
