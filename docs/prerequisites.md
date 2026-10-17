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
  organization and links each to the billing account.
- **An open billing account**, given by ID (`--billing-account`), with or
  without the `billingAccounts/` prefix.
- **An Admin Google Group** that already exists in the domain
  (`--admin-group`). The installer does not create it; its members are the
  Installation's [Admins](admin.md). If the group is not visible to the
  Operator, preflight reports it as unknown and the install goes on.

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

| Input | Flag | Required | What to decide |
|---|---|---|---|
| `workspaceDomain` | `--workspace-domain` | yes | The Workspace domain; the organization is looked up from it. |
| `billingAccount` | `--billing-account` | yes | The billing account to link the projects to. |
| `region` | `--region` | yes | The Installation's Google Cloud region. **Permanent:** the Firestore database and the buckets are created there. Choose it on purpose. |
| `companyHost` | `--company-host` | yes | The Front Door's host, for example `artifacts.example.com`: where Publishers sign in, where members open the Hub (`https://<companyHost>/app/`) and every shareable link starts. You must be able to create DNS records for it. |
| `artifactDomain` | `--artifact-domain` | no | The parent of every Artifact's own host, `<id>.<artifactDomain>`. Default `a.<companyHost>`. You must be able to create a wildcard record under it. |
| `adminGroup` | `--admin-group` | yes | The Admin Google Group's email. |
| `artifactCapacity` | `--artifact-capacity` | no | How many Artifacts to request Cloud Run quota for. Default 4000. |
| `projectPrefix` | `--project-prefix` | no | The prefix of the three project IDs. Default `relay-factory`. |

The inputs can be flags, or a JSON file given with `--config` whose keys
are the first column; a flag wins over the file. The installer never
prompts. In this release every input is fixed by the first run: a later
run that gives any of them a different value is refused. `workspaceDomain`,
`region`, `companyHost`, `artifactDomain` and `projectPrefix` stay
permanent for good; the others become changeable in a later release
([Upgrade](upgrade.md#what-this-release-does-not-do-yet)).

The three projects are `<prefix>-cp-<suffix>` (the control plane, whose ID
is the Installation's handle), `<prefix>-art-<suffix>` (the Artifacts) and
`<prefix>-ntf-<suffix>` (the notifier), with one random six-character
suffix.

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
| `artifact-capacity` | Someone with Owner on the Artifacts project, usually the Operator: a Cloud Run quota request. Never holds the install up. | [Artifact capacity](handoffs/artifact-capacity.md) |
| `invoker-iam` | An Organization Policy Administrator, only when the policy is enforced. | [Invoker IAM](handoffs/invoker-iam.md) |

Allow an hour for the DNS Handoff: Google issues the two certificates
about 20 to 25 minutes after the records resolve, and TLS answers a few
minutes after that.
