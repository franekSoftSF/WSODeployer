# WSODeployer server – setup from the release

Polski: [pl/server-setup.md](pl/server-setup.md)

This guide takes you from an empty host to the first sign-in to the web console. The rest (admin workstation, content,
Cloud Connector, boot media) is in [installation.md](installation.md).

## 1. What to download

Everything is in the **Releases** of the public repository `franekSoftSF/WSODeployer`. Every ZIP has a `.sha256` next
to it.

| File | What it is | Where it goes |
|---|---|---|
| `WSODeployer-server-X.Y.Z.zip` | **The server alone**: `server\docker-compose.yml`, `server\.env.example`, `server\scripts\run-linux.sh` and `run-windows.ps1`, the connector MSI and the technician app in `server\data\downloads`, the sample content in `content\` | the Docker host |
| `WSODeployer-X.Y.Z.zip` | **The full admin package**: the server above, plus the scripts of the admin workstation (`client\`), of the domain agent / connector (`agent\`), the device engine (`engine\`) and the UEM console extension | the admin workstation (and the Docker host, if it is the same machine) |
| `docker-compose.yml`, `.env.example` | The same two files on their own, to read before downloading anything | - |

The server images are pulled by the scripts. You do not build anything:
`ghcr.io/franeksoftsf/wsodeployer-api:X.Y.Z` and `ghcr.io/franeksoftsf/wsodeployer-nginx:X.Y.Z`, plus `postgres:17-alpine`.

## 2. Requirements

### Host

| | Minimum (lab, up to ~50 devices) | Recommended (production) |
|---|---|---|
| CPU | 2 vCPU | 4 vCPU |
| RAM | 4 GB | 8 GB |
| Disk for the server | 40 GB | 100 GB + about 6 GB per Windows image and language kept in the LAN |
| OS | Ubuntu 22.04/24.04, Debian 12, RHEL 9 (Docker Engine), or Windows 10/11 Pro/Enterprise with Docker Desktop | a Linux VM (Ubuntu 24.04 LTS) |

The sizes are estimates for planning, not limits. Measured on a test server: the three containers use about 200 MB RAM when idle (API ~125 MB, PostgreSQL ~50 MB,
nginx ~25 MB). The images take about 0.9 GB. Most of the disk goes to the LAN copies of Windows images (`data/os`) and
to the packages / applications you upload.

### Software

**Linux** (`run-linux.sh`):
- Docker Engine 24 or newer with the compose plugin (`docker compose version` must work). Install it from
  https://docs.docker.com/engine/install/.
- `curl`. `perl` is only needed for `--domain`.
- root, or a user in the `docker` group.

**Windows** (`run-windows.ps1`):
- **Docker Desktop** (https://www.docker.com/products/docker-desktop/) with the **WSL 2** backend. It must be **running**
  and set to **Linux containers** (tray icon → *Switch to Linux containers…* if it offers that).
- Windows 10/11 Pro or Enterprise. Docker Desktop is not supported on Windows Server: there, use a Linux VM.
- PowerShell 5.1 or 7, **run as administrator**. `curl.exe` is part of Windows.
- If the ZIP came from the internet, unblock the scripts first:
  `Get-ChildItem -Recurse .\server\scripts | Unblock-File`.

### Network

| Port | Direction | Why |
|---|---|---|
| 443/tcp (or your `WSO_HTTPS_PORT`) | devices, admin workstation, Cloud Connector, browsers → server | API and web console (nginx, TLS) |
| 80/tcp (optional) | → server | only redirects to HTTPS |
| outbound 443 | server → `ghcr.io`, Docker Hub | images (first run and updates) |
| outbound 443 | server → `*.awmdm.com`, `*.uemauth.workspaceone.com` | only with the Workspace ONE UEM REST API |

The devices do **not** need Active Directory access: the **Cloud Connector** (a Windows service on a domain member)
connects outbound to the server and does the domain work (Offline Domain Join, computer accounts).

## 3. Install, step by step

### Linux
The server lives in one fixed folder (`/opt/wsodeployer`, `C:\WSODeployer`); every update is copied into it.
```bash
unzip WSODeployer-server-X.Y.Z.zip -d /tmp/wso
sudo mkdir -p /opt/wsodeployer && sudo cp -r /tmp/wso/WSODeployer-server-X.Y.Z/. /opt/wsodeployer/
cd /opt/wsodeployer
sudo ./server/scripts/run-linux.sh --host=wso.corp.example.com
```

### Windows (PowerShell as administrator)
```powershell
Expand-Archive .\WSODeployer-server-X.Y.Z.zip $env:TEMP\wso -Force
New-Item -ItemType Directory C:\WSODeployer -Force | Out-Null
Copy-Item $env:TEMP\wso\WSODeployer-server-X.Y.Z\* C:\WSODeployer -Recurse -Force
cd C:\WSODeployer
Get-ChildItem -Recurse .\server\scripts | Unblock-File
.\server\scripts\run-windows.ps1 -PublicHost wso.corp.example.com
```

### What the script asks
1. **Server address** (`--host` / `-PublicHost`): the DNS name the devices and browsers use. Add it to your DNS first.
2. **Ports**: 443 / 80 when they are free, otherwise it offers the next free ones.
3. **Deployment certificate** `deploy.cer`: press **Enter** if you have none yet. You create it later on the admin
   workstation and give it to the server with `--deployment-cert=` / `-DeploymentCertificate`.
4. **TLS certificate**: the path to a **PFX** of your company / CA (then its password), or **Enter** for a self-signed
   certificate.
5. A summary. Confirm it, and the script downloads the images and starts the server.

Everything you answer is kept in `.env`. Running the script again (any time, e.g. after an update) asks nothing and never
overwrites keys or passwords.

### At the end
The script waits until `https://<server>/api/health` answers, then prints:
- the console address: `https://<server>/console`;
- **the user `admin` and its password**. The password is also in `server/secrets/local-admin.txt`, readable by you only.
  Sign in with it. This local administrator always exists, also when you later sign in through Workspace ONE Access or
  Entra ID. A new password: `run-linux.sh local-admin` / `run-windows.ps1 -Action local-admin`.

