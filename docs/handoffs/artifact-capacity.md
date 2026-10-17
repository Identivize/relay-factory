# Handoff: Artifact capacity

**Who does it:** someone with Owner on the Installation's Artifacts
project, which is the Operator who ran the install.

**What it is for:** each Artifact is its own Cloud Run service in the
Artifacts project. A new project's Cloud Run quota allows far fewer
services than an organization's Artifacts will need, so the install asks
Google for more. Google reviews quota requests, which can take days, so
this Handoff **never holds the install up**: the Installation is ready
without it and holds as many Artifacts as the current quota allows until
the request is granted.

## Steps

The installer's instructions name the Artifacts project (`<art>`), the
region and the target, `artifactCapacity` (default 4000).

1. In the Google Cloud console, in project `<art>`: IAM & Admin, Quotas &
   System Limits
   (`https://console.cloud.google.com/iam-admin/quotas?project=<art>`).
2. Filter on the Cloud Run Admin API and find, for the Installation's
   region, **Services per project** (`ServicesPerProject`) and **Active
   revisions per project** (`ActiveRevisionsPerProject`).
3. Request an increase of each to the target, with a short justification
   (one Cloud Run service per published Artifact).
4. Google answers the request by email, usually within a few days; a large
   request may be granted in steps.

## Done when

The Cloud Quotas API reports both quotas at or above the target. Until
then the report shows the Handoff as pending with a `Now:` line giving the
current quotas and the target. Nothing else waits for it: the ready report
and exit 0 do not depend on this Handoff, and it is verified again on every
run of `factory admin install --installation <handle>`.
