---
name: factory
description: "Publish a folder as a Relay Factory Artifact with the factory CLI: run factory status first, sign in with factory login, publish with factory publish, read the --json result and the exit codes. Use when asked to publish, update, check or end an Artifact, or whenever the factory command is involved."
license: Apache-2.0
metadata:
  version: "v0.1.0"
---

# factory: publish a folder as an Artifact

`factory` publishes a folder as an Artifact on a Relay Factory
Installation: a web page that every member of the organization's Google
Workspace domain can open at `https://<company-host>/a/<id>` after signing
in with Google. Each publish is a Revision; an Artifact lives for its Let
(15 days by default) and expires afterwards. The CLI never prompts:
progress and diagnostics go to stderr, the result to stdout, and `--json`
makes the result exactly one JSON object, on success and on failure. Every
failure has a stable code and exit code.

This skill describes the `factory` of its own release: the version in this
file's frontmatter is the one the CLI's `version` command prints. If they
differ, refresh the skill with the CLI's `skills install` command.

## 1. First move: `factory status`

Always start with

```sh
factory status [dir] --json
```

It is read-only: it changes nothing, never prompts, never waits for a
deploy and never opens the keychain (`--wait`, section 3, is the one mode
that does both). Branch on it:

| Result | Meaning | Next |
|---|---|---|
| exit 0, `link` null | Signed in; the folder has never been published. | Publish (section 3). |
| exit 0, `artifact` an object | Signed in; the folder is linked to that Artifact. Read `artifact.state`, `artifact.letEndsAt`, `artifact.url` and `artifact.revision`. | Publish again to update it, or report its state. |
| exit 0, `link` set, `artifact` null | Signed in, but the Sign-in is in the OS keychain, which `status` never reads; stderr says so. | `factory publish` reads the Artifact when it runs (and is `internal` naming the store when this session cannot reach that keychain: see section 4). |
| exit 3 `unauthenticated` | Not signed in to the Installation, or the Sign-in is stale. | Sign in (section 2), then `status` again. |
| exit 2 `usage` | A malformed link file, or `--installation` names another Installation than the folder's link. | Read the message; fix the folder or drop the flag. |

The `--json` object:

```json
{"installation": "artifacts.example.com",
 "signIn": {"store": "file", "current": true, "email": "pat@example.com", "name": "Pat Publisher"},
 "link": {"installation": "artifacts.example.com", "artifactId": "q3-roadmap-x7k2q9"},
 "artifact": {"id": "q3-roadmap-x7k2q9", "title": "Q3 Roadmap", "kind": "static",
              "state": "live", "url": "https://artifacts.example.com/a/q3-roadmap-x7k2q9",
              "letEndsAt": "2026-10-07T12:00:00Z", "idleRenewals": 0,
              "accessPolicy": {"domain": true, "listed": false},
              "publisher": {"email": "pat@example.com", "name": "Pat Publisher"},
              "revision": {"revisionId": "…", "state": "live"}}}
```

`link` is the folder's link file (`<dir>/.relay-factory/artifact.json`),
or `null`. `artifact` is the Installation's own record of the Artifact, or
`null` when the folder has no link file or its Sign-in is in a keychain.
`artifact.state` is `new`, `live` or `expired`; an Expired Artifact adds
`expiredAt` and, after an early end, `endedBy`. `artifact.revision` is the
pending Revision, else the one that failed since the current went live
(with `failureReason`), else the current one; absent when none was ever
published. `letEndsAt` is absent while no Let runs. A pending Revision
(`revision.state` `issued` or `committed`) is a publish in flight:
`factory status --wait` waits for it (section 3). For a `streamlit` or
`python` Artifact, `revision.stage` says where a publish is (`uploading`,
`building`, `scanning`, `deploying`, then `live` or `failed`) and
`revision.logTail`, once the build has run or the app failed to start, is
the last lines of the Publisher's own build or container log: read it
before changing anything after a `build failed` or `app did not start`.
The human report prints the stage on the Revision line and the tail under
it. A `scan failed` reason is the scan gate's: between the scanning and
deploying stages the Installation judges the scan's findings in the app's
own packages, and a known-malicious package, or a finding at or above the
policy's threshold that has a fix, fails the publish. The reason names
the package, the advisory, the severity and the fixed version: pin the
package at or above that version in `requirements.txt` or
`pyproject.toml`, or replace it, then publish again; a known-malicious
package must be removed. Do not try to lower the threshold: only an Admin
sets it, to `CRITICAL` or `HIGH`, never off.