## 4. TLS certificate

| Option | How | Then |
|---|---|---|
| **Self-signed** (default) | Enter at the question; nginx creates it for `WSO_PUBLIC_HOST` (+ `WSO_EXTRA_SAN`) | give `server/tls/server.cer` to the admin workstations, the Cloud Connector and the boot media (Trusted Root) – the scripts there ask for it |
| **PFX** of your company / a public CA | `--pfx=/path/wso.pfx` / `-TlsPfx C:\path\wso.pfx`, or the path at the question | with a public CA nothing to distribute; with an internal CA its root must be trusted (it usually already is in the domain) |

Change it later without touching anything else (the old certificate goes to `tls/old-<time>`):
```bash
sudo ./server/scripts/run-linux.sh tls --pfx=/root/wso.pfx      # or: tls --self-signed
```
```powershell
.\server\scripts\run-windows.ps1 -Action tls -TlsPfx C:\certs\wso.pfx   # or: -Action tls -SelfSigned
```
The PFX is read inside the nginx container (openssl), with its password only in the environment of that one process. Its
certificate and chain become `server.crt`, its key `server.key`, and a DER copy `server.cer`. The script checks that the
key matches the certificate. PFX files exported from Windows (3DES) work.

## 5. Keys and secrets (made by the script on the first run)

| What | Where | Used for |
|---|---|---|
| Database password | `.env` `WSO_DB_PASSWORD` | PostgreSQL |
| Admin API key | `secrets/admin.key` (only its SHA-256 in `.env`) | the admin workstation (`Install-WsoWorkstation.ps1 -AdminKeyFile`) |
| Agent key | `secrets/agent-1.key` (hash in `.env`) | the domain agent (older setups; the Cloud Connector gets its code in the console) |
| Local console administrator | `secrets/local-admin.txt` (PBKDF2 hash in `.env`) | the first sign-in |
| Secret vault + device CA | `data/ca/` (created by the API on its first start) | encrypts the staging account, packages, local admin passwords; signs the device certificates |
| Data protection keys | Docker volume `keys` | encrypts the console settings in the database (UEM API secret, SAML certificate, OIDC secret) and the sign-in cookies |
| TLS certificate | `tls/` | HTTPS |

