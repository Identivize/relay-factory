# Upgrade

An Installation upgrades in place: the Operator runs the newer release's
`factory admin install` against it. There is no `upgrade` command and no
downgrade.

```sh
factory admin install --installation <handle>
```

Get the newer release's `factory` as in [Install](install.md#1-get-factory),
then run it with the Installation's handle. Run `--check` first: its plan
shows what the upgrade would do, the images to copy, the migrations to
run, the policy fields to seed and the Terraform changes resource by
resource, and the release check says `an upgrade of this Installation from
<release>`. The plain run prints the same plan and applies it with no
confirmation.

## What an upgrade does

The same convergent sequence as every run, with the new release's parts:

1. The new release's images are verified against its signer and copied by
   digest into the Installation's registries, before anything switches.
2. The registry is brought forward to the new release's schema by additive
   migrations, each recorded as it completes, so the release still serving
   keeps working until the switch, and a run that fails part-way resumes
   at the first migration not yet recorded.
3. The policy is seeded field by field: a field the new release knows and
   the document lacks is written at its default; a field already present
   is never touched, whoever set it.
4. The Terraform module is applied, switching the services to the new
   images, and the new release is recorded in the ledger.
5. When the release changed the base image live Artifacts are served
   from, every live Artifact is redeployed onto it
   ([The rebase pass](#the-rebase-pass)).
6. The Handoffs are verified as on every run and are not re-issued.

Publishers' Artifacts, Lets and sign-ins stay as they are.

## The rebase pass

Live Artifacts are served from a base image the release carries, with the
Publisher's files added on top. When a release changes that image (a fix
in the server, say), the upgrade redeploys every live Artifact onto it
after the apply: each gets a new Revision of the same bundle, built on the
new base. Nothing changes for Publishers or Viewers: links keep working,
Lets are unchanged, and no one has to publish again. Artifacts already on
the new base, Artifacts with a publish in progress, and Expired ones are
left alone.

The deploys go through the Installation's deploy queue one a second, and
the run waits while they go out: a thousand live Artifacts take about
seventeen minutes, during which the plan's `Rebase:` line and the `rebase`
step say what is happening. Stopping the run (Ctrl-C) is safe; running it
again resumes. While an Artifact's own redeploy runs, about a minute, a
publish or early end of that Artifact is refused as "deploy in progress";
try again after it. An Artifact whose bundle the new release refuses stays
on its previous base; its activity record says why. An upgrade or a
`factory admin rebase` while the Installation is Paused (its Spend cap
enforced) cannot redeploy anything: each rebase Revision fails with
`Installation paused`, every Artifact stays live on its previous base, and
the pass is run again with `factory admin rebase` once the Installation is
serving.

```sh
factory admin rebase --installation <handle>
```

runs the pass again and prints the counts: how many live Artifacts there
are, how many are being redeployed, how many were still deploying from an
earlier pass (resumed), how many are on the base already, how many have a
publish in progress, and how many stopped being live meanwhile. Run it
when an upgrade stopped during the pass (exit 28; the re-run resumes), or
after an upgrade that skipped a release which changed the base image. An
Installation upgraded from a release before this one must run
`factory admin install` from this release first: the command says so
(exit 28, `install_failed`), because the pass needs what that install
records.

## What is refused

A `factory` **older** than the release the Installation runs is refused by
the release check with exit 30, `older_release`, before anything changes;
its message names the Installation's release, to use that one or a newer
one. Releases are ordered as semantic versions: `v0.1.0-rc.3` is older than
`v0.1.0`, which is older than `v0.2.0`. An Installation never moves back.

A permanent input given with a different value is refused too, as on any
run, with exit 37, `permanent_input`, before preflight and with nothing
changed: `workspaceDomain`, `region`, `companyHost`, `artifactDomain`,
`projectPrefix` and the pre-created projects' names name the Installation.
The message names the input and
both values; drop it to resume
([Troubleshooting](troubleshooting.md#exit-37-permanent_input)).

## Changing a mutable input

The inputs that configure the Installation rather than name it are
changed by a re-run with the input's flag (or a `--config` file with the
key): the run applies the new value and then stores it, and a later run
without the flag keeps it.

- `--billing-account <id>`: the projects are linked to the new
  account in the run's projects step, and the account is stored then.
- `--admin-group <email>`: the services carry the new group from the
  run's second-phase apply (the plan shows the Control API's service
  updated), and the group is stored once that apply has succeeded. A run
  held by the `invoker-iam` Handoff, or one whose apply fails, does not
  store it; the progress says so, and the re-run needs the flag again.
- `--artifact-capacity <n>`: the `artifact-capacity` Handoff asks for the
  new quota and is verified against it; stored in the projects step.
- `--spend-cap <amount>`: the [Spend cap](install.md#spend-cap)'s budget
  on the Artifacts project is created or updated to the new amount in
  the run's spend-cap step, and the amount is stored then. An enforced
  cap must be lifted first; `0` leaves an existing budget as it is.
- `--source-built`: the build project is created, or adopted with
  `--build-project`, in the run's projects step, and the switch is stored
  then; the applies install the capability's resources and hand the
  Control API the build project
  ([Install](install.md#source-built-apps)). `--source-built=false`
  removes the resources and keeps the project.
- `--email-sender <address>`: the three sending services carry the new
  mailbox from the run's second-phase apply (the plan shows the Control
  API, the deploy worker and the sweep updated), and it is stored once
  that apply has succeeded, like the Admin group. On an Installation that
  had no sender this issues the [Email](handoffs/email.md) Handoff; on one
  that had another, email is off again until the sweep has sent the test
  mail as the new mailbox (the delegation covers the sender account, not
  the mailbox, so no new console step is needed unless the mailbox does
  not exist). A sender cannot be removed by a re-run: to turn email off,
  a super admin revokes the delegation.

Every run's plan lists the changes first, as `Input <key> (<flag>): from
<stored> to <given>; applied by this run, then stored.`, so `--check`
with the flag shows what a run would change before it does, and the
module's change for a new Admin group is not reported as drift. Check
first, as for any run:

```sh
factory admin install --check --installation <handle> --admin-group platform-admins@example.com
factory admin install --installation <handle> --admin-group platform-admins@example.com
```

`factory admin config get --installation <handle>` prints the stored
config, with `--json` as the one JSON object the state bucket holds: every
input as stored, the organization, the project IDs (the build project's
among them once `--source-built` has been on) and the OAuth
client's ID, and no secret. It reads the state bucket with your `gcloud`
login, as `install` does. There is no `config set`: the re-run is the
change.

`viewingRecords` (`--viewing-records`, or `--viewing-records=false`) and
`viewingRecordsRetentionDays` (`--viewing-records-retention-days <days>`)
are mutable like the rest: a re-run with the flag installs, removes or
re-times the [viewing records](install.md#viewing-records), and stores
the value once the second-phase apply has succeeded.

The account that re-runs the installer needs roles in the Installation's
projects (what their Owner holds) and the Billing Account User role
on its billing account; the Project Creator role on the organization is
needed by the first install only, and by the re-run that creates the
build project with `--source-built`, and not at all with
[pre-created projects](prerequisites.md#pre-created-projects).

## What this release does not do yet

Upgrades are promised between stable releases from `v1.0.0` on. For an
Installation on a prerelease, the upgrade is tried but not promised; a
fresh install is the fallback, and it means new projects, a new load
balancer address and the four DNS records again
([What v1 does not do](what-v1-does-not-do.md)).
