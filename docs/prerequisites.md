# Prerequisites

What an Operator needs before running `factory admin install`. The installer
checks most of this itself in its preflight and reports what is missing
without changing anything (`--check`), so this page is what to have ready,
not a form to fill in. Nothing here is specific to one organization: the
installer has no built-in domain, organization or owner.

## The organization and the account

- **A Google Workspace domain with a Google Cloud organization.** The
  installer looks the organization up from the Workspace domain
  (`--workspace-domain`). Viewers of every Artifact are the domain's
  members, signed in with Google.
- **An Operator account in that domain** with
  `resourcemanager.projects.create` on the organization (Project Creator)
  and `billing.resourceAssociations.create` on the billing account (Billing
  Account User). The installer creates three projects under the
  organization and links each to the billing account (four with
  `--source-built`: [Install](install.md#source-built-apps)). An Operator
  who may not create projects can instead be **Owner** of existing ones
  and name them ([Pre-created projects](#pre-created-projects) below);
  the Billing Account User role is needed either way.
- **An open billing account**, given by ID (`--billing-account`), with or
  without the `billingAccounts/` prefix.
- **An Admin Google Group** that already exists in the domain
  (`--admin-group`). The installer does not create it; its members are the
  Installation's [Admins](admin.md). If the group is not visible to the
  Operator, preflight reports it as unknown and the install goes on.
- **A sender mailbox, if the Installation is to send email**
  (`--email-sender`, optional): a licensed user of the Workspace account,
  such as `relay-factory@example.com`, created by a Workspace admin before
  or after the install. The Installation sends every mail as it (the Share
  mail, the expiry mails, the Admin-change mails), and only as it; a Workspace
  super admin then completes the [Email](handoffs/email.md) Handoff.
  Without a sender the Installation runs with no email, and every command
  that would mail says so.

## The machine

- **`gcloud`**, installed and signed in as the Operator (`gcloud auth
  login`). The Operator's gcloud login is the installer's only credential:
  it mints the access tokens for Google's APIs and for Terraform, and uses
  no service account and no Application Default Credentials.
- **A supported machine:** macOS, Linux or Windows on amd64 or arm64. The
  release ships one `factory` archive per pair.
- **Nothing else.** The installer downloads its own pinned Terraform,
  verifies the release's image signatures itself, and copies the images
  with the Operator's token. `cosign` is needed only to verify the `factory`
  archive before running it ([Install](install.md#1-get-factory)), and
  `gh` or `curl` to fetch it.

## Names and a region

| Input | Flag | Required | Permanent | What to decide |
|---|---|---|---|---|
| `workspaceDomain` | `--workspace-domain` | yes | yes | The Workspace domain; the organization is looked up from it. |
| `billingAccount` | `--billing-account` | yes | no | The billing account to link the projects to. |
| `region` | `--region` | yes | yes | The Installation's Google Cloud region: the Firestore database and the buckets are created there. Choose it on purpose. |
| `companyHost` | `--company-host` | yes | yes | The Front Door's host, for example `artifacts.example.com`: where Publishers sign in, where members open the Hub (`https://<companyHost>/app/`) and every shareable link starts. You must be able to create DNS records for it. |
| `artifactDomain` | `--artifact-domain` | no | yes | The parent of every Artifact's own host, `<id>.<artifactDomain>`. Default `a.<companyHost>`. You must be able to create a wildcard record under it. |
| `adminGroup` | `--admin-group` | yes | no | The Admin Google Group's email. |
| `artifactCapacity` | `--artifact-capacity` | no | no | How many Artifacts to request Cloud Run quota for. Default 4000. |
| `projectPrefix` | `--project-prefix` | no | yes | The prefix of the project IDs the run creates. Default `relay-factory`. Not with pre-created projects. |
| `controlPlaneProject` | `--control-plane-project` | no | yes | A pre-created project to adopt as the control plane; its ID becomes the Installation's handle. With the next two. |
| `artifactsProject` | `--artifacts-project` | no | yes | A pre-created project to adopt as the Artifacts project. |
| `notifierProject` | `--notifier-project` | no | yes | A pre-created project to adopt as the notifier project. |
| `buildProject` | `--build-project` | no | yes | A pre-created project to adopt as the build project of source-built apps, with `sourceBuilt`. Checked on every run; the one permanent input a later run may add to an Installation that has none. |
| `spendCap` | `--spend-cap` | no | no | The monthly [Spend cap](install.md#spend-cap) on Cloud Run in the Artifacts project, a whole number of the billing account's currency units. Without one the Installation has no money ceiling. `0` is no cap. |
| `viewingRecords` | `--viewing-records` | no | no | Keep [viewing records](install.md#viewing-records): who opened which Artifact, and when, in BigQuery in the Artifacts project, read through the datasets' sharing, which is yours. Off by default; `--viewing-records=false` removes them and keeps the datasets. |
| `viewingRecordsRetentionDays` | `--viewing-records-retention-days` | no | no | How many days the viewing records are kept. Default 400. |
| `sourceBuilt` | `--source-built` | no | no | Install [source-built apps](install.md#source-built-apps): the build project and its resources, the no-egress network. Off by default; `--source-built=false` removes the resources and keeps the project. |
| `emailSender` | `--email-sender` | no | no | The mailbox the Installation sends mail as (above). Unset means no email. |

The inputs can be flags, or a JSON file given with `--config` whose keys
are the first column; a flag wins over the file. The installer never
prompts. A **permanent** input names the Installation: a later run that
gives it a different value is refused (exit 37, `permanent_input`) and
changes nothing. A **mutable** one configures it: a later run that gives
it a new value applies the value and then stores it
([Upgrade](upgrade.md#changing-a-mutable-input)).

The three projects are `<prefix>-cp-<suffix>` (the control plane, whose ID
is the Installation's handle), `<prefix>-art-<suffix>` (the Artifacts) and
`<prefix>-ntf-<suffix>` (the notifier), with one random six-character
suffix, and with `--source-built` a fourth, `<prefix>-bld-<suffix>` (the
build project); or the ones you name (below).

## Pre-created projects

If the Operator is not a Project Creator on the organization, someone who
is creates three empty projects under it (any IDs, any display names) and
makes the Operator their **Owner** (the Owner role on each project). The
Operator then names them: `--control-plane-project`,
`--artifacts-project` and `--notifier-project` together, instead of
`--project-prefix`. The installer adopts them as they are: it creates no
project, links each to the billing account, enables its APIs and applies
the module into them. The names are permanent, like the other names of
the Installation, and the control-plane project's ID is the handle. A
fourth, `--build-project`, is the build project of
[source-built apps](install.md#source-built-apps): name it with
`--source-built`, on the first run or on the later run that turns the
switch on, and the installer adopts it like the three; named without the
switch it is checked and stored, and nothing is put in it.

Preflight checks, for each named project, that it exists with the Operator
as its Owner (`project-owner`) and that it is under the Workspace domain's
organization, through any folder (`project-parent`). For the control-plane
project it also checks the **Firestore location caveat**
(`firestore-location`): the install creates the Installation's Firestore
database as the project's `(default)` database, in the region, and a
project's `(default)` database and its location of default Google Cloud
resources (set by an App Engine application) can never move. A project
that already has a `(default)` database, or has fixed its default
resources somewhere other than the region, cannot hold the Installation:
name another, or install in that location. The reads need the project's
Firestore and App Engine Admin APIs, which the run enables before
reading again and before applying anything; a `--check` before that
reports the check as unknown. Use projects nothing else has used.

Two people cannot run the installer on one Installation at once: every
write of its config and ledger is checked against what the run read, and
a run that another run overtakes stops with exit 39
([Troubleshooting](troubleshooting.md#exit-39-concurrent_run)).

## Organization policies the preflight checks

The installer reads these organization policies on the organization before
any project exists and on the control-plane project afterwards. A policy it
cannot read is reported as unknown and the install goes on; a policy that
denies what the Installation needs is a failure before anything is created.

| Check | Fails when |
|---|---|
| `region` | `gcp.resourceLocations` denies the chosen region. |
| `cross-project-lb` | `compute.restrictCrossProjectServices` denies every cross-project backend (the one load balancer sits in the control-plane project and serves the Artifacts project). |
| `global-lb` | `compute.disableGlobalLoadBalancing` is enforced. |
| `ingress` | `run.allowedIngress` excludes `internal` or `internal-and-cloud-load-balancing`. |
| `invoker-iam` | Not a failure: when `run.managed.requireInvokerIam` is enforced, the control-plane project needs an exemption, the [Invoker IAM Handoff](handoffs/invoker-iam.md). |

## People who complete the Handoffs

An install pauses on Handoffs: things someone must do in a console the
installer has no right to touch. Each is a page you can forward, and the
installer verifies each one rather than trusting a reply.

| Handoff | Who | Page |
|---|---|---|
| `dns` | Whoever manages DNS for the company host and the Artifact domain: four records. | [DNS](handoffs/dns.md) |
| `oauth-client` | Someone with Owner on the control-plane project, usually the Operator: a Desktop app OAuth client. | [OAuth client](handoffs/oauth-client.md) |
| `groups-reader` | A Google Workspace super administrator: the Groups Reader role on the Installation's two service accounts, so the services can tell who is an Admin. Does not hold readiness up, but the run exits pending until it is done. | [Groups Reader](handoffs/groups-reader.md) |
| `email` | A Google Workspace super administrator, only with `--email-sender`: domain-wide delegation of the Installation's sender account for the one `gmail.send` scope, so the Installation can send as the mailbox. Does not hold readiness up, but the run exits pending until the test mail has gone out. | [Email](handoffs/email.md) |
| `artifact-capacity` | Someone with Owner on the Artifacts project, usually the Operator: a Cloud Run quota request. Never holds the install up. | [Artifact capacity](handoffs/artifact-capacity.md) |
| `invoker-iam` | An Organization Policy Administrator, only when the policy is enforced. | [Invoker IAM](handoffs/invoker-iam.md) |

Allow an hour for the DNS Handoff: Google issues the two certificates
about 20 to 25 minutes after the records resolve, and TLS answers a few
minutes after that.
