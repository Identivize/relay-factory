# factory commands

The Publisher's commands of the `factory` CLI, with their flags, from the
one table the CLI's own help is rendered from (`factory help <command>`
prints the same). The tables between the markers are generated from that
table; the headings are one per command, in the order a Publisher meets
them. Flags may come before or after a command's positional arguments;
`--json` is accepted by every command.

<!-- cliref:begin publisher-commands -->

| Command | Summary |
|---|---|
| `factory end` | End the Let of your own Artifact now |
| `factory login` | Sign in to an Installation and make it the default |
| `factory logout` | Sign out of an Installation and forget its Sign-in |
| `factory publish` | Publish a folder as a new Revision of its Artifact |
| `factory renew` | Renew the Let of your own Artifact without publishing |
| `factory restore` | Bring an Expired Artifact back from its retained bundle, as a publish |
| `factory share` | Let named members and groups view an Artifact |
| `factory skills install` | Write the skills to each agent's skills directory |
| `factory skills status` | Compare each installed skill's version with this factory's |
| `factory skills uninstall` | Remove what "factory skills install" wrote |
| `factory status` | Report the Installation, the Sign-in and the folder's Artifact |
| `factory transfer` | Give an Artifact to another Publisher (Admins only) |
| `factory unshare` | Take named members, groups or the domain off an Artifact |
| `factory version` | Print the release this factory was built from |

<!-- cliref:end -->

## factory status

<!-- cliref:begin flags status -->

```
factory status [dir] [--wait [--timeout <duration>]]
               [--installation <host>] [--json]
```

**Flags**

| Flag | Meaning |
|---|---|
| `--wait` | wait for the pending Revision to go live or fail, with publish's codes; at once when none is pending |
| `--timeout <duration>` | how long --wait waits for the Revision to go live (default 16m for a static app, 31m for a source-built one) |
| `--installation <host>` | the Installation to report on |
| `--json` | print the result as one JSON object on stdout |

<!-- cliref:end -->

## factory login

<!-- cliref:begin flags login -->

```
factory login [host] [--no-browser] [--json]
```

**Flags**

| Flag | Meaning |
|---|---|
| `--no-browser` | print the sign-in URL without opening a browser |
| `--json` | print the result as one JSON object on stdout |

<!-- cliref:end -->

## factory publish

<!-- cliref:begin flags publish -->

```
factory publish [dir] [--artifact <id>] [--title <title>]
                [--kind <kind>] [--entrypoint <entrypoint>]
                [--let <duration>] [--no-wait] [--installation <host>]
                [--timeout <duration>] [--json]
```

**Flags**

| Flag | Meaning |
|---|---|
| `--artifact <id>` | publish to this Artifact instead of the folder's; the link file is neither read nor written |
| `--title <title>` | title for a new Artifact, written to factory.yaml (default: the folder's name) |
| `--kind <kind>` | the App type written to factory.yaml: static, streamlit or python (default: detected) |
| `--entrypoint <entrypoint>` | written to factory.yaml: the file to run for streamlit, the start command for python |
| `--let <duration>` | the Let's length, 2d, 36h or 1d12h, instead of the default; at most the Installation's maximum |
| `--no-wait` | return once the commit is accepted with the Artifact and Revision IDs; status --wait waits |
| `--share <who>` | on a first publish, who may view instead of the whole domain: addresses, group:&lt;address&gt;, domain or nobody; repeatable, or comma-separated |
| `--no-notify` | with --share: record no Share mail to the named |
| `--message <text>` | with --share: a note for the Share mail |
| `--installation <host>` | the Installation to publish to |
| `--timeout <duration>` | how long to wait for the Revision to go live (default 16m for a static app, 31m for a source-built one) |
| `--json` | print the result as one JSON object on stdout |

<!-- cliref:end -->

## factory renew

<!-- cliref:begin flags renew -->

```
factory renew [dir|id] [--let <duration>] [--installation <host>] [--json]
```

**Flags**

| Flag | Meaning |
|---|---|
| `--let <duration>` | the Let's length from now, 2d, 36h or 1d12h, instead of the default; at most the Installation's maximum |
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

## factory end

<!-- cliref:begin flags end -->

```
factory end [dir|id] [--installation <host>] [--json]
```

**Flags**

| Flag | Meaning |
|---|---|
| `--installation <host>` | the Installation the Artifact is on |
| `--json` | print the result as one JSON object on stdout |

<!-- cliref:end -->

## factory share

<!-- cliref:begin flags share -->

```
factory share <who>... [dir|id] [--only] [--no-notify] [--message <text>]
              [--timeout <duration>] [--installation <host>] [--json]
factory share listed [dir|id] [--installation <host>] [--json]
```

**Flags**

| Flag | Meaning |
|---|---|
| `--only` | replace the policy with the names given, the domain off unless named |
| `--no-notify` | record no Share mail to the newly named |
| `--message <text>` | a note for the Share mail |
| `--timeout <duration>` | how long to wait for the policy to reach the service (default 16m0s) |
| `--installation <host>` | the Installation the Artifact is on |
| `--json` | print the result as one JSON object on stdout |

<!-- cliref:end -->

## factory unshare

<!-- cliref:begin flags unshare -->

```
factory unshare <who>... [dir|id] [--timeout <duration>] [--installation <host>] [--json]
factory unshare listed [dir|id] [--installation <host>] [--json]
```

**Flags**

| Flag | Meaning |
|---|---|
| `--timeout <duration>` | how long to wait for the policy to reach the service (default 16m0s) |
| `--installation <host>` | the Installation the Artifact is on |
| `--json` | print the result as one JSON object on stdout |

<!-- cliref:end -->

## factory logout

<!-- cliref:begin flags logout -->

```
factory logout [--installation <host>] [--json]
```

**Flags**

| Flag | Meaning |
|---|---|
| `--installation <host>` | the Installation to sign out of |
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

## factory skills status

<!-- cliref:begin flags skills status -->

```
factory skills status [--json]
```

**Flags**

| Flag | Meaning |
|---|---|
| `--json` | print the result as one JSON object on stdout |

<!-- cliref:end -->

## factory skills uninstall

<!-- cliref:begin flags skills uninstall -->

```
factory skills uninstall [--json]
```

**Flags**

| Flag | Meaning |
|---|---|
| `--json` | print the result as one JSON object on stdout |

<!-- cliref:end -->

## factory transfer

An Admin's command (the `factory-admin` skill); a Publisher who runs it
gets `not_admin` (34).

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
