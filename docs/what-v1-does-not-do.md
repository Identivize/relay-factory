# What v1 does not do

Things an Operator should know Relay Factory v1 does not do, by design.
The second list is what v1 will do and this release does not have yet.

## By design

- **It never prompts.** The installer and the CLI take every input from
  flags or a `--config` file and ask nothing on a terminal, so a coding
  agent can run them; a missing input is a `usage` failure naming it, not a
  question.
- **A bookmarked Artifact origin answers 404 after expiry.** An Artifact's
  own host (`<id>.<artifact-domain>`) stops answering when its Let ends.
  The shareable link is `https://<company-host>/a/<id>`: it redirects to the
  origin while the Artifact is live and shows the expired page afterwards,
  so hand out `/a/<id>` links, never the origin.
- **Uninstall is a project delete.** Removing an Installation deletes its
  three projects, with Google's 30-day undelete as the only recovery and no
  export of Artifacts or records. The two cleanups the delete cannot do are
  manual: remove the four DNS records, and revoke any delegation granted
  outside the projects. (The `factory admin uninstall` command itself is not
  in this release; see below.)
- **A fresh install means the four DNS records again.** Reinstalling
  creates new projects and a new load balancer with a new address; the two
  CNAMEs and two A records must be edited by hand. There is no DNS
  automation.
- **No error-redirect bucket and no second load balancer.** The option to
  serve the expired and not-found pages from a bucket of your own, and the
  two-load-balancer layout, are not offered. An Admin can point the expired
  and not-found outcomes at a URL of the organization's own instead
  (`errorRedirect`, [Admin](admin.md)).
- **No money ceiling without a Spend cap.** The Spend cap is the Operator's
  installer input, and this release does not have it yet, nor the cap of
  three instances per static Artifact that v1 adds beside it: an
  Installation's spending has no ceiling of Relay Factory's. Set a budget
  on the three projects in Google's console meanwhile.
- **Viewing is Google's log, not Relay Factory's.** Whoever can read
  private logs in the Artifacts project can read who viewed what for the
  last 30 days. The optional Viewing records (an Operator switch, not in
  this release) copy those entries into a dataset of the organization's
  own; entries in the sink's first minute can be lost.
- **Google identities only.** Viewers are members of the Workspace domain,
  signed in with Google. There are no other identity providers, no
  anonymous viewing and no sharing outside the domain.
- **Static Artifacts only in this layout.** Bring-your-own-container
  Artifacts, antivirus scanning of bundles and a documentation site are out
  of scope for v1. The `factory` binary is not notarized for macOS.

## In v1, not in this release

Each later release of v1 adds its part; this release has none of these:

- The Admin's override of a Publisher's Artifact, and the Admin's read of
  the Activity records; `factory admin policy get|set` and `factory admin
  status` are in this release ([Admin](admin.md)).
- `factory admin uninstall` and `factory admin end`. (Changing a mutable
  input on a re-run and `factory admin config get` are in this release:
  [Upgrade](upgrade.md#changing-a-mutable-input).)
- Pre-created projects (an Operator who is Owner of existing projects but
  not a Project Creator).
- The Spend cap, the Paused state, the Usage limits per Publisher, and
  email: the Spend cap and email are Operator inputs, with the email
  sender's own Handoff. The installer already accepts and stores
  `--spend-cap` and `--email-sender`, and does nothing with them yet
  ([Upgrade](upgrade.md#changing-a-mutable-input)).
- The viewing-records switch and its retention, and the source-built
  switch: `--viewing-records`, `--viewing-records-retention-days` and
  `--source-built` are accepted and stored the same way. (The Groups
  Reader Handoff is in this release: [Groups
  Reader](handoffs/groups-reader.md).)
- The Hub's pages beyond My Artifacts (Shared with me, Across the
  domain, an Artifact's page and its actions, the Publish page, All
  Artifacts for Admins, Restore Requests, branding), named shares
  (`factory share`, `factory unshare`),
  and source-built apps: `factory.yaml` already accepts `kind:
  streamlit` and `kind: python`, and the Installation refuses them with
  `source_built_disabled` until then.

Each is listed here until the release that carries it, whose
documentation then describes it.
