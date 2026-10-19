# Restore Requests and their two policy fields

A Restore Request is how a member whom an Expired Artifact's Access Policy
allows asks for it back, from the Artifact's Hub page
(`https://<host>/app/a/<id>`, the one link on the Front Door's expired
page). Each member asks once until the Artifact is live again. A request
restores nothing by itself.

Two fields of the policy shape them (`factory admin policy set`):

- `restoreRequestButton` (true) offers the request at all: off, the Hub
  shows no button and the Control API takes none.
- `publisherSelfRestore` (true), besides letting Publishers restore their
  own Artifacts, decides who sees the requests: on, the Publisher, on My
  Artifacts and the Artifact's Hub page; off, the Admins instead, on the
  Artifact's Hub page and the Hub's All Artifacts, since only an Admin
  restores it then.

Each new request is mailed once, when the Installation's email is on, to
the Publisher or, with `publisherSelfRestore` off, to the Admin group: from
the member via the Installation, so a reply reaches them, with the Hub
page's link, where the requests are listed with who asked and when.

To answer one, restore the Artifact (`factory restore <id>`, SKILL.md
section 9, or the Hub page's **Restore**) or leave it to its Publisher to
publish again. The requests clear when the Artifact is live again,
whatever brought it back, and the members who asked are not told.
