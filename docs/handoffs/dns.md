# Handoff: DNS

**Who does it:** whoever manages DNS for the company host (for example
`artifacts.example.com`) and for the Artifact domain (`a.artifacts.example.com`
unless the Operator chose another). They need no Google Cloud access.

**What it is for:** the company host is where members sign in and every
shareable link starts; each Artifact answers at its own host under the
Artifact domain; and the two `_acme-challenge` records let Google issue the
load balancer's certificates for both.

## The records

The installer's report lists four records with their exact values, which
the Operator forwards with this page:

| Type | Name | Points at |
|---|---|---|
| CNAME | `_acme-challenge.<company-host>` | Google's certificate-authorization target for the company host |
| CNAME | `_acme-challenge.<artifact-domain>` | Google's certificate-authorization target for the Artifact domain |
| A | `<company-host>` | The load balancer's IPv4 address |
| A | `*.<artifact-domain>` | The same address |

Create them exactly as listed: name, type and value. Each CNAME must be
exactly the target given, and each name must answer with that one value:
a second A record on the same name (a parked-page record on the Artifact
domain, say) sends some Viewers elsewhere and keeps the Handoff pending.
Remove any record that already exists on those names.

## Done when

The installer verifies this Handoff in the order it completes, through a
public resolver (not the Operator's own), so what it sees is what Google's
certificate issuance sees:

1. Public DNS returns both CNAMEs.
2. Both certificates are active. Google issues them about 20 to 25 minutes
   after the CNAMEs resolve; allow up to an hour before suspecting the
   records.
3. The company host and a random label under the Artifact domain (which
   only the wildcard can answer) resolve to the load balancer's address.

Until then the report shows the Handoff as pending with a `Now:` line
saying which of the three it is waiting on, with the answer it got and the
one it expected. TLS answers a few minutes after the certificates are
active, and only then is the Installation ready.

The Operator re-verifies with `factory admin install --installation
<handle>`, or lets one run wait: `--wait 40m` re-checks every 30 seconds
([Install](../install.md#5-resume-and-wait)).
