# WSODeployer server (v0.2)

Management server for WSODeployer: identity and approvals of devices, Offline Domain Join on demand through the domain
agent, deployment journal. .NET 10 (ASP.NET Core minimal API) + PostgreSQL 17 + NHibernate / FluentNHibernate; the
operations that must be atomic under concurrency are PostgreSQL functions (`src/WSODeployer.Data/Sql/002_functions.sql`).

```
src/WSODeployer.Core   entities, key normalization (same as the PowerShell engine), request signatures, assets.csv
src/WSODeployer.Data   FluentNHibernate mappings, migrator (embedded Sql/NNN_*.sql), stored function calls
src/WSODeployer.Api    endpoints, auth filters, static /content
tests/                 xUnit: unit tests + integration tests on PostgreSQL in Docker (Testcontainers)
```

## Install and run (scripts)
Same pattern as GateBridge: `scripts/run-windows.ps1` (Docker Desktop with Linux containers / Windows Server) and
`scripts/run-linux.sh`. The default action `all` is idempotent - run it again after any change; generated secrets are
never overwritten. In the release package (`server\` folder of `WSODeployer-X.Y.Z.zip`) there is no source and no
Dockerfile: the scripts pull the released images `ghcr.io/franeksoftsf/wsodeployer-api` / `-nginx` named in its
`docker-compose.yml` (`docker compose pull`) instead of building them; in a clone of the repository they build.

```powershell
.\scripts\run-windows.ps1        # asks for everything that is missing, shows a summary, then builds (or pulls)
.\scripts\run-windows.ps1 -PublicHost wso.corp.example.com -DeploymentCertificate D:\WSO\cert\deploy.cer
```
```bash
./scripts/run-linux.sh all --host=wso.corp.example.com --deployment-cert=/root/deploy.cer
```
`WSO_PUBLIC_HOST` is the address of THIS server (where the console and the API run), not the Workspace ONE Access tenant.
In a terminal the scripts ask for every missing value (server address, ports, deploy.cer) and confirm before building;
answers stay in `.env`. `-NonInteractive` / `--non-interactive` never ask.

**The console administrator always exists after the script**: the first run creates the local account `admin` with a
random password, prints user name and password at the end and keeps them in `secrets/local-admin.txt` (owner only;
`.env` gets only the PBKDF2-SHA256 hash). Sign in with it at `https://<host>/console` and set up everything else in the
console - sign-in with Workspace ONE Access / Entra ID included (page **Sign-in**, see below). Later runs keep the
password and switch the account on again if `WSO_LOCAL_ADMIN_ENABLED` was false. Lost password: `-Action local-admin`
(`local-admin`) makes a new one and shows it.

**DEV / test instance** - `-Dev` (`--dev`): a separate instance next to production on the same machine (`.env.dev`,
`data-dev`, `tls-dev`, `secrets-dev`, Docker project `wsodeployer-dev`), `localhost` by default (SAML works for a browser on
the same machine - the IdP never calls the server; give it its own SP entity ID, e.g. `.../saml2-dev`, a separate test
app in Workspace ONE Access). Ports are picked on the first run: the first free of 8443 / 9443 / 10443 / 18443 and
8080 / 8081 / 9080 / 18080 (production: 443 / 80 when free, otherwise the same search with a warning); interactive runs
offer them as the default. "Free" = nothing listens on it (Docker Desktop keeps listening on ports of stopped projects).
```powershell
.\scripts\run-windows.ps1 -Dev
.\scripts\run-windows.ps1 -Dev -Action down
```

What `all` does:
1. `.env` from `.env.example`; database password, admin and agent API keys generated - only the SHA-256 goes to `.env`,
   the keys to `secrets/admin.key` and `secrets/agent-1.key` (owner only). Admin: `$env:WSO_ADMIN_KEY` for the client;
   agent: the key file for `Start-WsoDomainAgent.ps1`.
2. The local console administrator (above).
3. `WSO_PUBLIC_HOST`, ports; `data/` (sample content when empty, `deploy.cer`).
4. `docker compose build` + `up -d`, waits for `https://<host>/api/health` - checked against `tls/server.crt`
   with `--resolve`, so no hosts entry is needed on the server; the summary with the administrator password.

Other actions: `build`, `up`, `down`, `restart`, `logs`, `status`, `local-admin`,
`grant-role -User <login> -Role viewer|operator|admin` (directly in the database - the account must exist; empty role
removes it), `backup` (pg_dump to `backups/`). `import-idp`, `export-sp`, `sp-cert` (before 0.24) only point to the
console page Sign-in.

Manual alternative: `copy .env.example .env`, fill it in (the local administrator hash:
`pbkdf2-sha256:<iterations>:<salt base64>:<hash base64>`), put the files in `data/`, `docker compose up -d --build`.
### TLS (nginx)
The compose file starts **nginx** in front of the API (ports 80 -> 443; the API port is not published). On the first
start nginx creates a **self-signed certificate** for `WSO_PUBLIC_HOST` (+ `localhost`, `127.0.0.1`, `WSO_EXTRA_SAN`) in
`./tls`: `server.crt`, `server.key` and `server.cer` (DER for Windows). To use a company certificate, put its
`server.crt` (with the chain) and `server.key` there and `docker compose restart nginx`.

Everything that talks to the server must trust the certificate:
- **WinPE media**: `client\WSODeployer.ps1 -Action BuildMedia -ServerUri https://wso.example.com/ -CaFolder <server>\tls ...`
  (the engine imports `ca\*.cer` into WinPE before the first request)
- **domain agent and admin machines**: `Import-Certificate -FilePath server.cer -CertStoreLocation Cert:\LocalMachine\Root`
- **browsers** of console users: the same Root import (or a GPO), otherwise a certificate warning
- **Workspace ONE Access** only redirects the browser, it does not call the server - import the SP metadata as a file

nginx keeps the request path unchanged (device signatures cover it) and sends `X-Forwarded-Proto/Host`, so cookies are
Secure and the SAML ACS URL is `https://<WSO_PUBLIC_HOST>/Saml2/Acs`. Checked locally: signed device requests, ODJ polling,
content downloads and the SAML challenge through nginx.

Settings (`Wso` section / environment `Wso__...`): `ContentPath`, `DeploymentCertificates[]` (several during a
certificate rollover), `AdminKeyHashes[]`, `AgentKeys[]` (`name:sha256`), `MigrateOnStartup`, `OdjRetryAfterSeconds`.

## API
| Caller | Auth | Endpoints |
|---|---|---|
| WinPE engine | RSA signature with deploy.pfx (`X-Wso-Timestamp`, `X-Wso-Signature`) | `POST /api/device/identify`, `GET /api/device/odj/{assetId}?deploymentId=` (202 + Retry-After until the agent is done) |
| Engine / SetupComplete | `X-Wso-Deployment-Token` (issued once by identify, only for that deployment) | `POST /api/deployments/{id}/events`, `PUT /api/deployments/{id}/state?status=` |
| Domain agent | `X-Wso-Agent-Key` | `POST /api/agent/odj/claim?max=`, `POST /api/agent/odj/{id}/complete` (CMS only - plain blobs are refused) |
| Admin (CLI, console later) | `X-Wso-Admin-Key` | `GET /api/admin/assets[?status=]`, `GET /api/admin/assets.csv`, `POST /api/admin/assets/import` (text/csv), `POST /api/admin/assets/{id}/approve`, `POST /api/admin/assets/{id}/retire`, `GET /api/admin/deployments`, `GET /api/admin/deployments/{id}`, `GET /api/admin/odj` |
| anyone | – | `GET /api/health`, `GET /content/...` (static, secrets in it are encrypted) |

Errors are RFC 7807 problem details with a language-neutral `code` (`signature.invalid`, `asset.notActive`, `odj.failed`...).

## Tests
```powershell
dotnet test WSODeployer.Server.slnx       # Docker must run (PostgreSQL container)
..\tests\Invoke-Tests.ps1 -E2E            # everything incl. the PowerShell engine + agent against the real server
```

## Web console (v0.3)

`https://<server>/console` - dashboard, approvals, devices (import / export assets.csv, retire), deployments with the
event journal, the domain join queue, users and roles, audit log. English / Polish (switch in the header).

**Sign-in**: SAML 2.0 with **Workspace ONE Access** (same Sustainsys.Saml2 setup as WSOGenerator, proven with WS1 Access)
and / or **Microsoft Entra ID** over OpenID Connect - both can be enabled, the login page offers the configured ones.
Entra ID also works over SAML (one SAML IdP at a time). **v0.24: everything is set up in the console, page Sign-in
(`/console/signin`, Admin only)** and stored in the database (table `console_auth`); a change applies at once, without a
restart. Secrets (SP signing key, OIDC client secret) are protected with the data protection keys (`keys` volume) and never
shown. The local administrator always works, so a wrong IdP setting cannot lock you out. Upgrade from an older version: the
`WSO_SAML_*` / `WSO_OIDC_*` values of `.env` (and `data/saml/idp-metadata.xml`, `sp-signing.pfx`) are taken over once on
the first start of 0.24 (audit `auth.imported`), afterwards they are ignored.

**Roles live in the WSODeployer database, not in the IdP**: a new user signs in, gets no role and sees "waiting for access";
an Admin assigns Viewer (read) / Operator (+ approve, import) / Admin (+ retire, users, audit) - or adds the login with a
role on the Users page before the first sign-in. Optional `WSO_CONSOLE_ADMIN` (`Wso:Console:BootstrapAdmins`) becomes
Admin on the first sign-in. The last admin cannot drop the own role. The login must be exactly what the IdP sends as
NameID - with Workspace ONE Access often the **user name** (e.g. `adm_jan.kowalski`), not the e-mail; the "waiting for
access" page shows it.

**Local administrator** (`admin`, created by the install script - see above): always Admin, 5 failed attempts lock the
client address for 15 minutes, every sign-in and failure is in the audit log. Its password can be changed on the page
Sign-in; that password stays until the installer writes a new hash (`-Action local-admin`, server restart takes it over -
audit `auth.localAdminReset`). Switch off: `WSO_LOCAL_ADMIN_ENABLED=false` in `.env` and `docker compose up -d` (the
install script switches it on again).

### Workspace ONE Access (SAML)
1. Console → **Sign-in** → SAML: check the SP entity ID (suggested `https://<host>/saml2`), save - an SP signing
   certificate is created (WS1 Access wants signed AuthnRequests).
2. WS1 Access console: **Catalog → Web Apps → New → SAML 2.0** - import the SP metadata from
   `https://<host>/console/sp-metadata.xml` (or upload the file from "Download SP metadata"), or by hand:
   - Single Sign-On URL (ACS) and Recipient: `https://<host>/Saml2/Acs`
   - Application ID / Audience: the SP entity ID
   - Name ID format: email address (the login in WSODeployer)
   - assign the app to the people who administer deployments (the Hub tile signs in IdP-initiated - supported)
3. Console → Sign-in → IdP metadata: the tenant host (e.g. `tenant.workspaceoneaccess.com` - the server reads its public
   metadata URL, so IdP certificate rollovers are picked up; the container needs outbound HTTPS to the tenant) or a
   metadata file / pasted XML (Entra ID SAML, other IdPs). Then tick "Sign-in with SAML is on" and save.

### Microsoft Entra ID (OpenID Connect)
1. Entra admin center → **App registrations → New registration**, redirect URI (Web) `https://<host>/signin-oidc`.
2. Certificates & secrets → new client secret; API permissions: `openid`, `profile`, `email` (Microsoft Graph, delegated).
3. Console → Sign-in → OpenID Connect: authority `https://login.microsoftonline.com/<tenant-id>/v2.0`, client ID, client
   secret, switch on. The login is `email` / `preferred_username` (UPN).
### Workspace ONE UEM (REST API, v0.24)
Console → **Workspace ONE UEM** (Admin). Only the server talks to UEM - devices never get UEM API credentials.
1. UEM: **Groups & Settings → Configurations → OAuth Client Management → Add** - a client whose role can read devices and
   edit a device (v0.25 also read organization groups and tags, and add devices to tags). Note client ID and secret; the token URL of your region is in the UEM documentation "Datacenter and Token
   URLs" (e.g. `https://emea.uemauth.workspaceone.com/connect/token`); the REST API URL is under **All Settings → System →
   Advanced → API → REST API** (e.g. `https://as1234.awmdm.com`).
2. Console: per WS1 environment (Default = deploy.json `Ws1`, plus the further ones) REST API URL, token URL, client ID,
   secret (stored data-protected, never shown), optional OG ID (number) for the device list, platform filter (`WinRT` =
   Windows desktop), "Set the UEM AssetNumber". **Test** gets a token and counts the devices.
3. **Import devices**: reads the device list (`GET /api/mdm/devices/search`, paged, up to `Wso:Uem:MaxPages` x
   `PageSize`) and matches it by serial number - *new* (checked), *exists*, *other asset number* (not checked), *no serial*,
   *placeholder serial* (firmware values such as "System Serial Number" - shared by many machines, never matched; the list is
   `Keys.SmbiosPlaceholders` = the engine's Gather). UEM keeps one record per enrollment: a re-enrolled machine is one row
   (the enrolled, most recently seen record; the older ones are counted).
   Checked new ones become approved devices (asset number and computer name from UEM when it has them; profile / OU as
   usual); existing ones are only linked (their fields stay; a missing asset number or computer name is taken from UEM).
   The computer name is the one the device reports to UEM (`DeviceReportedName`, column *Computer name (UEM)*), used only
   when it is a valid NetBIOS name that no other device has (names are unique).
4. **After every successful deployment** (`PUT /api/deployments/{id}/state?status=succeeded`, end of SetupComplete) the
   server looks the device up by serial number (`GET /api/mdm/devices?searchby=Serialnumber&id=`): after
   `Wso:Uem:FirstCheckSeconds` (120), then every 2 / 5 / 15 minutes, hourly, every 2 hours - about a week (Entra OOBE
   enrollment waits for the user). Found: UEM device ID, enrollment state, OG, user, last seen are kept and, when it
   differs, `PUT /api/mdm/devices/{id}` `{"AssetNumber": "<inventory number>"}`. API errors are shown on the device and
   retried. The device page has **Check now**. Audit: `uem.scheduled`, `uem.found`, `uem.assetNumber`, `uem.notFound`,
   `uem.import`, `uem.settings`, `uem.tag`, `uem.groups`.
5. **Tags (v0.25)**: with "Add a found device to its Workspace ONE tag" on, a found device whose own or group setting
   *Workspace ONE tag* is set is added to that tag once (`POST /api/mdm/tags/{id}/adddevices`; the tag is looked up by name in
   the tags of the environment's OG - a missing tag is shown on the device). The OAuth client role needs tag rights.
6. **Groups from UEM (v0.25)**: Groups → **Groups from UEM** reads the organization groups
   (`GET /api/system/groups/search` finds the top OG - the configured OG ID, else Global, else the Customer OGs -
   then `GET /api/system/groups/{id}/children` gives the tree with parents) and the tags (`GET /api/mdm/tags/search`), kept
   10 minutes. Picked OGs become device groups: WS1 group = the OG's Group ID, optional tag and profile; an OG already used
   by a group is not offered. The group form offers the same OGs and tags as lists (free text still allowed).
   **Membership by OG**: a device linked to UEM that has no group joins the group whose WS1 group is the Group ID of its OG
   (the device's `LocationGroupId`), or of the nearest parent OG with a group - after an import, after groups are created in
   the wizard, when a deployed device is found, and with **Add devices to groups by OG**. A device already in a group keeps
   it. Audit: `uem.groupMembers` (plus `assets.assign` per group).
Read from a real tenant on 2026-11-24: token, device list, OG tree, tags. Lookup by serial number, AssetNumber and
`adddevices` only against the stand-in of the tests so far.
### Notes
- Behind the TLS reverse proxy the forwarded headers are trusted (`Wso:Console:TrustForwardedHeaders`), so cookies are
  Secure and the ACS URL is https. Restrict access to the container port to the proxy.
- Console sessions, SAML state and the sign-in secrets use the data protection keys in the `keys` volume (survive restarts; losing the volume means a new SP certificate / client secret). They are not
  encrypted at rest inside the volume - protect the Docker host accordingly.
- Tested locally: SAML sign-in with a signed response (and rejection of a wrong signature), roles, approvals, CSRF, CSP,
  Docker compose. **Not yet tested against a real Workspace ONE Access or Entra ID tenant.**
