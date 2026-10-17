# Publisher

A Publisher is a member of the Workspace domain who publishes a folder as
an Artifact with the `factory` CLI. Each publish is a Revision; the first
publish from a folder creates its Artifact and gives you a shareable link,
`https://<company-host>/a/<id>`, that every member of the domain can open
after signing in with Google. An Artifact lives for its Let (15 days by
default) and expires afterwards.

**The Front Door host comes from the Installation's Admin:** ask them for
it. Everything below uses `artifacts.example.com` for it. The CLI never prompts, so it
suits a coding agent as well as a terminal: progress goes to stderr, the
result to stdout, and `--json` makes the result one JSON object.

## Get `factory`

With Homebrew (stable releases only):

```sh
brew install --cask identivize/tap/relay-factory
```

Or download a release archive for your machine and verify it. Replace
`darwin_arm64` with your OS and architecture (`darwin_amd64`, `linux_amd64`,
`linux_arm64`, `windows_amd64.zip`, `windows_arm64.zip`):

```sh
gh release download v0.1.0-rc.5 --repo Identivize/relay-factory --pattern 'factory_*_darwin_arm64.tar.gz' --pattern checksums.txt --pattern checksums.txt.sigstore.json
```

```sh
cosign verify-blob \
  --bundle checksums.txt.sigstore.json \
  --certificate-identity 'https://github.com/Identivize/relay-artifactory/.github/workflows/release.yml@refs/tags/v0.1.0-rc.5' \
  --certificate-oidc-issuer https://token.actions.githubusercontent.com \
  checksums.txt
```

```sh
sha256sum --ignore-missing -c checksums.txt
```

(`shasum -a 256 --ignore-missing -c checksums.txt` on macOS.) Then
`tar xzf` the archive and put `factory` on your `PATH`. The binary is not
notarized for macOS: a browser download is quarantined; `gh`, `curl` and
Homebrew downloads are not. Any release's `factory` works as a Publisher
against any Installation: the Publisher commands are versioned by the
Installation's API, not by release.

## The Hub

Every member of the Workspace domain can open the Installation's web
page, the Hub, at `https://<company-host>/app/` (`https://<company-host>/`
leads there). Google's Identity-Aware Proxy asks for the Workspace account
first; nobody outside the organization gets past it. Its page in this
release is **My Artifacts**: everything you published, newest first, with
each Artifact's state (Live, Expired, or not published yet) and when its
Let ends or ended; Expired Artifacts stay in the list, marked. The title
links to the shareable address. **Sign out** in the header ends the
Hub's session; a browser still signed in to Google may be let straight
back in. The Hub's other pages are later releases'
([What v1 does not do](what-v1-does-not-do.md)); publishing stays on the
CLI.

## Sign in

```sh
factory login artifacts.example.com
```

The sign-in URL is printed on stderr and a browser is opened; sign in with
your Workspace account. Google redirects back to the CLI on this machine,
so a browser on another device cannot finish the sign-in. The Sign-in's
refresh token goes into the OS keychain where one answers (the macOS
Keychain, Windows Credential Manager, or a running Secret Service on
Linux), else into `credentials.json` under the OS config directory (or
`$RELAY_FACTORY_CONFIG_DIR`), which in both cases records the Sign-in and
makes this Installation your default. An account outside the
Installation's domain is refused with `wrong_domain`. `--no-browser`
prints the URL without opening one; `factory login` with no host signs in
again to your default Installation.

```sh
factory logout
```

Forgets the Sign-in to your default Installation (or `--installation
<host>`) from the keychain and the file, and clears the default. Nothing
is sent anywhere: the refresh token is forgotten, not revoked. `factory
version` prints the release this `factory` was built from.

## Publish

```sh
factory publish ./site
```

