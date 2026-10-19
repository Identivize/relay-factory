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
gh release download v0.1.0-rc.10 --repo Identivize/relay-factory --pattern 'factory_*_darwin_arm64.tar.gz' --pattern checksums.txt --pattern checksums.txt.sigstore.json
```

```sh
cosign verify-blob \
  --bundle checksums.txt.sigstore.json \
  --certificate-identity 'https://github.com/Identivize/relay-artifactory/.github/workflows/release.yml@refs/tags/v0.1.0-rc.10' \
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
first; nobody outside the organization gets past it. Its pages in this
release are **My Artifacts**, **Shared with me**, **Across the domain**,
**Publish** and each Artifact's own page.
My Artifacts is everything
you published, newest first, with each Artifact's state (Live, Expired, or
not published yet) and when its Let ends or ended; Expired Artifacts stay
in the list, marked. The title links to the shareable address, and
**Details** opens the Artifact's page. **Sign
out** in the header ends the Hub's session; a browser still signed in to
Google may be let straight back in. When members ask for an Expired
Artifact of yours back ([Restore Requests](#restore-requests)), a
**Restore Requests** column names who asked. An Admin also gets the
**All Artifacts** and **Installation** pages
([Admin](admin.md#the-hubs-admin-pages)).

### Shared with me

**Shared with me** lists the Artifacts other people shared with you by
name, or with a Google group you belong to (a nested group counts),
newest first, with who published each, its state and when its Let ends
or ended; Expired Artifacts stay, marked. Open one by its title. The Hub
works out your groups when you open the page, from the groups anyone has
shared an Artifact with, by asking Google whether you are in each, and
keeps the answer for five minutes, so a share to a group you were just
added to can take a few minutes to appear (and Google's own group changes
take minutes to reach the Artifact's door anyway). If Google cannot be
asked, the page says it can't be shown right now rather than showing a
shorter list; try again in a minute. Artifacts open to the whole domain
are not "shared with you": find them in Across the domain when their
Publisher lists them, or by their address. Your own Artifacts are in My
Artifacts, never here.

### Across the domain

**Across the domain** lists every Artifact whose Publisher chose to show
it to the whole domain ([Listed](#listed)), newest first, with who
published each, its state and its Let; Expired Artifacts stay, marked.
Everyone in the domain sees the same list. An Artifact open to the whole
domain that is not listed is found by its address alone.

### An Artifact's page

`https://<company-host>/app/#artifact/<id>` (or `/app/a/<id>`, the link on
the expired page) is one Artifact as its Publisher sees it (an Admin sees
any Artifact's page the same way; a member it is shared with sees what
[Restore Requests](#restore-requests) describes; anyone else sees *Not
found*): the title, its state and Let (live until
when, with the Renewals so far; expired since when, and who ended it
early), the App type of its live Revision, who may view it, whether it
is listed in Across the domain, the Publisher, and everything that
happened to it, newest first (publishes, Revisions going live or failing,
shares, Renewals, an Admin's end with their name and reason). From it you
do what the commands below do, without a terminal:

- **Renew** restarts the Let from now ([Renew](#renew)); **End now**
  takes the Artifact down at once, after a confirmation and an optional
  reason ([End](#end)); **Restore** brings an Expired Artifact back
  ([Restore](#restore)) and the page follows the deploy.
- **Share** takes names as `factory share` does (addresses in your
  Workspace domain, `group:<address>`, `domain`), an optional message,
  and the two switches (*replace the current policy*, *email the newly
  named*); **Remove** beside a name unshares it. The page returns at once
  and shows the Access Policy as **applying** until the change is on the
  Artifact's service, with what changed and since when; a reload shows
  it applied.
- **List in Across the domain** and **Unlist** are `factory share
  listed` and `unshare listed` ([Listed](#listed)).

A refusal shows the reason in words (the Let has no Renewals left, the
Artifact is not open to the whole domain, Google does not recognise a
name, a publish is in flight until when), and nothing changes. The
page's *About this page* repeats what each action does. An Expired
Artifact's page also lists its [Restore Requests](#restore-requests), who
asked and when.

### Restore Requests

When an Artifact's Let ends, its link answers with the Installation's
expired page, which names nothing about it but links its page on the Hub.
There a member it was shared with (by address, through a group, or the
whole domain) sees its title, you as its Publisher, and when it ended
(for an Early end only that you or an Admin ended it, and when: never who
or why), and one button to ask for it back: *Ask the Publisher to restore
it*, or *Ask the Publisher to publish it again* when the Installation no
longer keeps its bundle. Anyone it was not shared with sees *Not found*.

Each member asks once until the Artifact is live again. The requests show
on your My Artifacts (*Restore Requests*: who asked) and on the Artifact's
page (who and when), and you get one mail per request when the
Installation has email, from the member via the Installation (by their
address: the Hub's sign-in carries no name), so a reply reaches them. A request restores nothing: [restore](#restore) the
Artifact, publish it again, or leave it. The requests clear once it is
live again, whatever brought it back, and nobody who asked is told; its
link simply works again.

When the Installation does not let Publishers restore their own
Artifacts, the requests go to its Admins instead (their mail and the
Artifact's page), and you do not see them. An Admin can also turn the
button off for the whole Installation.

### Publish without a terminal

The Hub's **Publish** page publishes a static site from the browser, the
way `factory publish` does from a folder, and is the path for a page made
in Gemini Canvas, which hands its code over only through the clipboard:

1. In Canvas, open **Code** at the top right of the Canvas panel, select
   all and copy. Ask Gemini for a single self-contained HTML file if what
   you see is not one: a React component (it starts with `import`) is not
   a page, and the Hub refuses it with that hint.
2. Paste it into **Page HTML**. The **Title** fills itself from the page's
   `<title>`; change it as you like. It names the Artifact.
3. **Publish**. The page shows the Revision deploying, then live, with the
   shareable address and when its Let ends, or failed, with the reason the
   deploy recorded. Share the address; only your Workspace domain can open
   it. A Canvas share link is not an input, and the opposite of an
   Artifact: public to the internet when your admin allows it.

If the page uses Gemini features or saves data, Canvas supplies those at
run time and nothing else does: the Hub warns you before publishing that
those parts will not work once published, and publishes the rest.

Instead of pasting, drop one `.html` or `.htm` file of any name (it is
published as `index.html`), or a `.zip` of a site with `index.html` at its
root (zip the files, not the folder), or choose a folder with `index.html`
at its root. A folder or zip follows the rules of `factory publish`: its
`factory.yaml` is read if it has one and written if not, a `.gitignore`
and a root `.factoryignore` are honoured, the same names are always left
out, and a site that breaks the static rule is refused before anything is
uploaded with the same message. Static sites only: a Python or Streamlit
app is published from the CLI, and the Hub refuses it with
`source_built_disabled` like the CLI does. The files may total 100 MiB.

A publish that fails after the Artifact was created says so, and **Try
again** publishes to that Artifact rather than a second one. Each
publish from the page creates a new Artifact otherwise; to publish a new
Revision of an existing Artifact, use the CLI with `--artifact <id>`.

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
`url=$(factory publish ./site)` works (without a terminal, the Hub's
Publish page does the same for a static site,
[above](#publish-without-a-terminal)). The first publish creates the
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
out in the message. A `streamlit` or `python` kind is refused with
`source_built_disabled` unless the Operator has installed source-built
apps and an Admin has allowed them (the message says which is missing).
Where they have, the app is **built from source**: the publish moves
through building, scanning and deploying (printed as it goes), and the
app runs as a service of its own with no permissions, under the
Installation's limits and egress setting. Its packages come only through
the Installation's PyPI mirror, so a `requirements.txt` or
`pyproject.toml` that names another index (`--index-url`,
`--extra-index-url`), a package source (`--find-links`), a direct URL or
`git+` requirement, or an include of another file (`-r`, `-c`) is
`invalid_bundle` before anything is uploaded: pin each package by name
and version. A build that fails, runs past the Installation's build
timeout, or an app that does not start is `revision_failed` with the
reason, and `factory status` then shows the last lines of your build or
container log; the full log is the Operator's.

Between the scan and the deploy the Installation judges the scan's
findings in your own packages: a known-malicious package, or a finding at
or above the Installation's threshold (`CRITICAL` by default; an Admin
can tighten it to `HIGH`) that has a fix, fails the publish with a
`scan failed` reason. The reason names each package with its advisory,
severity and fixed version: pin the package at or above that version in
`requirements.txt` or `pyproject.toml`, or replace it, then publish
again; a known-malicious package has to go. Findings in the base image
never fail a publish; the scan's full report, base image included, is an
Admin's to read. A scan that does not complete is `scan unavailable`:
publish again, and tell an Admin if it repeats. A Restore rebuilds and is
scanned again; a Renewal is not.

What is published is the folder's regular files, with the rules of a
`.gitignore` in the folder and of a `.factoryignore` at its root honoured,
and a few names always left out (`.git`, `.relay-factory`, agent folders);
`factory.yaml` is always included. A symlink or a special file makes the
folder unpublishable (`invalid_bundle`). The compressed bundle may be at
most 100 MiB.

A publish waits up to 16 minutes for a static site to go live, 31 for a
source-built app (`--timeout`). A Revision that deploys and fails is `revision_failed` with
the reason; a publish still waiting at the timeout is `wait_timeout`, and
the Revision may yet go live. One publish of an Artifact at a time: a
second one while the first is in flight is `deploy_in_progress`.

To publish without waiting, `factory publish --no-wait` returns as soon
as the upload is committed, printing the Artifact and Revision IDs, and
`factory status --wait` in the folder waits for that Revision later, with
the same outcomes a waiting publish has (`revision_failed`,
`wait_timeout`, its own `--timeout`); when nothing is pending it reports
at once.

`--share <who>` on a first publish sets who may view from the start
instead of the whole domain ([Share](#share)): `--share
carol@example.com,group:team@example.com`, repeatable, with `--no-notify`
and `--message` as `factory share` has them; `--share nobody` keeps the
Artifact to yourself. An Artifact that must never be open to the whole
domain never is.

`--let <duration>` gives the Revision a Let of that length instead of the
Installation's default: `--let 2d` ends it two days after it goes live
(`36h`, `1d12h` and `2h30m` work too). The most it may ask for is the
Installation's maximum, which is the default unless the Admin raises it;
a longer Let is refused before anything is uploaded.

The Installation limits what one Publisher may hold and do: a number of
live Artifacts (50 unless the Admin changed it) and a number of publishes
per UTC day (100; a Restore counts, a Renewal does not). When a publish
leaves you within 10 % of either, it prints a `Warning:` line on stderr
naming your count against the limit, so an agent publishing in a loop
gets a signal before the wall. At the first limit, a publish that would
need a new live Artifact (a new folder, or an Artifact that has expired)
is refused with `publisher_at_limit`; publishing again to an Artifact
that is live never is, since it holds its place already. At the second,
every publish is refused with `publish_rate_limited` until the next UTC
midnight, which the message names. The way out is in the message too:
let an Artifact expire or end one you no longer need (`factory end`),
wait for the day to turn, or ask an Admin to raise the limit
([Admin](admin.md#the-policy)).

While the Installation is Paused, because the Operator's Spend cap is
enforced and Google has stopped its Artifacts for the rest of the month,
every publish is refused with `installation_paused` (exit 19) before
anything is uploaded. Nothing in your folder is wrong: the cap lifts when
the Operator lifts it, or on the first of next month, so ask the
Operator. `factory renew` still works meanwhile.

## Share

```sh
factory share ./site carol@example.com group:team@example.com
factory unshare ./site domain
```

Every Artifact starts open to everyone in your Workspace domain, and
nobody outside the domain can ever open one. `factory share` narrows or
widens that: name members by their work address, a group by
`group:<its address>`, or `domain` for everyone; `factory unshare`
takes names off, and `unshare domain` leaves only the names you gave
(you can always view your own Artifact, named or not). `share --only
<who>…` replaces the list in one step, the domain off unless you name
it, and `share --only nobody` keeps the Artifact to yourself. At most
100 names. The folder (default: the current one) or the Artifact's ID
goes among the names.

A name outside the domain is refused, and a name Google does not
recognise (a mistyped or deleted address, or a group's address given
without `group:`) fails the whole command with `unknown_name` and
changes nothing. The command returns once the change is on the
Artifact's service, which takes seconds (`--timeout`, as `publish`);
Google then takes a few minutes to admit a newly named member, and
longer to notice a change in a group's membership. It prints what
changed and the `Viewers:` line on stderr and the link on stdout, or
with `--json` the share facts with `added`, `removed` and `applied`. On
an Artifact that has expired the change is recorded and takes effect
when it is restored.

When the Installation has email, each member or group you newly name
gets a mail from the Installation once the policy is live, with the link,
who may open it and until when; `--message` adds a note and
`--no-notify` holds the mail back. The mail comes from `<you> via
<your Installation>` with you as its Reply-To, so a colleague who replies
reaches you. A group is mailed once at its own address, and the group's
settings decide whether its members see it. Without email the command
says no mail was sent, and a share on an Artifact that has expired mails
nobody, then or when it is restored: share again once it is live. An
account deleted since you named it is dropped the next time the policy is
applied, and the command tells you.

An Admin can change any Artifact's Access Policy, including naming
themselves; they have no standing access otherwise. When an Admin shares,
unshares, lists or unlists your Artifact, the Installation mails you what
changed, in the Admin's name, and the people they named get the Share
mail from the Admin.

### Listed

```sh
factory share ./site listed
factory unshare ./site listed
```

An Artifact open to everyone in the domain is still found only by its
address until you list it: `factory share listed` shows it in the Hub's
Across the domain, where every member can find it, and `factory unshare
listed` takes it out again. Who may view does not change. Only an
Artifact open to the whole domain can be listed: if you have narrowed it
the command is refused with `not_whole_domain` and changes nothing, so
`factory share domain` first; and narrowing a listed Artifact later
(`unshare domain`, `share --only`) unlists it, and the command says so.
Whether a new Artifact starts listed is your Installation's policy
(`listedByDefault`, off unless an Admin turns it on); `factory status`
says whether yours is. The word `listed` goes alone, with at most the
folder or ID; nothing waits and nobody newly named is mailed. An Admin
can list and unlist any Artifact, and the Installation then mails its
Publisher in the Admin's name.

## Status

```sh
factory status ./site
```

Reports, changing nothing: the Installation, who you are signed in as and
where the Sign-in is kept, and, for a linked folder, the Artifact's title,
state, Let end, Access Policy, whether it is listed in the Hub's Across
the domain, and its pending, last failed or current Revision, with a
failed one's reason (`--json` adds the Publisher and the
Artifact as the Installation reports it). For a source-built app the
Revision line names the stage a publish is at (`building`, `scanning`,
`deploying`), and once the build has run, or the app failed to start, the
last lines of your build or container log follow it. A folder with no link file
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
`not_live`: an Expired Artifact is restored or published again, not
renewed. Nothing is written locally and no email is sent.

While the Installation's email is on, it mails you about each of your
Lets twice: once before it ends, by the Installation's warning lead (two
days by default; a shorter Let gets no warning), with the end, the
Renewals left and the `factory renew` line (and, while the Installation is
Paused, that it is, that the Let keeps running and that a renew is still
allowed and free); and once when it runs out,
with how to restore it when the Installation keeps bundles, or to publish
it again. A Let ended early is not mailed this way. A renew or a publish
starts a Let of its own, which is warned again.

## Restore

```sh
factory restore ./site
```

Brings your Expired Artifact back from the bundle the Installation kept
when its Let ended, with nothing to upload: the content is the Revision
that was live when it expired, at the same address, with the same title
and the same Viewers. It is a publish without the upload, and it behaves
like one: the command waits for the restored Revision to go live (up to
`--timeout`, 16 minutes by default; `--no-wait` returns at once and
`factory status --wait` waits later), then prints the link, who can view
it, and with `--json` the same object `publish` prints, with `created`
`false`. The restored Let starts fresh at the Installation's default
length; `--let 2d` asks for a shorter one. The argument is the linked
folder (default: the current folder) or an Artifact ID. Restore when the
content is unchanged; publish again when it changed.

Three refusals are the Restore's own: `not_expired` (31) when the Artifact
is live (renew it, or publish), `restore_disabled` (32) when the
Installation does not let Publishers restore their own Artifacts (ask an
Admin, who restores any Artifact with the same command), and
`nothing_retained` (38) when there is no bundle to restore (the Artifact
was never live, its Retention has passed, or the bundle is gone): publish
it again. A publish of it in flight is `deploy_in_progress`. A Restore
counts as a publish and brings back a live Artifact, so the two limits of
[Publish](#publish) apply to it too: `publisher_at_limit` (17) when you
hold the Installation's limit of live Artifacts, `publish_rate_limited`
(18) when you made its limit of publishes this UTC day. While the
Installation is Paused (the Operator's Spend cap is enforced) a Restore
is refused with `installation_paused` (19), as a publish is: ask the
Operator, and renew a live Artifact meanwhile if it needs it. Nothing is
written locally and no email is sent for your own Restore; when an Admin
restores, transfers or ends your Artifact, you get a mail in their name,
if the Installation has email.

## End

```sh
factory end ./site
```

Ends the Let of your own Artifact now: the link goes down when the command
returns and the service within seconds. The Artifact stays yours, keeps
its address and may be published again. The argument is the linked folder
(default: the current folder) or an Artifact ID. An Artifact with no
running Let is `not_live`; one with a publish in flight is
`deploy_in_progress`. Nothing is written locally and no email is sent for
your own end; an Admin ending your Artifact mails you their name, the
reason and what you may do next, if the Installation has email.

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

`factory admin install` is the Operator's command ([Install](install.md));
`factory transfer` is an Admin's, giving an Artifact to another Publisher
([Admin](admin.md#transfer-an-artifact)). `factory help <command>` prints
a command's flags.

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
| `--share <who>` | on a first publish, who may view instead of the whole domain: addresses, group:&lt;address&gt;, domain or nobody; repeatable, or comma-separated |
| `--no-notify` | with --share: record no Share mail to the named |
| `--message <text>` | with --share: a note for the Share mail |
| `--installation <host>` | the Installation to publish to |
| `--timeout <duration>` | how long to wait for the Revision to go live (default 16m for a static app, 31m for a source-built one) |
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
| `--timeout <duration>` | how long --wait waits for the Revision to go live (default 16m for a static app, 31m for a source-built one) |
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

### `factory restore`

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

### `factory share`

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

### `factory unshare`

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
| `usage` | 2 | CLI | Bad command line, a host that is not an Installation host, or a malformed link file. `factory admin install`: also a missing or invalid input, an unknown Installation, or an OAuth client Google rejects. `factory admin rebase`, `factory admin status` and `factory admin config get`: an unknown Installation, or one whose install never wrote its config. `factory admin uninstall`: those too, a missing or wrong `--confirm`, or an Installation whose projects are already pending deletion; nothing was changed. |
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
| `publisher_at_limit` | 17 | API | `factory publish` and `factory restore`: the Publisher holds the live Artifacts the Installation allows per Publisher (50 by default), so a publish that needs a new one is refused; a republish of a live Artifact never is. `factory transfer`: the receiving Publisher would go over. The message gives the count and the way out: let one expire, `factory end` one, or ask an Admin to raise `liveArtifactsPerPublisher`. |
| `publish_rate_limited` | 18 | API | `factory publish` and `factory restore`: the Publisher has made the publishes the Installation allows per Publisher per UTC day (100 by default; Restores count). The message names when the count resets (the next UTC midnight); wait, or ask an Admin to raise `publishesPerPublisherPerDay`. |
| `installation_paused` | 19 | API | `factory publish` (at create and at issue) and `factory restore`: the Installation is Paused because the Operator's Spend cap is enforced; Google has stopped its Artifacts until the cap is lifted. Nothing to fix in the bundle: the Operator lifts the cap, or it lifts on the first of next month; `factory renew` still works. |
| `network` | 20 | CLI | The Installation, Google or the upload host could not be reached, did not answer in time, or answered with something that is not the contract. |
| `invalid_bundle` | 21 | CLI | The folder cannot be published as it is: its `factory.yaml` is invalid or its kind could not be detected, a `static` app has no root `index.html` or holds Python, a `streamlit` or `python` app's `requirements.txt` or `pyproject.toml` names a package source other than the Installation's mirror (`--index-url`, `--extra-index-url`, `--find-links`, a direct URL or `git+` requirement, `-r`/`-c`), or the folder has a symlink or special file, a whiteout name, an unreadable file, or no regular file. The message names the rule and the way out. |
| `revision_failed` | 22 | CLI | The Revision was uploaded and its publish failed (`publish`, or `status --wait` on it); the message is its failure reason: a bundle the deploy worker refused (its manifest, kind, static rule or dependency sources), or a source-built build, scan or start that failed (`build failed`, `build timed out`, `scan failed`, `scan unavailable`, `app did not start`), whose log tail `factory status` shows. |
| `wait_timeout` | 23 | CLI | The Revision was not live or failed when `--timeout` passed (`publish`, or `status --wait` on it); or the Access Policy was not yet on the service when `share`'s or `unshare`'s `--timeout` passed (the change stands and is applied when the deploy worker catches up). |
| `login_failed` | 24 | CLI | The sign-in was refused, abandoned, timed out, or Google's answer lacked what is needed. |
| `not_expired` | 31 | API | `factory restore`: the Artifact is live, so there is nothing to restore; publish or renew it instead. |
| `restore_disabled` | 32 | API | `factory restore`: this Installation does not let Publishers restore their own Artifacts; ask an Admin to restore it. |
| `renewals_exhausted` | 33 | API | `factory renew`: the Artifact has used the idle Renewals the Installation allows per Let; publish again to start a fresh Let. |
| `not_admin` | 34 | API | `factory admin policy`, `factory admin status`, `factory admin end`, `factory transfer` and `factory restore --publisher`: the member signed in is not a member of the Admin group (`RELAY_ADMIN_GROUP`). Sign in as an Admin, or ask one. |
| `not_live` | 35 | API | `factory end` and `factory renew`: the Artifact has no running Let (never live, or Expired already). Its link is already down; publish again to bring it back. |
| `source_built_disabled` | 36 | API | The `factory.yaml` declares `streamlit` or `python` and this Installation does not build apps from source: the Operator has not installed the capability (`factory admin install --source-built`), or an Admin has the policy's `sourceBuiltAllowed` off; the message says which. Only `static` publishes until then. |
| `nothing_retained` | 38 | API | `factory restore`: the Artifact has no retained bundle (never live, its Retention has passed, or the bundle is gone); publish it again. |
| `unknown_name` | 40 | API | `factory share` and `factory publish --share`: a name is not a member or group Google recognises in the Workspace account (a deleted or mistyped address, or a group's address given as a member); the message names it. The whole command was refused and nothing changed; fix the name and share again. |
| `not_all_ended` | 41 | CLI | `factory admin end`: with `--yes`, at least one Artifact was skipped (a publish in flight; the line says when its lock lapses) or failed, after every one was tried; without it, an ID named is not on the Installation. Each Artifact's outcome is in the output; run it again for the ones that need it. |
| `not_whole_domain` | 42 | API | `factory share listed`: the Artifact is not open to the whole Workspace domain (its domain switch is off), so it cannot be shown in the Hub's Across the domain. Nothing was changed; `factory share domain` first, then list it. |
| `interrupted` | 130 | CLI | SIGINT or SIGTERM stopped the command. A Revision already committed may still go live. |

<!-- cliref:end -->
