# factory-admin commands

The commands an Admin or Operator uses, with their flags, from the one
table the CLI's own help is rendered from (`factory help admin install`,
`factory help admin policy set` and so on print the same). The tables
between the markers are generated from that table. Flags may come before
or after a command's positional arguments; `--json` is accepted by every
command.

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

## factory admin install

<!-- cliref:begin flags admin install -->

```
factory admin install [--check] [--installation <control-plane-project>]
                      [--config <file>] [input flags]
                      [--oauth-client-id <id> --oauth-client-secret-file <path>]
                      [--wait <duration>] [--json]
```

**Flags**

| Flag | Meaning |
|---|---|
| `--check` | plan the run and change nothing: the preflight checks, the Handoffs, and what the run would create, copy, migrate and apply |
| `--installation <project>` | the control-plane project ID of an Installation to resume or check |
| `--config <file>` | a JSON object of inputs, keyed workspaceDomain, billingAccount, region, companyHost, artifactDomain, adminGroup, artifactCapacity, projectPrefix, controlPlaneProject, artifactsProject, notifierProject, buildProject, spendCap, viewingRecords, viewingRecordsRetentionDays, sourceBuilt, emailSender |
| `--wait <duration>` | keep verifying a pending DNS Handoff for up to this long (default 0: verify once) |
| `--json` | print the report as one JSON object on stdout |

**Input flags**

| Flag | Meaning |
|---|---|
| `--workspace-domain <domain>` | the Google Workspace domain; permanent (required) |
| `--billing-account <id>` | the billing account to link; mutable (required) |
| `--region <region>` | the Installation's region; permanent (required) |
| `--company-host <host>` | the Front Door's host; permanent (required) |
| `--admin-group <email>` | the Admin Google Group; mutable (required) |
| `--artifact-domain <domain>` | parent of each Artifact's host; permanent (default a.&lt;company-host&gt;) |
| `--artifact-capacity <n>` | how many Artifacts to request quota for; mutable (default 4000) |
| `--project-prefix <prefix>` | prefix of the project IDs the run creates; permanent (default relay-factory); not with the pre-created projects |
| `--control-plane-project <id>` | a pre-created project to adopt as the control plane, with --artifacts-project and --notifier-project; permanent |
| `--artifacts-project <id>` | a pre-created project to adopt as the Artifacts project; permanent |
| `--notifier-project <id>` | a pre-created project to adopt as the notifier project; permanent |
| `--build-project <id>` | a pre-created project to adopt as the build project of source-built apps, with --source-built; permanent |
| `--spend-cap <amount>` | the monthly Spend cap on Cloud Run in the Artifacts project, a whole number of the billing account's currency units; mutable (0 is no cap: an existing cap is left as it is) |
| `--source-built` | install source-built apps: the build project and its resources, the no-egress network; mutable (default off; --source-built=false removes them and keeps the project) |
| `--email-sender <address>` | the mailbox the Installation sends mail as; mutable (default none: no email) |

**Viewing records**

| Flag | Meaning |
|---|---|
| `--viewing-records` | install the viewing records: IAP's entries routed to BigQuery, an hourly thin record and its counts; mutable (default off; --viewing-records=false removes them and keeps the two record datasets) |
| `--viewing-records-retention-days <days>` | days the viewing records are kept; mutable (default 400) |

**Completing the oauth-client Handoff**

| Flag | Meaning |
|---|---|
| `--oauth-client-id <id>` | the Desktop app OAuth client's ID |
| `--oauth-client-secret-file <path>` | a file holding its secret; - is stdin |

<!-- cliref:end -->

## factory admin config get

<!-- cliref:begin flags admin config get -->

```
factory admin config get --installation <control-plane-project> [--json]
```

**Flags**

| Flag | Meaning |
|---|---|
| `--installation <project>` | the control-plane project ID of the Installation (required) |
| `--json` | print the stored config as one JSON object on stdout |

<!-- cliref:end -->

## factory admin policy get

<!-- cliref:begin flags admin policy get -->

```
factory admin policy get [--installation <host>] [--json]
```

**Flags**

| Flag | Meaning |
|---|---|
| `--installation <host>` | the Installation whose policy to read |
| `--json` | print the policy and the defaults as one JSON object |

<!-- cliref:end -->

## factory admin policy set

The arguments are `<field>=<value>`, by the wire names; `factory help
admin policy set` lists every field with its default and what it takes.

<!-- cliref:begin flags admin policy set -->

```
factory admin policy set <field>=<value>... [--installation <host>] [--json]
```

**Flags**

| Flag | Meaning |
|---|---|
| `--installation <host>` | the Installation whose policy to change |
| `--json` | print the policy, the defaults and the changes as one JSON object |

<!-- cliref:end -->

## factory admin status

<!-- cliref:begin flags admin status -->

```
factory admin status --installation <control-plane-project> [--json]
```

**Flags**

| Flag | Meaning |
|---|---|
| `--installation <project>` | the control-plane project ID of the Installation (required) |
| `--json` | print the result as one JSON object on stdout |

<!-- cliref:end -->

## factory admin end

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

## factory admin rebase

<!-- cliref:begin flags admin rebase -->

```
factory admin rebase --installation <project> [--json]
```

**Flags**

| Flag | Meaning |
|---|---|
| `--installation <project>` | the control-plane project ID of the Installation (required) |
| `--json` | print the result as one JSON object on stdout |

<!-- cliref:end -->

## factory admin uninstall

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

## factory transfer

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

## factory restore

<!-- cliref:begin flags restore -->

```
factory restore [dir|id] [--publisher <email>] [--let <duration>] [--no-wait]
                [--timeout <duration>] [--installation <host>] [--json]
```

**Flags**

| Flag | Meaning |
|---|---|
| `--publisher <email>` | give the Artifact to this member as it comes back (Admins only) |
| `--let <duration>` | the Let's length, 2d, 36h or 1d12h, instead of the default; at most the Installation's maximum |
| `--no-wait` | return once the Revision is committed, without waiting for it to go live |
| `--timeout <duration>` | how long to wait for the Revision to go live (default 16m for a static app, 31m for a source-built one) |
| `--installation <host>` | the Installation the Artifact is on |
| `--json` | print the result as one JSON object on stdout |

<!-- cliref:end -->

## factory version

<!-- cliref:begin flags version -->

```
factory version [--json]
```

**Flags**

| Flag | Meaning |
|---|---|
| `--json` | print the result as one JSON object on stdout |

<!-- cliref:end -->

## factory skills install

<!-- cliref:begin flags skills install -->

```
factory skills install [--admin] [--agent <name>]... [--json]
```

**Flags**

| Flag | Meaning |
|---|---|
| `--admin` | write the factory-admin skill as well |
| `--agent <name>` | write only this agent's directory: claude-code, agents or antigravity; repeat the flag for more than one |
| `--json` | print the result as one JSON object on stdout |

<!-- cliref:end -->
