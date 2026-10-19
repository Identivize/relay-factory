# Handoff: Groups Reader

**Who does it:** a Google Workspace super administrator of the
Installation's Workspace account.

**What it is for:** an Installation's Admins are the members of its Admin
Google Group. To know whether a member is an Admin, the Installation's
services ask Google whether that member is in the group, as themselves:
each service runs as its own service account, and that account needs the
pre-built Workspace admin role **Groups Reader** to read group membership.
No domain-wide delegation is involved, and the role reads only groups:
it cannot change anything. The Control API uses it for the Admin check
and to recognise the names a share gives, and the Hub uses the same role
for its Shared with me page, asking whether the signed-in member is in
each group an Artifact was shared with. Only a super
admin can assign a Workspace admin role, so the installer cannot do it.

## Steps

The installer's instructions name the two service accounts, in the
control-plane project `<cp>`:
`control-api@<cp>.iam.gserviceaccount.com` (the Control API) and
`hub-service@<cp>.iam.gserviceaccount.com` (the Hub).

1. In the Google Admin console (`https://admin.google.com`), signed in as
   a super administrator: Menu, Account, Admin roles.
2. Point to the pre-built role **Groups Reader** and choose **Assign
   admin**, then **Assign service accounts**.
3. Enter each address the installer's instructions name, both of them.
4. Set the role's scope to **all groups** (not only security groups or
   only non-security groups: the Admin group may be either).
5. **Add**, then **Assign role**.
6. Tell the Operator it is done.

## Done when

The Control API and the Hub can each read the Admin group as themselves.
Each tries at start-up and every five minutes, and reports the outcome in
the Installation's discovery document, which the installer reads: the
Handoff is done when both report the read succeeded. The Operator re-runs
`factory admin install --installation <handle>` once the role is assigned,
or waits five minutes first so that the next check has run.

Until then the report shows the Handoff as pending with a `Now:` line
saying what the last check found. Google answering **403** means the role
is missing on that account or its scope is narrower than all groups; Google
does not say which, so check both. A **404** means the Admin group the
Installation was given was not found. "Not reported yet" means the service
has not checked since the role was assigned: wait five minutes and run
again ([Troubleshooting](../troubleshooting.md#the-groups-reader-handoff-stays-pending)).

The Installation serves Publishers while this Handoff is pending, so the
run reports it ready with this Handoff pending and exits 26
(`handoff_pending`) until it is done; it exits 0 once it is
([Install](../install.md#6-the-ready-report)).