## 2. Sign in: `factory login`

```sh
factory login <host>
```

**Finding the host.** The Installation is named by its Front Door host
(`artifacts.example.com`), which comes from the Installation's Admin.
Before asking anyone, look for it in the Source folder's agent
instructions, `AGENTS.md` or `CLAUDE.md` (and the files they import): a
line naming the Relay Factory Installation or its host. If they have none,
ask the Publisher once. The CLI remembers it: after one login that
Installation is the default on this machine, `factory login` with no host
signs in to it again, and `publish` and `status` need no host. Never
write the host into `AGENTS.md`, `CLAUDE.md` or any other file: those are
the Publisher's own, and a Publisher may use several Installations.

**The browser step is the Publisher's.** `factory login` prints the sign-in
URL on stderr, opens the browser on this machine and waits up to five
minutes for Google to redirect back to a listener here. Tell the Publisher
to finish the sign-in in that browser; a browser on another device cannot
finish it. Pass `--no-browser` when no browser can be opened here (the
Publisher opens the URL by hand, on this machine). The sign-in URL carries
no secret. Success prints `Signed in to <host> as <email>` on stdout (with
`--json`: `{"installation", "email", "name", "releaseVersion"}`); the
refresh token goes to the OS keychain where one answers, else to the
credential file, with no flag and no message. A consumer Google account, or
one outside the Installation's domain, is refused with `wrong_domain`
(exit 4).

After a successful login, one stderr line may say that an installed copy
of this skill is older than the binary; it is advice, not a failure: run
`factory skills install`.

## 3. Publish: `factory publish`

```sh
factory publish [dir] --json
```

Publishes `dir` (default `.`) as a new Revision and **blocks until it is
live or failed, up to 16 minutes for a static app and 31 for a streamlit
or python one** (`--timeout`), which is built from source first: its
stages (`Building…`, `Scanning…`, `Deploying…`) go to stderr as they
come. The first publish from a
folder creates the Artifact and writes the folder's link file right away;
later publishes from the folder are Revisions of that Artifact.
`--artifact <id>` publishes to an Artifact of the Publisher's without using
the folder's link. On success the human mode prints the shareable URL
alone on stdout, so `url=$(factory publish)` works; stderr carries the
progress and the Viewers line.

**Publishing without waiting.** `factory publish --no-wait` returns as
soon as the upload is committed (exit 0), with `Artifact <id>, Revision
<rev>: committed` on stdout, or the `--json` object below without the
share facts and `letEndsAt`, and makes no further request. Then `factory
status --wait [dir] --json` waits for that Revision and exits as the
publish would have: 0 and the status object once it is live,
`revision_failed` (22) with the reason, `wait_timeout` (23) at its own
`--timeout` (default 16 or 31 minutes, by kind); with nothing pending it reports at once,
and a Revision that already failed is 22 at once. Unlike plain `status`,
`--wait` reads the Sign-in as `publish` does, keychain included. Use the
pair when the agent has other work to do while the deploy runs; otherwise
a plain blocking `publish` is simpler.

**Sharing from the start.** `factory publish --share <who>…` on a first
publish (a folder with no link file) creates the Artifact with that
Access Policy instead of the whole domain, so an Artifact that must
never be open to everyone never is: the names as section 5's `share`
takes them (`--share carol@example.com,group:team@example.com`,
repeatable; `--share nobody` is the Publisher alone), with `--no-notify`
and `--message`. On a linked folder it is `usage` (2): use `factory
share`. The Viewers line and `accessPolicy` then name the list.

