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
on its previous base; its activity record says why.

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
run: `workspaceDomain`, `region`, `companyHost`, `artifactDomain` and
`projectPrefix` name the Installation.

The account that re-runs the installer needs roles in the Installation's
three projects (what their Owner holds) and the Billing Account User role
on its billing account; the Project Creator role on the organization is
needed by the first install only.

## What this release does not do yet

- **Changing the mutable inputs** (`billingAccount`, `adminGroup`,
  `artifactCapacity`) on a re-run, and `factory admin config get`. In this
  release every input is fixed after the first run.

Upgrades are promised between stable releases from `v1.0.0` on. For an
Installation on a prerelease, the upgrade is tried but not promised; a
fresh install is the fallback, and it means new projects, a new load
balancer address and the four DNS records again
([What v1 does not do](what-v1-does-not-do.md)).
