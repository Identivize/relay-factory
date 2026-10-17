# Handoff: Invoker IAM

**Who does it:** an Organization Policy Administrator of the Google Cloud
organization (`roles/orgpolicy.policyAdmin`).

**What it is for:** the Installation's three services behind the load
balancer in the control-plane project, the Front Door, the Control API
and the Hub, run with Cloud Run's invoker check disabled, so that the load
balancer can reach them (the Hub is guarded by Google's Identity-Aware
Proxy on its load-balancer path instead; each Artifact's service, by
contrast, keeps the check on and is guarded by IAP directly). The organization
policy `run.managed.requireInvokerIam` forbids disabling the check when it
is enforced. This Handoff is issued only when the installer sees that
policy enforced on the control-plane project; most organizations never
see it.

## Steps

The installer's instructions name the control-plane project, `<cp>` below.

1. In the Google Cloud console, Organization policies, constraint
   `run.managed.requireInvokerIam`, viewed on project `<cp>`
   (`https://console.cloud.google.com/iam-admin/orgpolicies/run-managed.requireInvokerIam?project=<cp>`).
2. Manage the policy on that project: override the parent's policy and set
   the constraint to **not enforced** for `<cp>` alone. The rest of the
   organization keeps its policy.
3. Tell the Operator it is done.

## Done when

The effective policy on the control-plane project is not enforced. The
installer reads it on every run; while it is enforced, the install stops
before creating the services (the second Terraform phase waits for this
Handoff), with the registry already prepared, and ends pending (exit 26).
The Operator's next run, `factory admin install --installation <handle>`,
creates the services and goes on to the other Handoffs.