**A shorter Let.** `--let <duration>` gives the Revision a Let of that
length instead of the Installation's default: `--let 2d` ends it two days
after it goes live; `36h`, `1d12h` and `2h30m` work too (whole seconds, at
least `1s`; no weeks). The most it may ask for is the Installation's
maximum, which is the default unless the Admin raised it; a longer one is
`invalid_request` (7) before anything is uploaded, and a malformed one is
`usage` (2).

**The manifest.** `factory.yaml` at the folder's root declares the App:
`title`, `kind` (`static`, `streamlit` or `python`) and, for the last two,
`entrypoint` (the file to run for `streamlit`, the start command for
`python`); nothing else may be in it. A folder with none gets one on its
first publish, by detection (a root `index.html` is `static`;
`requirements.txt` or `pyproject.toml` is `python`, or `streamlit` when the
dependencies name it), written before packing with a stderr line `Wrote
factory.yaml: …`; `--title` (default the folder's name), `--kind` and
`--entrypoint` set its fields only then. Afterwards edit the file, not the
flags. A manifest that is wrong, a `kind` that could not be detected, or a
`static` folder that breaks the static rule (an `index.html` at the root,
no Python files) is `invalid_bundle` (exit 21) before anything is
uploaded; the message names the rule and the way out. A `streamlit` or
`python` manifest is refused by the Installation with
`source_built_disabled` (exit 36) while it does not build apps from
source: the Operator has not installed the capability, or an Admin has
the policy's `sourceBuiltAllowed` off (the message says which, and who
can change it; neither is the Publisher's to change); the manifest stays
in the folder. Where both are on, the publish is admitted and built from
source. **The dependency-source rule**: a `streamlit` or `python` app's
packages come only through the Installation's PyPI mirror, so a
`requirements.txt` or `pyproject.toml` that names another index
(`--index-url`, `--extra-index-url`), a package source (`--find-links`, a
`[tool.*.source]` or `[tool.uv.index]` table), a direct URL or `git+`
requirement, or an include of another file (`-r`, `-c`) is
`invalid_bundle` (21) before anything is uploaded; pin each package by
name and version instead. Never work around it by vendoring a wheel from
a URL.

```json
{"artifactId": "q3-roadmap-x7k2q9", "revisionId": "…", "state": "live",
 "url": "https://artifacts.example.com/a/q3-roadmap-x7k2q9",
 "letEndsAt": "2026-10-07T12:00:00Z", "created": true,
 "title": "Q3 Roadmap", "accessPolicy": {"domain": true, "listed": false},
 "publisher": {"email": "pat@example.com", "name": "Pat Publisher"}}
```

`created` says whether this publish created the Artifact. `title`,
`accessPolicy` and `publisher` are the share facts, read after the publish;
when that read failed they are absent and stderr says so, and the publish
has still succeeded. `accessPolicy.domain` true means every member of the
domain can open the link.

**The Usage limits.** The Installation limits each Publisher to a number
of live Artifacts (50 by default) and of publishes per UTC day (100 by
default; Restores count, Renewals do not). A publish that would need a
new live Artifact (a new folder, an Expired Artifact) is refused with
`publisher_at_limit` (17) at the first limit; every publish is refused
with `publish_rate_limited` (18) at the second. A republish of a live
Artifact never meets the first. Before the wall there is a signal: when a
publish leaves the Publisher within 10 % of either limit, stderr carries
one `Warning: <n> of <limit> live Artifacts held, …` or `Warning: <n> of
<limit> publishes today, …` line, in `--json` mode too, and the object
gains `usage` (`liveArtifacts`, `liveArtifactsLimit`, `publishesToday`,
`publishesPerDayLimit`, `resetsAt`) and `warnings` (the same lines
without the prefix). An agent loop that sees one should slow down or
tell the Publisher, not publish on until it is refused. Apart from the
limits, while the Installation is Paused (the Operator's Spend cap is
enforced, so Google has stopped its Artifacts for the month) every
publish and Restore is refused with `installation_paused` (19): nothing
in the folder is wrong, so tell the Publisher to ask the Operator.

**What is packed:** the folder's regular files, minus every `.gitignore`
rule (git's semantics), minus a root `.factoryignore`, and never `.git`,
`.relay-factory`, `.claude`, `.agents` or `.gemini` at the root. The
compressed bundle may be at most 100 MiB. A symlink, a special file or a
folder with no regular file left is `invalid_bundle` (exit 21), before
anything is uploaded; its message names `.factoryignore` as the way to
bring back a gitignored build output (`!/dist/`).

**Report the link only after a publish.** Tell the Publisher the URL once
`publish` has exited 0, and quote `url` from its output. Before that there
is nothing to share: an unpublished folder has no link, and a publish that
failed has no live Revision.

## 4. Reading a failure

Every failure is `{"error": {"code": "…", "message": "…"}}` with `--json`
(exit code = the code's number), or `factory: <message>` on stderr. A
publish that got far enough adds `artifactId`, `revisionId` and `url` next
to `error`. The codes to branch on:

| Exit | Code | Do |
|---|---|---|
| 3 | `unauthenticated` | Sign in again (section 2). |
| 10 | `deploy_in_progress` | Another publish of this Artifact is in flight. **Never publish again**; the message says when its lock lapses. Run `factory status` later. |
| 17 | `publisher_at_limit` | The Publisher holds the Installation's limit of live Artifacts (`publish` of a new or Expired Artifact, `restore`, `transfer`). Let one expire, `factory end` one the Publisher no longer needs, or ask an Admin to raise the limit; the message names the count. **Never create a new Artifact to get around it.** |
| 18 | `publish_rate_limited` | The Publisher made the Installation's limit of publishes for this UTC day (`publish`, `restore`). Stop publishing until the time the message names, or ask an Admin to raise the limit; **do not retry in a loop**. |
| 19 | `installation_paused` | `publish`, `restore`: the Installation is Paused, because the Operator's Spend cap is enforced; its links show a paused page until the Operator lifts the cap or it lifts on the first of next month. **Do not change the folder or retry in a loop**: tell the Publisher to ask the Operator. `factory renew` still works. |
| 21 | `invalid_bundle` | The folder, its `factory.yaml`, the static rule or the dependency-source rule: fix what the message says, then publish. |
| 22 | `revision_failed` | The Revision was committed and its publish failed (`publish`, or `status --wait` on it); the message is the failure reason. For `build failed`, `build timed out` or `app did not start`, run `factory status --json` and read `revision.logTail` (the last lines of the build or container log) before changing anything; `scan failed` names the package, its advisory, its severity and its fixed version: pin it at or above that version in `requirements.txt` or `pyproject.toml`, or replace it (a known-malicious package must be removed); never ask for the threshold to be lowered, which only an Admin can do, to `CRITICAL` or `HIGH` and never off; `scan unavailable` or a build that "did not run" means publish again, and tell an Admin if it repeats. Fix the folder, then publish. |
| 23 | `wait_timeout` | Not live at `--timeout` (`publish`, or `status --wait`). Do not publish at once: run `factory status --wait` to keep waiting, or `factory status`; the Revision may yet go live, and publishing again while its lock is young meets `deploy_in_progress`. |
| 31 | `not_expired` | `restore`: the Artifact is live, so there is nothing to restore. Renew it, or publish. |
| 32 | `restore_disabled` | `restore`: this Installation does not let Publishers restore their own Artifacts. Ask an Admin to restore it; do not publish again on your own. |
| 33 | `renewals_exhausted` | `renew`: the Let has used the idle Renewals the Installation allows (2 by default). Publish again to start a fresh Let. |
| 36 | `source_built_disabled` | The manifest's `kind` is `streamlit` or `python` and this Installation does not build from source: the message says whether the Operator has not installed the capability or an Admin has it off. Tell the Publisher, naming which; do not change the kind to `static` for a Python app. |
| 38 | `nothing_retained` | `restore`: no retained bundle (never live, its Retention has passed, or the bundle is gone). Publish again. |
| 20 | `network` | The Installation, Google or the upload host did not answer as expected. Retry once later. |
| 1 | `internal` naming a keychain | `publish`, `restore`, `transfer`, `end` or `renew`: the Sign-in is in a keychain this session cannot reach (an SSH session on a Mac). `factory login` in this session moves the Sign-in to the file. |
| 130 | `interrupted` | The command was stopped. A Revision already committed may still go live: `factory status`. |

Every code, who reports it and when: [exit-codes.md](references/exit-codes.md).

## 5. After a publish

- **`factory status [dir]`** at any time, as in section 1.
- **`factory renew [dir|id] [--let <duration>]`** restarts the Artifact's
  Let from now at the Installation's default length, with no new upload;
  `--let` shortens it. A Let may be renewed a bounded number of times (the
  Installation's policy, 2 by default). Renew when `status` shows
  `letEndsAt` coming up and the content is unchanged; publish again when
  the content changed or the Renewals are used up. `renewals_exhausted`
  (33) at the cap; `not_live` (35) when it has no running Let (an Expired
  Artifact is restored or published again, not renewed).
- **`factory restore [dir|id] [--let <duration>]`** brings an Expired
  Artifact back from the bundle the Installation retained, with no upload:
  the same content, address, title and Viewers, on a fresh Let of the
  default length (`--let` shortens it). Restore when `status` shows the
  Artifact expired and the content is unchanged; publish again when it
  changed. It behaves like `publish`: it waits for the Revision to go live
  (`--no-wait` and `--timeout` as there, the same `revision_failed` and
  `wait_timeout`), then reports the link, the Viewers line and `publish`'s
  `--json` object with `created: false`. Its refusals: `not_expired` (31)
  when the Artifact is live (renew or publish instead), `restore_disabled`
  (32) when the Installation does not let Publishers restore their own
  (ask an Admin), `nothing_retained` (38) when there is no bundle to
  restore (publish again); `deploy_in_progress` (10) while a publish of it
  is in flight; and the Usage limits as for a publish, since a Restore
  counts as one and takes a live Artifact: `publisher_at_limit` (17) and
  `publish_rate_limited` (18), section 4, and `installation_paused` (19)
  while the Installation is Paused. Its response carries no
  `usage`, so it prints no near-limit warning.
- **`factory end [dir|id]`** ends the Artifact's Let now: the link goes
  down, the Artifact stays the Publisher's and may be published again. Only
  on the Publisher's say-so. `not_live` (35) when it has no running Let;
  `deploy_in_progress` (10) while a publish is in flight. An Admin may end
  any Artifact the same way; `status` then shows `endedBy` as their
  address, the Artifact's activity record carries their reason, and the
  Publisher is emailed in the Admin's name when the Installation has
  email (the command then prints what became of that mail on stderr; the
  Publisher's own end mails nobody).
- **`factory share <who>… [dir|id]`** and **`factory unshare <who>…
  [dir|id]`** change who may view (the Access Policy, ADR-0007). Every
  Artifact starts open to everyone in the Workspace domain; nobody
  outside it can ever view. A name is a member's address in the domain,
  `group:<address>` for a group of it, or `domain` for everyone; `share`
  adds, `unshare` removes, `unshare domain` leaves only the named list
  (the Publisher always views), `share --only <who>…` replaces the list
  with the domain off unless named, and `share --only nobody` is the
  Publisher alone. At most 100 names. The folder or Artifact ID goes
  among the names. Refusals: `invalid_request` (7) for a name outside
  the domain or the 101st, `unknown_name` (40) when Google does not
  recognise a name (mistyped, deleted, or a group given without
  `group:`): the whole command is refused and nothing changes, so fix
  the name and run it again; `not_publisher` (5) for another member's
  Artifact unless the caller is an Admin; `unavailable` (16) when Google
  could not be asked. The command returns once the policy is on the
  service, polling the Artifact read up to `--timeout` (default 16
  minutes, `wait_timeout` (23) then; the change stands and is applied
  shortly), and Google takes minutes more to admit a named member. On an
  Expired Artifact the change is recorded and applies at the Restore;
  the command says so and does not wait, and no mail goes then or at the
  Restore. Stderr gets what changed, the `Viewers:` line and whether a
  Share mail is pending (the Installation mails each newly named member
  or group once the policy is live, when it has email, from `<sharer> via
  <Installation>` with the sharer as Reply-To; `--no-notify` holds it
  back, `--message` adds a note; without email it says no mail was sent),
  and, for an Admin's change to another member's Artifact, whether that
  Publisher was emailed; stdout the link, or with `--json` the share
  facts (`title`, `accessPolicy`, `letEndsAt`, `publisher`) with `added`,
  `removed`, `applied`, `dropped` (accounts deleted since they were
  named, dropped at the apply), `mail` and, for an Admin's change,
  `publisherMail`. Report the Viewers line's content to the Publisher
  when they ask who can see an Artifact; the `accessPolicy` object under
  `status --json` says the same.
- **`factory share listed [dir|id]`** and **`factory unshare listed
  [dir|id]`** show the Artifact in, or take it out of, the Hub's Across
  the domain, where every member can find it (unlisted, it is found by
  its address alone); who may view is unchanged. The word `listed`
  stands alone, with at most the folder or ID and no share flags
  (`usage` (2) otherwise). Only an Artifact open to the whole domain can
  be listed: `not_whole_domain` (42) on a narrowed one, nothing changed,
  so `factory share domain` first and then list it; narrowing a listed
  Artifact later unlists it, and that command's change line says so.
  Whether a new Artifact starts listed is the Installation's policy
  (`listedByDefault`). Nothing waits and nobody newly named is mailed;
  an Admin's listing of another member's Artifact mails its Publisher,
  and the command says so. Stderr says the outcome and the Viewers line;
  stdout the link, or with `--json` the share facts with `listed` and
  `changed` (and `publisherMail` for an Admin's change). `status` prints
  a line after the Viewers line saying whether the Artifact is listed,
  and `accessPolicy.listed` is in every `--json` object.
- **`factory logout`** forgets the Sign-in on this machine (nothing is
  revoked); **`factory version`** prints the release.

The commands this release has, with their flags, are in
[commands.md](references/commands.md), and `factory help <command>` prints
the same.

## 6. Traps

1. **A publish blocks for up to 16 minutes (31 for a source-built app).** Do not kill it and do not run
   a second one; wait for it, give a shorter `--timeout` and then follow
   `wait_timeout` above, or publish with `--no-wait` and come back with
   `factory status --wait`.
2. **Never republish on `deploy_in_progress`.** The Installation allows one
   publish of an Artifact at a time; the message says when the lock lapses.
3. **No secret ever appears in output.** The refresh token, ID tokens and
   the signed upload URL are never printed; never read or quote the
   credential file (`credentials.json`), and never commit it. The link
   file holds no secret and may be committed.

## 7. Gemini Canvas

- A React export from Gemini Canvas is not a page: a static Artifact needs
  a root `index.html` (the static rule, checked before any upload), so
  build the export first and publish the build output (bring it back from
  `.gitignore` with `!/dist/` in `.factoryignore` if needed).
- An app's Gemini-powered features stop working outside Canvas: what calls
  Gemini from the page has no key on the Artifact's origin.
- A single self-contained HTML page from Canvas needs no terminal: the
  Hub's Publish page (`https://<company-host>/app/#publish`) takes it
  pasted, names it `index.html`, writes the manifest and runs the same
  publish sequence; the member can do that themselves.

## Reference

- [commands.md](references/commands.md): every command of this release
  with its flags.
- [exit-codes.md](references/exit-codes.md): every exit code.
- `factory help`: its last line is the Operator documentation's root at
  this release, whose Publisher page covers fetching and verifying
  `factory`.
