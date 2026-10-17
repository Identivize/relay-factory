# Handoff: the OAuth client

**Who does it:** someone with Owner on the Installation's control-plane
project, which is the Operator who ran the install.

**What it is for:** Publishers sign in to the Installation with `factory
login`, which runs Google's installed-app sign-in on a Desktop app OAuth
client that belongs to the Installation. Google's console is the only place
such a client can be created, so the installer cannot do it.

## Steps

The installer's instructions name the control-plane project; `<cp>` below
is its ID.

1. **Set the consent screen's user type to Internal.** In the Google Cloud
   console, in project `<cp>`: APIs & Services, OAuth consent screen
   (`https://console.cloud.google.com/apis/credentials/consent?project=<cp>`).
   Choose **Internal**: only members of the Workspace domain can sign in.
   The app name is what members see on Google's sign-in page.
2. **Create an OAuth client of type Desktop app.** APIs & Services,
   Credentials (`https://console.cloud.google.com/apis/credentials?project=<cp>`),
   Create credentials, OAuth client ID, application type **Desktop app**.
   Note the client ID (it ends in `.apps.googleusercontent.com`) and
   download or copy the client secret.
3. **Save the secret to a file** readable by you alone (mode 0600). It is
   sent only to Google and is never stored by the installer in a file, the
   state or any output.
4. **Hand both to the installer:**

   ```sh
   factory admin install --installation <cp> --oauth-client-id <client-id> --oauth-client-secret-file <secret-file>
   ```

   `-` as the file reads the secret from stdin.

## Done when

Google's token endpoint accepts the client ID and secret together: the
installer asks Google before changing anything, so a wrong ID or secret is
refused as a `usage` failure (exit 2) with nothing stored. Once accepted,
the secret is added as a version of the Installation's
`oauth-desktop-client` secret, the ID is stored in the config, and the
second-phase apply carries the ID to the Front Door. The Handoff is done
after all three, and stays done only while the stored secret belongs to
the ID in the config and in the last apply: a run that stopped between
storing the two says so and asks for the pair again.

The Installation is ready once this Handoff and the [DNS Handoff](dns.md)
are both done and the Front Door answers
([Install](../install.md#6-the-ready-report)).