Publishes the folder (default: the current folder) as a new Revision,
waits until it is live and prints the shareable URL on stdout, so
`url=$(factory publish ./site)` works. The first publish creates the
Artifact and records it in `./site/.relay-factory/artifact.json`, the
folder's link file; later publishes from the folder are Revisions of that
Artifact. Give `--title` on the first publish (default: the folder's name)
and `--artifact <id>` to publish to an Artifact of yours without using the
folder's link.

The folder's `factory.yaml` declares what the Artifact is: `title`,
`kind` (`static`, `streamlit` or `python`) and, for the last two,
`entrypoint`; nothing else may be in it. A folder with none gets one on
its first publish: a root `index.html` is `static`, `requirements.txt` or
`pyproject.toml` is `python`, a dependency list naming streamlit is
`streamlit`, and `--title`, `--kind` and `--entrypoint` set the fields only
then (edit the file afterwards). A static site needs `index.html` at the
folder's root and holds no Python; one that breaks that rule is
`invalid_bundle` before anything is uploaded, with the rule and the way
out in the message. This release deploys static sites only: a `streamlit`
or `python` kind is refused with `source_built_disabled`.

What is published is the folder's regular files, with the rules of a
`.gitignore` in the folder and of a `.factoryignore` at its root honoured,
and a few names always left out (`.git`, `.relay-factory`, agent folders);
`factory.yaml` is always included. A symlink or a special file makes the
folder unpublishable (`invalid_bundle`). The compressed bundle may be at
most 100 MiB.

A publish waits up to 16 minutes for the Revision to go live
(`--timeout`). A Revision that deploys and fails is `revision_failed` with
the reason; a publish still waiting at the timeout is `wait_timeout`, and
the Revision may yet go live. One publish of an Artifact at a time: a
second one while the first is in flight is `deploy_in_progress`.

To publish without waiting, `factory publish --no-wait` returns as soon
as the upload is committed, printing the Artifact and Revision IDs, and
`factory status --wait` in the folder waits for that Revision later, with
the same outcomes a waiting publish has (`revision_failed`,
`wait_timeout`, its own `--timeout`); when nothing is pending it reports
at once.

`--let <duration>` gives the Revision a Let of that length instead of the
Installation's default: `--let 2d` ends it two days after it goes live
(`36h`, `1d12h` and `2h30m` work too). The most it may ask for is the
Installation's maximum, which is the default unless the Admin raises it;
a longer Let is refused before anything is uploaded.

## Status

```sh
factory status ./site
```

Reports, changing nothing: the Installation, who you are signed in as and
where the Sign-in is kept, and, for a linked folder, the Artifact's title,
state, Let end, Access Policy and its pending, last failed or current
Revision, with a failed one's reason (`--json` adds the Publisher and the
Artifact as the Installation reports it). A folder with no link file
reports no Artifact. It never waits, never prompts and never opens the
keychain: when the Sign-in's token is in the keychain, a linked folder
reports the link alone, with a line saying so, and `factory publish` is
what reads the Artifact then. It is the first thing to run when a publish
did not do what you expected.

`factory status --wait` is the exception that waits: after a `factory
publish --no-wait`, it waits for the folder's pending Revision to go live
or fail and exits as the publish would have, then reports; with nothing
pending it reports at once. Because it waits on the Installation it reads
the Sign-in as `publish` does, keychain included.

## Renew

```sh
factory renew ./site
```

Renews the Let of your own Artifact without publishing: the Let restarts
from now at the Installation's default length, with no new upload, and the
command prints the new end. It restarts from now, not from the Let's
current end, so renewing early buys nothing. `--let 2d` (or `36h`,
`1d12h`) asks for a shorter Let, at most the Installation's maximum. A Let
may be renewed a bounded number of times (the Installation's policy, 2 by
default); at the cap the renew is `renewals_exhausted`, and publishing
again starts a fresh Let. The argument is the linked folder (default: the
current folder) or an Artifact ID. An Artifact with no running Let is
`not_live`: an Expired Artifact is published again, not renewed. Nothing
is written locally and no email is sent.

## End

```sh
factory end ./site
```

Ends the Let of your own Artifact now: the link goes down when the command
returns and the service within seconds. The Artifact stays yours, keeps
its address and may be published again. The argument is the linked folder
(default: the current folder) or an Artifact ID. An Artifact with no
running Let is `not_live`; one with a publish in flight is
`deploy_in_progress`. Nothing is written locally and no email is sent.

## Agent skills

