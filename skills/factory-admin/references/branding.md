# The branding and its three policy fields

Three fields of the policy name and dress the Installation
(`factory admin policy set`; the spec's section 5.12):

- `displayName` (empty: the Workspace domain stands in): one line of at
  most 80 characters, no leading or trailing space. Quote it for the shell
  when it holds a space: `"displayName=Example Corp"`.
- `logoPng` (none): the path of a PNG file of at most 64 KiB, which the CLI
  reads and sends as base64. An SVG, a JPEG or a larger file is refused.
- `accentColor` (none): `#rrggbb`.

```sh
factory admin policy set "displayName=Example Corp" logoPng=logo.png accentColor=#1a73e8 --installation <host> --json
```

Where each shows:

- **The Hub**, on every page: the name and the logo in the header, the name
  in the browser's tab, the accent on primary buttons, selected tabs and focus rings.
- **The public pages** on the Installation's host, expired, not found and
  paused: the name and the logo. Their colours stay their own.
- **Every mail** the Installation sends names it by the display name.

The Hub and the public pages read the branding at most once a minute, so a
change shows within a minute. A refused value is `invalid_request` (7)
naming the field, and nothing is written. An empty value clears a field:
`logoPng=` removes the logo, `displayName=` brings back the Workspace
domain. `policy get` shows the logo as `PNG, <n> bytes`, never its bytes.
