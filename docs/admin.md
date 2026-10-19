# Admin

The Admins of an Installation are the members of its Admin Google Group,
the `adminGroup` the Operator gave at install. Admins own the
Installation's policy: how long a Let lasts, what the expired page shows,
the limits per Publisher, the branding and whether source-built apps are
allowed and how they run. Installing and upgrading is the Operator's
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
| `expiryWarningLeadSeconds` | 2 days | How long before a Let ends its Publisher is mailed that it expires soon, once per Let; a Let shorter than this gets no warning. A change applies to the Lets started or renewed after it. |
| `publisherSelfRestore` | `true` | A Publisher may restore their own Expired Artifact (`factory restore`); off, their attempt is refused with `restore_disabled` and only an Admin restores it. |
| `restoreRequestButton` | `true` | The Hub's page of an Expired Artifact, which the expired page links, offers a member the Artifact was shared with a Restore Request: one button to ask for it back, once per member until it is live again. Off, there is no button and none is taken. With `publisherSelfRestore` on the requests go to the Publisher (My Artifacts, the Artifact's page, a mail); off, to the Admins instead ([Restore Requests](#restore-requests)). |
| `listedByDefault` | `false` | A new Artifact starts Listed, shown to every member in the Hub's Across the domain (unless its first publish narrows it with `--share`). Off, an Artifact is found by its address until its Publisher runs `factory share listed`; an Admin can list and unlist any Artifact the same way. |
| `errorRedirect` | empty | Where the expired and not-found outcomes send a Viewer instead of the built-in pages: an absolute `https` URL of the organization's own. Empty means the built-in pages, which say why the link does not answer. |
| `displayName` | empty | The Installation's display name, one line of at most 80 characters, no leading or trailing space: the name in the Hub's header, on the public expired, not-found and paused pages, and in every mail. Empty means the Workspace domain ([Branding](#branding)). |
| `logoPng` | empty | The logo, a PNG file of at most 64 KiB (an SVG or any other format is refused), shown beside the name in the Hub and on the public pages; `factory admin policy set` takes the file's path. Empty means none. |
| `accentColor` | empty | The Hub's accent colour as `#rrggbb` (its primary buttons, selected tabs and focus rings); the public pages keep their own colours. Empty means the Hub's own. |
| `staticMaxInstances` | 3 | Max instances of a static Artifact's service; a change applies to the Revisions deployed after it, never to a running service. |
| `liveArtifactsPerPublisher` | 50 | Live Artifacts one Publisher may hold; at the limit a publish that needs a new one, a Restore and a Transfer to them are refused with `publisher_at_limit`, a republish of a live Artifact never. |
| `publishesPerPublisherPerDay` | 100 | Publishes one Publisher may make per UTC day, Restores included; at the limit their publishes are refused with `publish_rate_limited` until the next UTC midnight. |
| `sourceBuiltAllowed` | `false` | Source-built (`streamlit`, `python`) apps may be published. The switch allows or pauses publishing them; it does not install anything: the Operator installs the capability first ([Source-built apps](#source-built-apps) below), and while it is not installed a source-built publish is refused whatever the switch says. |
| `sourceBuiltVcpu` | 1 | vCPU of a source-built app's service, 1 to 8. |
| `sourceBuiltMemoryMiB` | 1024 | Its memory in MiB, 128 to 32768. |
| `sourceBuiltMaxInstances` | 1 | Its max instances. |
| `sourceBuiltMinInstances` | 0 | Its min instances, up to `sourceBuiltMaxInstances`. |
| `sourceBuiltRequestTimeoutSeconds` | 3600 | Its request timeout, up to 3600. |
| `sourceBuiltBuildTimeoutSeconds` | 600 | Its build timeout, up to 86400. |
| `sourceBuiltEgress` | `none` | Its egress: `none` (the Installation's no-egress network: the app reaches nothing on the internet) or `internet`. One setting for every source-built app. |
| `scanThreshold` | `CRITICAL` | The vulnerability finding in the app's own packages that fails a build: `CRITICAL` (the default), or `HIGH` to tighten it. The scan cannot be turned off. |

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
(below). The Hub's Installation page shows the policy read-only
([The Hub's Admin pages](#the-hubs-admin-pages)).

## Branding

Three fields name and dress the Installation: `displayName`, `logoPng` and
`accentColor`. Set them together or one at a time:

```sh
factory admin policy set "displayName=Example Corp" logoPng=logo.png accentColor=#1a73e8
```

- **Where they show.** The Hub shows all three on every page: the name and
  the logo in its header and the name in the browser's tab, the accent on
  its primary buttons, selected tabs and focus rings. The public pages a Viewer can meet
  on the Installation's host, expired, not found and paused, show the name
  and the logo. Every mail the Installation sends names it by the display
  name.
- **The defaults.** Until a display name is set, the Workspace domain
  stands in for it everywhere. With no logo there is no image; with no
  accent the Hub keeps its own colours.
- **The checks.** The display name is one line of at most 80 characters
  with no leading or trailing space; the logo a PNG file of at most 64 KiB
  (an SVG, a JPEG or a larger file is refused); the accent `#` and six hex
  digits. A refused value is `invalid_request` (exit code 7) naming the
  field, and nothing changes. `logoPng=` with nothing after it removes the
  logo, as `displayName=` and `accentColor=` clear theirs.
- **When it shows.** The Hub and the public pages each read the branding
  at most once a minute, so a change shows within a minute; a mail sent
  after the change carries it.

## Source-built apps

A source-built app (`kind: streamlit` or `kind: python` in its
`factory.yaml`) is built from its code and run, rather than served as
files. Two switches gate it, one each side:

- **The Operator installs the capability** with `factory admin install
  --source-built` ([Install](install.md#source-built-apps)): a fourth,
  build project where the builds run under an account with no rights
  anywhere else, a package mirror, and a no-egress network for the
  apps. Until then every source-built publish is refused with
  `source_built_disabled`, and the policy switch changes nothing.
- **An Admin allows publishing them** with `factory admin policy set
  sourceBuiltAllowed=true`, and turns it off again with `=false`, which
  refuses new source-built publishes and keeps the Artifacts that are
  live (static publishing goes on; this is not the Spend cap's Paused
  state). The limits above (`sourceBuiltVcpu` to
  `sourceBuiltBuildTimeoutSeconds`), the egress and the scan threshold
  apply to the Revisions published after a change, never to a running
  service: to close egress for an app that is live, end its Let and have
  it published again.

An admitted publish is built by Cloud Build in the build project (one
build per publish, up to `sourceBuiltBuildTimeoutSeconds`), its image
scanned on push, then deployed into the Artifacts project as a service of
its own: no permissions, the first-generation sandbox, the limits above,
and under egress `none` a network that reaches nothing but DNS and the
metadata server. The Publisher sees the stage and the last lines of their
own build or container log through `factory status`; the full logs are in
the build project and the Artifacts project, yours and the Operator's.

**The scan gate.** Before a scanned image is deployed, the deploy worker
reads the scan's findings in the Publisher's own packages and judges them
against `scanThreshold`: a known-malicious package anywhere, or a finding
at or above the threshold that has a fix, fails the publish with a
`scan failed` reason naming the package and its fix, which the Publisher
sees through `factory status`. Findings in the base image and the build
tooling never fail a publish: they are Google's to patch. The whole
report, base image included, stays on the Revision for you to read
through the Control API,
`GET /api/v1/artifacts/{id}/revisions/{rev}/scan` (Admins only; no CLI
command in this release), or on the Artifact's page in the Hub
([The Hub's Admin pages](#the-hubs-admin-pages)). The gate cannot be turned off: `CRITICAL` is
the default and `HIGH` tightens it. A Restore rebuilds and is scanned
again; a Renewal is not rescanned.

## What an Admin can see

- **Who viewed what** is Google's log, not Relay Factory's: whoever can
  read private logs in the Artifacts project can read the Identity-Aware
  Proxy's entries for the last 30 days. Where the Operator has turned
  [viewing records](install.md#viewing-records) on, a thin record of each
  request is kept longer in BigQuery, read through the datasets' sharing,
  which the Operator controls; the Hub shows Admins counts of it only,
  never who ([The Hub's Admin pages](#the-hubs-admin-pages)).
- **Each Artifact's Activity record** (publishes, Revisions going live or
  failing, expiry, shares, Renewals, an Admin's end with the name and
  reason) is kept with the Artifact for 400 days; `factory status`
  shows its Publisher the Artifact's pending, last failed or current
  Revision from it, and the Hub's page for an Artifact shows the whole
  record to its Publisher and to any Admin
  ([An Artifact's page](publisher.md#an-artifacts-page)); no command
  reads the record itself yet.
- **Any Artifact**, through the Installation's Control API: it knows who
  is an Admin by asking Google whether the member is in the Admin group
  (the [Groups Reader Handoff](handoffs/groups-reader.md) gives it the
  right to ask), and it lets an Admin read any Artifact, not only their
  own, and to end any Artifact
  ([End an Artifact's Let](#end-an-artifacts-let)). In the Hub, an
  Artifact's page shows an Admin the same actions its Publisher gets
  (renew, restore, share, unshare, list, unlist, end), each recorded with
  the Admin's name
  ([An Artifact's page](publisher.md#an-artifacts-page)). `factory transfer`,
  `factory restore --publisher`
  ([Transfer an Artifact](#transfer-an-artifact)), `factory admin end`,
  `factory admin policy` and `factory admin status` need it; the rest of
  the Admin commands come with their releases. A member whose address is
  a service account is
  never an Admin.
- **The Installation's state** is `factory admin status --installation
  <handle>`: the release the installer recorded, whether the Installation
  is ready, each Handoff's status, the services' readiness as the Control
  API last checked it, the **capacity**: the service slots live
  Artifacts hold against the Artifacts project's Cloud Run services
  quota, which the Control API reads live from Google, at most once a
  minute (a raise Google grants counts at the next publish or status
  check, with no re-install), with a warning at 80 % and when your group
  was mailed about it, and the [Spend cap](install.md#spend-cap): `unset;
  the Installation has no money ceiling.`, or its amount and the budget's
  state, `configured`, `enforced: the Installation is Paused` or `lifted
  for the rest of the month`, read from Google by the budget's stored
  name. It reads the installer's ledger and the budget with the
  Operator's `gcloud` login and the Control API as the member signed in
  to the Installation's host, who must be an Admin, so it needs both. It
  also says whether **email is on**: `Email: on, sent as <mailbox>` once
  the sweep has sent the test mail to your group, or `Email: off` with the
  reason (no sender configured, or the [Email](handoffs/email.md) Handoff
  pending: the test mail not yet tried, Google refusing the delegation, or
  the sweep unable to reach Google). Until it is on, every command that
  would mail says so in its own output. And the **usage per Publisher**,
  in counts and states, never money: one line naming every member who
  has published, with the live Artifacts they hold, the publishes they
  made this UTC day and whether they are `ok`, `near limit` (within 10 %
  of either limit) or `at limit`, then the two limits and when today's
  counts reset; a Publisher at the day's limit is refused their next
  publish, one at the live limit their next publish that needs a new live
  Artifact, and
  `factory admin policy set` raises the limit. And the Installation's own
  **serving state**: `Serving: yes.`, or `Serving: Paused since <time>`
  with the budget's state as the sweep last read it, while the Spend cap
  is enforced: links then answer a paused page, and publishes and
  Restores are refused with `installation_paused` until the Operator
  lifts the cap or it lifts on the first of next month (Renewals still
  work). When the slots reach 80 % of the quota and email is on, the
  Installation mails your group once, with the count, the quota and where
  the Operator raises it; it mails again only after the count has dropped
  below 80 % and reached it again. At 95 % new Artifacts are refused with
  `installation_at_capacity`; republishes, Renewals and Restores go on up
  to the quota. The Operator's `factory admin install
  --check --installation <handle>` still shows the plan: what a run would
  change, which on a ready Installation is nothing; and `factory admin
  config get --installation <handle>` prints the inputs the install
  stored, which only a re-run of the installer changes
  ([Upgrade](upgrade.md#changing-a-mutable-input)).

## The Hub's Admin pages

The Hub at `https://<company-host>/app/` shows an Admin two pages more
than a member gets, and more on an Artifact's page. It shows them only to
a member the Control API's Admin check says is an Admin, and caches the
check per member for five minutes, so a member added to or removed from
the Admin group sees the change within that.

- **All Artifacts** (`/app/#all`): every Artifact on the Installation,
  whoever published it, newest first, with the filter of
  `factory admin end`: the kind (static, source-built, `streamlit`,
  `python`), the state (live, expired, not published), the Publisher's
  exact address, and published before a date (read as midnight UTC), an
  instant or an age such as `7d`, which matches the Artifacts whose current Let started
  before it. Each row gives the Artifact, linked to its address, its
  Publisher, state and Let, and a link to its page. A Restore Requests
  column names who asked: on every Artifact while `publisherSelfRestore`
  is off, since the requests then come to you, and on your own Artifacts
  while it is on. While the Operator has the
  [viewing records](install.md#viewing-records) on, the page also gives
  each Artifact's views, distinct Viewers and last view, with the
  Installation's totals above.
- **Installation** (`/app/#installation`): the Installation's state,
  read-only. The serving state (Serving, or Paused since a time with the
  [Spend cap](install.md#spend-cap) budget's state as the sweep last read
  it), the Cloud Run services in use against the quota, the usage per
  Publisher (live Artifacts and publishes today against the two limits,
  `ok`, `near a limit` or `at a limit`), the viewing totals or that the
  viewing records are off, and the whole policy, each field with its
  value and its default. You change the policy with
  `factory admin policy set`, never in the Hub, and nothing on the page
  edits anything. The Operator's commands (install, uninstall, config,
  rebase) are on no Hub page.
- **On an Artifact's page**, besides the actions its Publisher gets: a
  **Transfer** to another member's address, the same as
  `factory transfer` ([Transfer an Artifact](#transfer-an-artifact)), and,
  for a source-built Artifact, the **scan report** of the Revision the
  page shows: the gate's verdict at `scanThreshold`, its reason when it
  failed, and the findings in the app's own packages and in the base
  image and build tooling ([The scan gate](#source-built-apps)), or a
  line saying the Revision has none.

A view is one Viewer on one Artifact in one UTC day, and only an admitted
request that names a Viewer counts: a refused or signed-out request never
does. The counts name nobody, and they trail the viewing records by up to
an hour, which trail the Identity-Aware Proxy by up to an hour.

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
not the current Publisher's. When the Installation's email is on, both
Publishers get a mail in your name (the Reply-To is you), each saying what
changed for them; the command's stderr says whether they were emailed,
and if not (email off, or a send the Installation logged), tell them
yourself. The Hub's page for the Artifact offers you the same Transfer
([The Hub's Admin pages](#the-hubs-admin-pages)).

```sh
factory restore q3-roadmap-x7k2q9 --publisher bob@example.com
```

Restores an Expired Artifact and gives it to another member in one step:
it comes back live as that member's, from the bundle the Installation
retained ([Restore](publisher.md#restore)). Without `--publisher`, an
Admin's `factory restore` restores any member's Artifact for that member,
whether or not the Installation lets Publishers restore their own. Either
way the Publisher is emailed in your name when email is on (both
Publishers with `--publisher`), and the command says what became of it
before it waits for the Artifact to go live.

### Restore Requests

A member an Expired Artifact was shared with can ask for it back from its
Hub page (`https://<company-host>/app/a/<id>`, the link on the expired
page), once until it is live again: a Restore Request
([Publisher](publisher.md#restore-requests)). While `publisherSelfRestore`
is on, the requests go to the Artifact's Publisher. While it is off, only
an Admin may restore, so they come to you instead: one mail per request
to the Admin group when the Installation's email is on, from the member
via the Installation (reply to reach them), naming the Artifact, its
Publisher, how many asked and whether its bundle is still kept; and the
Artifact's Hub page, opened from that mail's link, lists who asked and
when. Restore it with `factory restore <id>` (or the page's **Restore**),
or leave it to its Publisher to publish again; the requests clear once it
is live again, and the members who asked are not told. `restoreRequestButton`
off removes the button for everyone ([The policy](#the-policy)).

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

## End an Artifact's Let

```sh
factory admin end --publisher bob@example.com --published-before 30d
```

Lists the live Artifacts the filter matches, one line each with the ID,
title, Publisher, App type and when its current Let started, and ends
nothing. Run it again with `--yes` and, if you like, a `--reason`:

```sh
factory admin end --publisher bob@example.com --published-before 30d \
  --yes --reason "the project closed"
```

Each Artifact's Let ends at once: its link goes down when the call
returns and its service within seconds, as if the Let had run out. The
Artifact stays its Publisher's, keeps its address and may be restored or
published again by them; your name and the reason go on its Activity
record, where the Publisher reads them, and when the Installation's email
is on each Publisher gets a mail in your name with the reason and what
they may do next (restore it while its bundle is kept, or publish again).
The report's last line says whether they were emailed; when email is off,
or a send failed, tell them yourself. To keep an Artifact down, transfer
it to yourself first ([Transfer an Artifact](#transfer-an-artifact)), then
end it.

Name Artifacts instead of a filter with `factory admin end <id>...`. The
filters combine: `--kind static|streamlit|python|source-built` (the App
type; `source-built` is every kind but `static`), `--publisher <email>`,
`--published-before <when>` (a date such as `2026-10-01`, an RFC 3339
instant, or an age such as `7d` or `36h`; it compares with the start of
the Artifact's current Let, which a publish, a Restore or a Renewal
starts), or `--all` on its own for every live Artifact. Only an Admin may
list or end this way; anyone else gets `not_admin` (exit 34).

With `--yes` every Artifact is tried and reported as `ended`, `skipped`
or `failed`: skipped means a publish of it is in flight (the line says
when that publish's lock lapses; run the command again after), failed is
any other refusal with its code (a named Artifact that is not live, or
one that expired on its own meanwhile, is `not_live`; an ID that is not
on the Installation is `not_found`; both show in the listing too). Any
skip or failure exits `not_all_ended` (41) with the
whole report, so a script or an agent reads the outcomes and retries
the ones that need it. The public expired page names nobody; the Hub's
page of the Expired Artifact says only that an Admin or the Publisher
ended it, and when.

<!-- cliref:begin flags admin end -->

```
factory admin end <id>... | [--kind <kind>] [--publisher <email>] [--published-before <when>] | --all
                  [--reason <text>] [--yes] [--installation <host>] [--json]
```

**Flags**

| Flag | Meaning |
|---|---|
| `--kind <kind>` | static, streamlit, python, or source-built |
| `--publisher <email>` | the Publisher's address |
| `--published-before <when>` | a date, an RFC 3339 instant, or an age (7d) the current Let started before |
| `--all` | every live Artifact (no other filter) |
| `--reason <text>` | why; recorded on each Artifact's activity record |
| `--yes` | end them; without it, list and end nothing |
| `--installation <host>` | the Installation the Artifacts are on |
| `--json` | print the result as one JSON object on stdout |

<!-- cliref:end -->

## The commands under `factory admin` in this release

<!-- cliref:begin admin-commands -->

| Command | Summary |
|---|---|
| `factory admin install` | Install an Installation, or resume its install |
| `factory admin config get` | Print an Installation's stored config: its inputs, organization, projects |
| `factory admin end` | End the Let of any Artifact now, by ID or by a filter (for Admins) |
| `factory admin policy get` | Print the Installation's policy, every field with its default |
| `factory admin policy set` | Change fields of the Installation's policy |
| `factory admin rebase` | Redeploy every live Artifact onto the release's base image (for Operators) |
| `factory admin status` | Report an Installation's state: readiness, Handoffs, capacity, Spend cap, email, usage |
| `factory admin uninstall` | Delete an Installation's projects, or empty pre-created ones (for Operators) |

<!-- cliref:end -->

A coding agent learns these commands from the `factory-admin` skill
`factory` carries: `factory skills install --admin`
([Agent skills](publisher.md#agent-skills)). It tells the agent to run
`factory admin install --check` itself and to hand the apply to the
Operator.

`factory admin install` and `factory admin rebase` are the Operator's:
installing and upgrading, and the upgrade's rebase pass again
([Upgrade](upgrade.md#the-rebase-pass)); `policy get`, `policy set`,
`status` and `end` are the Admin's, as are `factory transfer` and
`factory restore --publisher` ([Transfer an Artifact](#transfer-an-artifact)) and
`factory share` and `unshare` on any Artifact, the Hub's page for
any Artifact with its actions and record, and the Hub's Admin pages, All
Artifacts and Installation ([The Hub's Admin pages](#the-hubs-admin-pages)).
Each later release of v1 adds the rest
([What v1 does not do](what-v1-does-not-do.md#in-v1-not-in-this-release)).