`factory` carries two skills in the open Agent Skills format that teach a
coding agent the CLI: `factory`, for Publishers, and `factory-admin`, for
Admins and Operators ([Admin](admin.md)). Each is stamped with this
release.

```sh
factory skills install
```

Writes the `factory` skill to the user-level skills directory of Claude
Code (`~/.claude/skills`, or `$CLAUDE_CONFIG_DIR/skills`), of the agents
that read `~/.agents/skills` (Codex, Cursor, GitHub Copilot) and of
Antigravity (`~/.gemini/config/skills`), creating the directories, and
prints where; `--admin` adds `factory-admin`, and `--agent <name>` limits
the write to `claude-code`, `agents` or `antigravity`. After upgrading
`factory`, run it again: `factory skills status` compares each installed
copy's version with the binary's, `factory login` says once when a copy is
older, and `factory skills uninstall` removes exactly what `install` wrote.
The skill tells the agent to start with `factory status`, to look for the
Installation's host in the folder's `AGENTS.md` or `CLAUDE.md` before
asking you for it, and to report the link only after a publish. The same
skills are in this repository's `skills/` folder, for `gh skill install`,
`npx skills add` and Claude Code's plugin marketplace
(`.claude-plugin/marketplace.json`), updated on stable releases only.

## Commands

<!-- cliref:begin publisher-commands -->

| Command | Summary |
|---|---|
| `factory end` | End the Let of your own Artifact now |
| `factory login` | Sign in to an Installation and make it the default |
| `factory logout` | Sign out of an Installation and forget its Sign-in |
| `factory publish` | Publish a folder as a new Revision of its Artifact |
| `factory renew` | Renew the Let of your own Artifact without publishing |
| `factory skills install` | Write the skills to each agent's skills directory |
| `factory skills status` | Compare each installed skill's version with this factory's |
| `factory skills uninstall` | Remove what "factory skills install" wrote |
| `factory status` | Report the Installation, the Sign-in and the folder's Artifact |
| `factory version` | Print the release this factory was built from |

<!-- cliref:end -->

`factory admin install` is the Operator's command ([Install](install.md)).
`factory help <command>` prints a command's flags.

### `factory login`

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

### `factory logout`

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

### `factory publish`

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
| `--installation <host>` | the Installation to publish to |
| `--timeout <duration>` | how long to wait for the Revision to go live (default 16m0s) |
| `--json` | print the result as one JSON object on stdout |

<!-- cliref:end -->

### `factory status`

<!-- cliref:begin flags status -->

```
factory status [dir] [--wait [--timeout <duration>]]
               [--installation <host>] [--json]
```

**Flags**

| Flag | Meaning |
|---|---|
| `--wait` | wait for the pending Revision to go live or fail, with publish's codes; at once when none is pending |
| `--timeout <duration>` | how long --wait waits for the Revision to go live (default 16m0s) |
| `--installation <host>` | the Installation to report on |
| `--json` | print the result as one JSON object on stdout |

<!-- cliref:end -->

### `factory renew`

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

### `factory end`

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

### `factory version`

<!-- cliref:begin flags version -->

```
factory version [--json]
```

**Flags**

| Flag | Meaning |
|---|---|
| `--json` | print the result as one JSON object on stdout |

<!-- cliref:end -->

### `factory skills install`

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

### `factory skills status`

<!-- cliref:begin flags skills status -->

```
factory skills status [--json]
```

**Flags**

| Flag | Meaning |
|---|---|
| `--json` | print the result as one JSON object on stdout |

<!-- cliref:end -->

### `factory skills uninstall`

<!-- cliref:begin flags skills uninstall -->

```
factory skills uninstall [--json]
```

**Flags**

| Flag | Meaning |
|---|---|
| `--json` | print the result as one JSON object on stdout |

<!-- cliref:end -->

**Which Installation.** `--installation <host>` if given; else the one the
folder's link file names; else the one you signed in to last. Flags may
come before or after the folder argument, and `--json` anywhere.

## Exit codes

Every failure has a stable code, in the exit status and in `--json`'s
`error.code`, and a message on stderr. "API" codes are the Installation's
Control API's, passed through with its message; "CLI" codes are the CLI's
own. The installer's codes are on the
[Troubleshooting](troubleshooting.md) page.