There is no separate master key (KEK) to type in or keep apart: **the secret vault in `data/ca` and the data protection
keys in the `keys` volume are the keys**. A database backup without them is useless, because its encrypted secrets
cannot be read. Keep them like keys: the `backup` action below saves them, and whoever holds the backup can read the
secrets.

## 6. Folders and volumes

All folders are next to `docker-compose.yml` (`server/`). Paths change with `WSO_DATA` / `WSO_TLS` in `.env`.

| Folder / volume | Container | Contents | Size | Backup |
|---|---|---|---|---|
| `data/content` | api `/data/content` | `deploy.json`, `assets.csv`, Hub, apps, encrypted secrets (`secrets/*.cms`) | Hub ~300 MB + your apps | **yes** |
| `data/ca` | api `/data/ca` | secret vault + device CA | KB | **yes – critical** |
| `data/cert` | api (read-only) | `deploy.cer` (public part of the deployment certificate) | KB | yes |
| `data/os` | api `/data/os` | LAN copies of Windows images | ~6 GB per image and language | no (made again from Microsoft) |
| `data/packages` | api `/data/packages` | provisioning packages uploaded in the console | your packages | yes |
| `data/downloads` | api (read-only) | connector MSI, technician app (console → Downloads) | ~100 MB | no (in the release ZIP) |
| `tls/` | nginx `/etc/nginx/certs` | `server.crt`, `server.key`, `server.cer` | KB | yes |
| volume `pgdata` | db | the database: devices, deployments, profiles, audit | tens of MB | **yes** (`pg_dump`) |
| volume `keys` | api `/keys` | data protection keys | KB | **yes – critical** |
| `secrets/`, `.env` | – (host only) | keys and passwords of section 5 | KB | **yes** |

## 7. Backup and restore

```bash
sudo ./server/scripts/run-linux.sh backup
```
```powershell
.\server\scripts\run-windows.ps1 -Action backup
```
This writes `backups/wso-<time>.dump` (the database) and `backups/wso-<time>-keys/` (`data/ca`, `data/cert`, `.env`,
`secrets`, `tls`, data protection keys). Copy both off the host. `data/content` and `data/packages` are plain files:
add them to your file backup.

Restore on a new host:
1. Unpack the same release. Copy `.env`, `secrets/`, `tls/` and `data/` back (including `ca` and `cert` from the keys
   backup).
2. Run the script once: it creates the containers and keeps all values.
3. Put the data protection keys back into the `keys` volume, and the database back with `pg_restore` (in `server/`):
   ```bash
   docker compose -p wsodeployer cp backups/wso-<time>-keys/dataprotection-keys/. api:/keys/
   docker compose -p wsodeployer exec -T db pg_restore -U wso -d wso --clean < backups/wso-<time>.dump
   docker compose -p wsodeployer restart api
   ```

## 8. Active Directory

The server does not join the domain. The domain work is done by the **Cloud Connector**: console → **Cloud Connectors**
→ **Connector configurator** (service account, domain, OUs, the MSI on a domain member). See [connector.md](connector.md).
Optional shortcut on the first run: `--domain=corp.example.com --ou="OU=Workstations,DC=corp,DC=example,DC=com"`
(`-Domain` / `-DefaultOU`) writes the domain and the default computer OU into the server's `deploy.json`.

## 9. Update

Download the new `WSODeployer-server-X.Y.Z.zip` and copy its content into the same folder as in section 3. It never
contains `.env`, `secrets/`, `tls/` or your data; only `server/data/downloads` and the scripts are replaced. Then run the
script again. It pulls the new images and restarts the containers. The database is
migrated by the API on its start. Take a backup first.

## 10. Next

Sign in as `admin`, then continue with [installation.md](installation.md): admin workstation, deployment certificate,
content, Cloud Connector, boot media, first device.
