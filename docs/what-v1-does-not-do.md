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
- **Uninstall is a project delete.** `factory admin uninstall` deletes
  the Installation's projects, with Google's 30-day undelete as the
  only recovery and no export of Artifacts or records; projects you
  pre-created are emptied of the Installation's resources and left in
  place instead. The two cleanups the delete cannot do are manual, and the
  command prints them: revoke the email sender's delegation, and remove
  the four DNS records ([Uninstall](uninstall.md)).
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
  installer input, `--spend-cap` ([Install](install.md#spend-cap)): one
  Google budget that caps Cloud Run spend in the Artifacts project. An
  Installation whose Operator set none has no money ceiling of Relay
  Factory's: its bill is bounded by its live Artifacts alone. The cap of
  three instances per static Artifact that v1 adds beside it is not in
  this release.
- **Viewing is Google's log, not Relay Factory's.** Whoever can read
  private logs in the Artifacts project can read who viewed what for the
  last 30 days. The optional [viewing records](install.md#viewing-records)
  (an Operator switch) keep a thin copy of those entries in BigQuery in
  the Artifacts project, read through the datasets' sharing, which the
  Operator controls; entries in the sink's first minute can be lost.
  With the records on, the Hub shows Admins counts only (views, distinct
  Viewers and the last view), never who.
- **Google identities only.** Viewers are members of the Workspace domain,
  signed in with Google. There are no other identity providers, no
  anonymous viewing and no sharing outside the domain.
- **Static Artifacts only in this layout.** Bring-your-own-container
  Artifacts, antivirus scanning of bundles and a documentation site are out
  of scope for v1. The `factory` binary is not notarized for macOS.

## In v1, not in this release

Each later release of v1 adds its part; this release has none of these:

- The Admin's read of the Activity records; the Admin's override of a
  Publisher's Artifact (read, renew, restore, transfer, share and end),
  `factory admin policy get|set`, `factory admin status` and `factory
  admin end` are in this release ([Admin](admin.md)).
  (Changing a mutable input on a re-run, `factory admin config get` and
  `factory admin uninstall` are in this release too:
  [Upgrade](upgrade.md#changing-a-mutable-input),
  [Uninstall](uninstall.md).)
- Some of the mails. The sender and its [Email](handoffs/email.md)
  Handoff are built, `factory admin status` says whether email is on, and
  the Admin-change mails are sent (an Admin's Transfer, Restore, Early
  end, share, unshare, listing or unlisting of a member's Artifact mails
  that Publisher, [Admin](admin.md)), as are the Share mail to the members
  and groups a share newly names ([Publisher](publisher.md#share)),
  the expiry mails to a Publisher whose Let ends soon or has ended
  ([Publisher](publisher.md#renew)) and the Restore Request mail to a
  Publisher or the Admin group ([Publisher](publisher.md#restore-requests)),
  as are the mails to the Admin group when the Installation is Paused and
  when it serves again, and the capacity warning to the Admin group at
  80 % of the Cloud Run services quota ([Admin](admin.md)).
- Telling the members who made a Restore Request what became of it: the
  Artifact's link simply works again once it is back.

Each is listed here until the release that carries it, whose
documentation then describes it.
