# factory exit codes

Every failure of a Publisher command has a stable code: the process exit
code, and `error.code` in the `--json` object, with a message a person or
an agent can act on. "API" codes are the Installation's, passed through
with its message; "CLI" codes are the CLI's own. The installer's codes
(25 to 30) belong to `factory admin install` and are in the factory-admin
skill. The table is generated from the CLI's own exit-code table.

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
