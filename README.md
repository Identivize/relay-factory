# Relay Factory

Relay Factory lets members of a Google Workspace domain publish static web
Artifacts into their organization's own Google Cloud and share them with the
domain. Each publish gets a short-lived Let; unused Artifacts expire, and the
organization controls Retention.

This documentation describes release **v0.1.3**. An Installation runs one
release at a time: if yours runs another one, read the documentation at that
release's tag, `https://github.com/Identivize/relay-factory/blob/<tag>/README.md`.
An Installation's release is the one its Operator last installed or
upgraded it with; `factory admin install --check --installation <handle>`
run with another release's `factory` names it in the `release` check.

## Where to start

- **Operators** install and run an Installation in their organization's
  Google Cloud: [Prerequisites](docs/prerequisites.md), then
  [Install](docs/install.md), which forwards a page per Handoff to the people
  who complete them ([DNS](docs/handoffs/dns.md),
  [OAuth client](docs/handoffs/oauth-client.md),
  [Groups Reader](docs/handoffs/groups-reader.md),
  [Email](docs/handoffs/email.md),
  [Artifact capacity](docs/handoffs/artifact-capacity.md),
  [Invoker IAM](docs/handoffs/invoker-iam.md)). Later:
  [Upgrade](docs/upgrade.md), [Uninstall](docs/uninstall.md),
  [Troubleshooting](docs/troubleshooting.md) and
  [What v1 does not do](docs/what-v1-does-not-do.md).
- **Admins** are the members of the Installation's Admin Google Group:
  [Admin](docs/admin.md).
- **Publishers** are members of the domain who publish folders with the
  `factory` CLI, or a page from the Hub's Publish page:
  [Publisher](docs/publisher.md). A coding agent learns the
  CLI from the skills `factory` carries (`factory skills install`,
  [Agent skills](docs/publisher.md#agent-skills)).

## Releases

A release is the `factory` CLI and the service images it installs, pinned by
digest. Every release is tested before it is published: the release
workflow installs the tag's own images on a test Installation on Google
Cloud, as an upgrade, with the same `release.json` it then publishes,
and runs a publish, view and expiry cycle against them; a tag that fails
that gate publishes nothing. The CLI archives, the signed
`checksums.txt` and `release.json` are attached to this repository's
[GitHub Releases](https://github.com/Identivize/relay-factory/releases); the
images are on GitHub's container registry. A tag with a `-` after the
version (an `rc`) is a prerelease. The Homebrew cask in
[Identivize/homebrew-tap](https://github.com/Identivize/homebrew-tap) follows
stable releases only. Fetching and verifying a release is the first step of
[Install](docs/install.md#1-get-factory) and of
[Publisher](docs/publisher.md#get-factory).

The agent skills `factory` and `factory-admin` are mirrored here under
`skills/`, with a Claude Code marketplace file at
`.claude-plugin/marketplace.json`, on stable releases only; the copy inside
each release's `factory` binary is the one that matches it.

The source repository is private; the releases, this documentation and the
issue tracker are public. Issues and questions about Relay Factory are
welcome here.

Licensed under the [Apache License 2.0](LICENSE).
