# Admin

The Admins of an Installation are the members of its Admin Google Group,
the `adminGroup` the Operator gave at install. Admins own the
Installation's policy: how long a Let lasts, what the expired page shows,
the limits per Publisher, the branding and whether source-built apps are
allowed. Installing and upgrading is the Operator's
([Install](install.md)); publishing is every member's
([Publisher](publisher.md)).

## The policy

The policy is one document the installer seeds at its defaults and never
overwrites: a field already set stays as set across upgrades, and a field
a new release adds appears at its default. Its fields, by the names
`factory admin policy` uses:

| Field | Default | Meaning |
|---|---|---|
| `letDefaultSeconds` | 15 days | How long a new Artifact lives before it expires. |
| `letMaxSeconds` | 15 days | The longest Let a publish may ask for; at least `letDefaultSeconds`. |
| `maxIdleRenewals` | 2 | How many times a Let may be renewed without a new Revision. |
| `retentionDays` | 0 | How long an Expired Artifact's bundle is kept: `0` for none past the sweep that expires it, a number of days, or `-1` for indefinite. |
| `expiryWarningLeadSeconds` | 2 days | How long before a Let ends its Publisher is warned. |
| `publisherSelfRestore` | `true` | A Publisher may restore their own Expired Artifact (`factory restore`); off, their attempt is refused with `restore_disabled` and only an Admin restores it. |
| `restoreRequestButton` | `true` | The expired page offers a Restore Request. |
| `listedByDefault` | `false` | A new Artifact open to the whole domain is Listed. |
| `errorRedirect` | empty | Where the expired and not-found outcomes send a Viewer instead of the built-in pages: an absolute `https` URL of the organization's own. Empty means the built-in pages, which say why the link does not answer. |
| `displayName` | empty | The Installation's display name, one line of at most 80 characters; empty means the Workspace domain. |
| `logoPng` | empty | The PNG logo, at most 64 KiB; `factory admin policy set` takes the file's path. Empty means none. |
| `accentColor` | empty | The accent colour as `#rrggbb`; empty means none. |
| `staticMaxInstances` | 3 | Max instances of a static Artifact's service. |
| `liveArtifactsPerPublisher` | 50 | Live Artifacts one Publisher may hold. |
| `publishesPerPublisherPerDay` | 100 | Publishes one Publisher may make per UTC day. |
| `sourceBuiltAllowed` | `false` | Source-built (`streamlit`, `python`) apps may be published. |
| `sourceBuiltVcpu` | 1 | vCPU of a source-built app's service, 1 to 8. |
| `sourceBuiltMemoryMiB` | 1024 | Its memory in MiB, 128 to 32768. |
| `sourceBuiltMaxInstances` | 1 | Its max instances. |
| `sourceBuiltMinInstances` | 0 | Its min instances, up to `sourceBuiltMaxInstances`. |
| `sourceBuiltRequestTimeoutSeconds` | 3600 | Its request timeout, up to 3600. |
| `sourceBuiltBuildTimeoutSeconds` | 600 | Its build timeout, up to 86400. |
| `sourceBuiltEgress` | `none` | Its egress: `none` or `internet`. |
| `scanThreshold` | `CRITICAL` | The vulnerability finding that fails a build: `CRITICAL` or `HIGH`. |

**Changing the policy.** `factory admin policy get` prints the policy, one
field per line with its value and its default, and
`factory admin policy set <field>=<value>...` changes the fields named
(`letMaxSeconds=30d retentionDays=30`, `sourceBuiltAllowed=on`,
`logoPng=logo.png`), as the member signed in to the Installation's host,
who must be an Admin. Every change is one write the Installation's Control
API validates as a whole, so an invalid value changes nothing and names
the field, and records in the Installation's Activity record (the field,
its value before and after, who and when), which `factory admin policy
set` prints as `Changed <field> from <from> to <to>.`. Changing
`letMaxSeconds`, `maxIdleRenewals` or `retentionDays` also rewrites the
Revisions bucket's lifecycle rule, the backstop that deletes bundles the
sweep did not. An agent runs these from the `factory-admin` skill
(below). The Hub's page for Admins (All Artifacts, with the policy shown
read-only) comes in a later release: the Hub at
`https://<company-host>/app/` has its My Artifacts page so far.

## What an Admin can see

- **Who viewed what** is Google's log, not Relay Factory's: whoever can
  read private logs in the Artifacts project can read the Identity-Aware
  Proxy's entries for the last 30 days. Relay Factory keeps no record of
  views in this release.
