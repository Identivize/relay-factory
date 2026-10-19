# Uninstall

`factory admin uninstall` removes an Installation. It is the Operator's
command, run with the Operator's own `gcloud` login, and it is one command
with one confirmation: the Installation's Workspace domain on `--confirm`,
in place of a prompt. Nothing is exported first: every Artifact, Revision,
record and Let the Installation held goes with it. Read
[What v1 does not do](what-v1-does-not-do.md) before you run it.

```sh
factory admin uninstall --installation <handle> --confirm <workspace-domain>
```

`<handle>` is the Installation's control-plane project ID, as every other
`factory admin` command takes it; `<workspace-domain>` is the
`workspaceDomain` of its config, exactly as
`factory admin config get --installation <handle>` prints it. A missing or
wrong value is refused as `usage` (exit 2) and changes nothing, and so is
an unknown Installation or one already uninstalled.

## What it does

**Projects the install created** are deleted: the control plane, the
Artifacts project, the notifier project and, when the install made one
with `--source-built`, the build project. Google keeps a deleted project
recoverable for about 30 days and then deletes it for good, with everything
in it. That undelete is the only way back, and the command prints it for
each project:

```
gcloud projects undelete <project-id>
```

An undeleted project comes back as it was. Running the Installation
again afterwards (`factory admin install --installation <handle>` from the
same release) has not been exercised in this release: treat it as a
recovery to check, not a promise.

**Projects you pre-created** (named to the first install with
`--control-plane-project` and its siblings, [Install](install.md#pre-created-projects))
are yours, so they are not deleted. They are emptied of the Installation's
resources instead: everything the Terraform module made (the registry
database and the Revisions bucket lose their delete protection on the way
out; this takes several minutes, a few more on an Installation whose first
apply never finished), then the Artifacts' Cloud Run services, then the
state bucket. The [viewing records'](install.md#viewing-records) datasets
are deleted with their rows while the records are on; datasets left
behind by an earlier `--viewing-records=false` are not, and are yours to
delete. The projects stay, with their APIs enabled and their billing
linked, and the report says so; there is nothing to undelete. A build
project named to the install is emptied with them once `--source-built`
adopted it, and not touched otherwise. Installing into the emptied
projects again is a first install; whether Google lets the registry
database be created again at once is not something this release has
checked.

**In either case** the Spend cap's budget is deleted: it lives on the
billing account, not in a project, and Google would refuse a fresh
Installation's budget for the same project while it exists.

The command prints its progress as it goes and ends with the report: each
project's fate, the undelete commands, the budget, and the two cleanups
below. With `--json` the same is one object on stdout.

## The two cleanups only a person can do

Deleting the projects revokes nothing outside Google Cloud, so the report
ends with two items, each saying whether there is anything to do:

1. **Revoke the email sender's delegation.** If the Installation had an
   email sender, a Workspace super admin authorized its sender account for
   `gmail.send` by domain-wide delegation
   ([Email](handoffs/email.md)). The report names the account's OAuth
   client ID: in the Admin console, Menu > Security > Access and data
   control > API controls > Manage Domain Wide Delegation, remove it. The
   mailbox is yours and is left as it is. With no sender configured the
   report says there is nothing to revoke.
2. **Remove the four DNS records.** The two `_acme-challenge` CNAMEs and
   the two A records of the [DNS Handoff](handoffs/dns.md) point at a load
   balancer that no longer exists; the report lists them. A fresh install
   issues new ones.

## If it stops

A step that fails ends the command with `install_failed` (exit 28), the
step named in the message, and the report as far as it got. Run the same
command again: every step is safe to repeat, what is deleted stays deleted,
and the run finishes the rest. Ctrl-C is safe in the same way
(`interrupted`, exit 130).

## Flags

<!-- cliref:begin flags admin uninstall -->

```
factory admin uninstall --installation <project> --confirm <workspaceDomain> [--json]
```

**Flags**

| Flag | Meaning |
|---|---|
| `--installation <project>` | the control-plane project ID of the Installation (required) |
| `--confirm <domain>` | the Installation's Workspace domain, as stored (required; the one confirmation, in place of a prompt) |
| `--json` | print the result as one JSON object on stdout |

<!-- cliref:end -->