<!-- cliref:begin publisher-exit-codes -->

| Code | Exit | From | When |
|---|---|---|---|
| `internal` | 1 | API, CLI | Anything else, including a credential store that cannot be read or written, and any code this CLI does not know. |
| `usage` | 2 | CLI | Bad command line, a host that is not an Installation host, or a malformed link file. `factory admin install`: also a missing or changed input, an unknown Installation, or an OAuth client Google rejects. `factory admin rebase`: an unknown Installation, or one whose install never wrote its config. |
| `unauthenticated` | 3 | API, CLI | Not signed in to the Installation, a sign-in whose OAuth client was replaced, a refresh Google refuses, or an ID token the Control API rejects. Sign in again with `factory login <host>`. |
| `wrong_domain` | 4 | API, CLI | The Google account has no Workspace domain, or not the Installation's. |
| `not_publisher` | 5 | API | The Artifact belongs to another Publisher. |
| `not_found` | 6 | API | No such Artifact or Revision on this Installation. |
| `invalid_request` | 7 | API | The Control API refused the request's body. |
| `unsupported_kind` | 8 | API | The kind sent is not `static`, `streamlit` or `python`. |
| `too_large` | 9 | API, CLI | The bundle is above 100 MiB compressed. |
| `deploy_in_progress` | 10 | API | Another publish of the Artifact holds its publish lock. Do not publish again, or end the Artifact, until it finishes; the message says when the lock lapses. |
| `installation_at_capacity` | 11 | API | The Installation has no room for another Artifact. |
| `superseded` | 12 | API | The Revision being committed is no longer the Artifact's pending one. |
| `upload_missing` | 13 | API | The commit found no uploaded bundle for the Revision. |
| `upload_mismatch` | 14 | API | The uploaded bundle's size or checksum differs from what was declared. |
| `not_configured` | 15 | API, CLI | The Installation's desktop OAuth client is not set up yet (a pending install Handoff). |
| `unavailable` | 16 | API | The Installation's discovery document was still unavailable after the CLI's retries. |
| `network` | 20 | CLI | The Installation, Google or the upload host could not be reached, did not answer in time, or answered with something that is not the contract. |
| `invalid_bundle` | 21 | CLI | The folder cannot be published as it is: its `factory.yaml` is invalid or its kind could not be detected, a `static` app has no root `index.html` or holds Python, or the folder has a symlink or special file, a whiteout name, an unreadable file, or no regular file. The message names the rule and the way out. |
| `revision_failed` | 22 | CLI | The Revision was uploaded and its deploy failed (`publish`, or `status --wait` on it); the message is its failure reason (a bundle the build step refused, its manifest or kind included). |
| `wait_timeout` | 23 | CLI | The Revision was not live or failed when `--timeout` passed (`publish`, or `status --wait` on it). |
| `login_failed` | 24 | CLI | The sign-in was refused, abandoned, timed out, or Google's answer lacked what is needed. |
| `renewals_exhausted` | 33 | API | `factory renew`: the Artifact has used the idle Renewals the Installation allows per Let; publish again to start a fresh Let. |
| `not_admin` | 34 | API | `factory admin policy` and `factory admin status`: the member signed in is not a member of the Admin group (`RELAY_ADMIN_GROUP`). Sign in as an Admin, or ask one. |
| `not_live` | 35 | API | `factory end` and `factory renew`: the Artifact has no running Let (never live, or Expired already). Its link is already down; publish again to bring it back. |
| `source_built_disabled` | 36 | API | The `factory.yaml` declares `streamlit` or `python` and this Installation does not build apps from source (the capability is not installed or the Admin switch is off); only `static` publishes. |
| `interrupted` | 130 | CLI | SIGINT or SIGTERM stopped the command. A Revision already committed may still go live. |

<!-- cliref:end -->

## Not in this release

Named shares (`factory share`, `factory unshare`), Restore of an expired
Artifact and the Hub's pages beyond My Artifacts are v1 features that
later releases add
([What v1 does not do](what-v1-does-not-do.md)).