- **Each Artifact's Activity record** (publishes, Revisions going live or
  failing, expiry) is kept with the Artifact for 400 days; `factory status`
  shows its Publisher the Artifact's pending, last failed or current
  Revision from it; no command reads the record itself yet, and the
  Admin's read of it comes with a later Admin command.
- **Any Artifact**, through the Installation's Control API: it knows who
  is an Admin by asking Google whether the member is in the Admin group
  (the [Groups Reader Handoff](handoffs/groups-reader.md) gives it the
  right to ask), and it lets an Admin read any Artifact, not only their
  own. `factory transfer`, `factory restore --publisher`
  ([Transfer an Artifact](#transfer-an-artifact)), `factory admin policy`
  and `factory admin status` need it; the rest of the Admin commands come
  with their releases. A member whose address is a service account is
  never an Admin.
- **The Installation's state** is `factory admin status --installation
  <handle>`: the release the installer recorded, whether the Installation
  is ready, each Handoff's status, the services' readiness as the Control
  API last checked it, and the capacity against its quota, with a warning
  at 80 %. It reads the installer's ledger with the Operator's `gcloud`
  login and the Control API as the member signed in to the Installation's
  host, who must be an Admin, so it needs both. The Spend cap, the usage
  per Publisher, email and the serving state are not yet available in
  this release; the report says so. The Operator's `factory admin install
  --check --installation <handle>` still shows the plan: what a run would
  change, which on a ready Installation is nothing; and `factory admin
  config get --installation <handle>` prints the inputs the install
  stored, which only a re-run of the installer changes
  ([Upgrade](upgrade.md#changing-a-mutable-input)).

## Transfer an Artifact

```sh
factory transfer q3-roadmap-x7k2q9 bob@example.com
```

Gives the Artifact to another member of the Workspace, who becomes its
Publisher: the one who renews, ends, restores and republishes it. The
Artifact changes hands in whatever state it is, live, Expired or never
live, and nothing else moves: its address, its Let, its Viewers and its
Revisions stay as they are. Only an Admin may do it; anyone else gets
`not_admin` (exit 34). The address must be one of the Workspace domain and
not the current Publisher's. The new Publisher is not emailed in this
release; tell them.

```sh
factory restore q3-roadmap-x7k2q9 --publisher bob@example.com
```

Restores an Expired Artifact and gives it to another member in one step:
it comes back live as that member's, from the bundle the Installation
retained ([Restore](publisher.md#restore)). Without `--publisher`, an
Admin's `factory restore` restores any member's Artifact for that member,
whether or not the Installation lets Publishers restore their own.

<!-- cliref:begin flags transfer -->

```
factory transfer <id> <email> [--installation <host>] [--json]
```

**Flags**

| Flag | Meaning |
|---|---|
| `--installation <host>` | the Installation the Artifact is on |
| `--json` | print the result as one JSON object on stdout |

<!-- cliref:end -->

## The commands under `factory admin` in this release

<!-- cliref:begin admin-commands -->

| Command | Summary |
|---|---|
| `factory admin install` | Install an Installation, or resume its install |
| `factory admin config get` | Print an Installation's stored config: its inputs, organization, projects |
| `factory admin policy get` | Print the Installation's policy, every field with its default |
| `factory admin policy set` | Change fields of the Installation's policy |
| `factory admin rebase` | Redeploy every live Artifact onto the release's base image (for Operators) |
| `factory admin status` | Report an Installation's state: readiness, Handoffs, capacity |

<!-- cliref:end -->

A coding agent learns these commands from the `factory-admin` skill
`factory` carries: `factory skills install --admin`
([Agent skills](publisher.md#agent-skills)). It tells the agent to run
`factory admin install --check` itself and to hand the apply to the
Operator.

`factory admin install` and `factory admin rebase` are the Operator's:
installing and upgrading, and the upgrade's rebase pass again
([Upgrade](upgrade.md#the-rebase-pass)); `policy get`, `policy set` and
`status` are the Admin's, as are `factory transfer` and `factory restore
--publisher` ([Transfer an Artifact](#transfer-an-artifact)). Each later
release of v1 adds the rest: the override that lets an Admin end or
re-share any Artifact, the Admin's read of the Activity records, the
Spend cap, the usage per Publisher and email in `status`, and the Hub's
remaining pages
([What v1 does not do](what-v1-does-not-do.md#in-v1-not-in-this-release)).
