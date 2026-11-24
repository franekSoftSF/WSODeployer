# WSODeployer installation – step by step (server mode)

Polski: [pl/installation.md](pl/installation.md)

Order: **server → admin workstation → certificate to the server → content → domain agent → boot media → VM test**.
Run everything from the folder of the unpacked release package (below); local files go to its `work\` folder.
Every script asks for missing values, shows the plan before changing anything and supports `-NonInteractive` and `-Language en|pl`.

## 0. Downloads (as of 2026-10-02)

### WSODeployer
| What | Where |
|---|---|
| Admin package `WSODeployer-<version>.zip` (scripts of the admin workstation, domain agent and media, device engine, sample content, server start kit, connector MSI and technician app) | **Releases** of the public repository `franekSoftSF/WSODeployer` (`.sha256` next to it) |
| Server images `wsodeployer-api`, `wsodeployer-nginx` | `ghcr.io/franeksoftsf/` - pulled by the server scripts of the package, nothing to build |

Unpack the ZIP on the admin workstation (e.g. `C:\WSODeployer`) and copy its `server\` folder to the Docker host (Linux or
Windows). The `server\data\downloads` folder of the package already holds the connector MSI and the technician app.

### Windows ADK + WinPE add-on (admin workstation)
ADK and the WinPE add-on **must be the same version**. Current: **ADK 10.1.26100.9457 (September 2026)**. It replaces
10.1.26100.2454 and supports Windows 11 26H2, 25H2 and 24H2 and Windows Server 2025/2022.

| File | Link |
|---|---|
| Windows ADK 10.1.26100.9457 | https://go.microsoft.com/fwlink/?linkid=2289980 |
| Windows PE add-on for ADK 10.1.26100.9457 | https://go.microsoft.com/fwlink/?linkid=2289981 |
| Microsoft page (versions) | https://learn.microsoft.com/windows-hardware/get-started/adk-install |
| ADK patches (msp) | https://learn.microsoft.com/windows-hardware/get-started/adk-servicing |

- ADK setup: **Deployment Tools** is enough. WinPE add-on: install everything.
- **Do not use ADK 10.1.28000.1**. It is for Windows 11 26H1 Arm64 only, although winget shows it as the "latest".
- With winget (what `Install-WsoWorkstation.ps1` does) you get 10.1.26100.2454 for both kits. That is the same release line: apply
  **KB5101684** (https://aka.ms/Windows_ADK_10.1.26100.2454_Update_KB5101684.zip). It contains DISM / WSIM security fixes and
  oscdimg for Secure Boot SVN 9.0. To install it, unzip, then in an elevated CMD:
  `for %i in (*.msp) do start /wait msiexec.exe /qn /p "%~fi"`
- `Install-WsoWorkstation.ps1` accepts any 10.1.26100.x. If you install 9457 manually first, the script keeps it.

### Windows 11 25H2 (OS for the devices)
**No ISO needed.** OSDCloud in WinPE downloads the ESD from Microsoft (`dl.delivery.mp.microsoft.com`). OSD 26.10.6.1 lists
Windows 11 25H2 (build 26200), pl-pl included. Set it in the `deploy.json` profile:
```json
"OSName": "Windows 11 25H2 x64",
"OSEdition": "Enterprise",
"OSLanguage": "pl-pl",
"OSActivation": "Volume"
```
The sample content uses 25H2 in both profiles.

You only need a 25H2 ISO for a manual reference VM. The public Microsoft pages (https://www.microsoft.com/software-download/windows11,
Evaluation Center) already offer **26H2**. Get 25H2 from volume licensing (Microsoft 365 admin center → Volume licensing) or a
Visual Studio subscription.

### Workspace ONE and AD
| What | Where |
|---|---|
| Intelligent Hub `AirwatchAgent.msi` (x64) | **downloaded by `Initialize-WsoContent.ps1`** from https://packages.omnissa.com/wsone/AirwatchAgent.msi (the getwsone.com file) into `workdownloads`, Omnissa signature checked; own file: `-HubFile` |
| Device Services host (e.g. `ds1234.awmdm.com`) | UEM → Groups & Settings → All Settings → System → Advanced → Site URLs |
| Group ID | UEM, OG name |
| Staging account (user + password) | UEM → Accounts → Users, staging enrollment in that OG only; Advanced → Staging: **Enable Device Staging = On** (off by default) and **Single User Devices = Standard** (the user signs in after staging; *Multi User Devices* there is for mobile platforms, not the Windows multi-user mode) - otherwise domain / Entra devices stay on the staging account (Hub log: *staging mode is not SingleUnknownUserStaging*) |
| Domain (FQDN) and computer OU | AD |
| Device list: serial number, inventory number (optional profile, OU) | `assets.csv` (format as `content\assets.csv`) |

### From the WSODeployer server (`server\`, DEV instance)
- `secrets-dev\admin.key` – for the admin workstation;
- `secrets-dev\agent-1.key` – for the domain agent;
- `tls-dev\server.cer` – the TLS certificate of the server (workstation, agent, media);
- `secrets-dev\local-admin.txt` – user `admin` and its password for the web console (also printed at the end of the script).

## 1. Server
`server\scripts\run-windows.ps1 [-Dev]` or `run-linux.sh [--dev]`, see [server/README.md](server.md). From the release
package the scripts pull the released images (`docker compose pull`); in a clone of the repository they build them. The script always creates
the console administrator `admin` and prints its password at the end. Sign in with it at `https://<server>/console` and
set up the console sign-in (Workspace ONE Access SAML / Entra ID) on the page **Sign-in** - no `.env` editing.

Workspace ONE UEM REST API (optional, v0.24): console page **Workspace ONE UEM** - an OAuth client per environment, then
import the UEM computers as devices; after each deployment the server sets the UEM AssetNumber and the Workspace ONE tag;
**Groups from UEM** creates device groups from the UEM organization groups (server/README.md).

## 2. Admin workstation (elevated PowerShell)
```powershell
.\client\Install-WsoWorkstation.ps1 -ServerUri https://<server>:8443/ -ServerCertificate .\server\tls-dev\server.cer -AdminKeyFile .\server\secrets-dev\admin.key -CertificateDir .\work\cert
```
- Installs or detects ADK and the WinPE add-on, the OSD module, the server certificate (Trusted Root) and the protected admin key.
- Asks for the **PIN of `deploy.pfx`** (8+ characters). Technicians type it in WinPE.
- Creates `.\work\cert\deploy.cer` / `deploy.pfx`. Settings go to `%APPDATA%\WSODeployer\workstation.json`.

## 3. Deployment certificate to the server
```powershell
.\server\scripts\run-windows.ps1 -Dev -DeploymentCertificate .\work\cert\deploy.cer
```
Without `data-dev\cert\deploy.cer`, devices cannot sign their requests to the server.

## 4. Deployment content
**Workspace ONE from the console** (simplest): Console → **Workspace ONE** - device services server, Group ID, ownership,
the **staging account** (write-only, encrypted on the server at once to the deployment certificate and the vault) and the
**Intelligent Hub** (download the current one from Omnissa or upload an MSI). The wizard below is then only needed for
apps and `assets.csv`.

**Several Workspace ONE environments** (v0.22, e.g. test and production tenants): the settings above are the default
environment; *Further Workspace ONE environments* adds more, each with its own server, Group ID, ownership and staging
account (`secrets/staging-<name>.cms`), stored in deploy.json `Ws1Environments`. A profile picks its environment
(*Workspace ONE environment*, empty = the default one); the Hub is shared. An environment used by a profile cannot be
deleted. The device checks the environment of its profile before the disk is wiped.

```powershell
.\client\Initialize-WsoContent.ps1 -ContentRoot .\work\content
```
The content goes to the server through its API (admin key of `Install-WsoWorkstation.ps1`): only changed files
(`deploy.json`, `assets.csv`, `hub\*.msi`, `apps\*`, encrypted `secrets\*.cms`), `deploy.json` last. The server checks the
SHA-256, takes secrets only as an encrypted CMS message and keeps the previous `deploy.json` as `deploy.json.bak-*`.
Upload only (e.g. after a change in `apps\`): `.\client\WSODeployer.ps1 -Action ServerPublish -ContentRoot .\work\content`.
With `-PublishTo <folder>` the wizard copies the content into a folder instead (server on the same computer, e.g.
`.\server\data-dev\content`).
The wizard asks for:
- organization and name prefix;
- domain join method (`ODJ`), domain and OU;
- WS1 server, Group ID and CD/CS ownership;
- the Hub: download the current x64 Hub (`y`, about 300 MB, reused while current) or `-HubFile <msi>`;
- the staging account (answer `y`).

It then hashes the files, tests the content, uploads it to the server and imports `assets.csv`.

Before you run it:
- put your devices into `work\content\assets.csv` (the folder is created on the first run - you can stop it and run it again);
- add `apps\VC_redist.x64.exe` or remove the sample `VCRedist` app from `deploy.json`;
- edit profiles (OS, language, time zone, apps) in `deploy.json` → `Profiles`.

## 5. Domain agent (domain-joined Windows server, elevated)
Copy these to the server:
- the `agent\` and `engine\` folders of the repository;
- `deploy.cer`;
- `server.cer`;
- `agent-1.key`.

Then run:
```powershell
.\agent\Install-WsoAgent.ps1 -ServerUri https://<server>:8443/ -AgentKeyFile C:\temp\agent-1.key -DeploymentCertificate C:\temp\deploy.cer -ServerCertificate C:\temp\server.cer -DelegateOU "OU=Workstations,OU=Computers,DC=corp,DC=example,DC=com"
.\agent\Install-WsoAgent.ps1 -Action test
```
- By default the task runs as SYSTEM, which is the server's computer account; that account gets the OU rights.
  `-GmsaName` uses a gMSA instead (needs a KDS root key).
- The server needs access to port 8443 of the WSODeployer server.
- Delete `C:\temp\agent-1.key` afterwards; the agent keeps its own protected copy.

## 6. Boot media (elevated)
```powershell
.\client\New-WsoBootMedia.ps1 -OutFile .\work\media\WSODeployer.iso
```
Creates `work\media\WSODeployer.iso`. With `-Usb` it also writes a USB drive: OSDCloud asks for the disk and **erases it**.

## 6b. The wizard on the device
WinPE starts the **graphical wizard** (as in MDT): welcome + PIN → device (identification / approval) → summary (**Install**)
→ progress with a live log → finish (restart in 15 s). Nothing is changed before **Install**. Console instead of the window:
`D:\WSODeployer\Start-WsoDeploy.ps1 -NoGui` (or `"Gui": false` in `media.json`). The black console window behind the wizard
shows the same output - leave it open.

## 6c. Profiles (web console)
What gets installed is set per profile in the console: **Profiles** → New / edit. A profile covers:
- Windows version, edition, language and activation;
- applications;
- OU, WS1 group and ownership;
- time zone, keyboard and locales;
- driver pack and Windows Update.

The device record (approval / assets.csv) selects the profile; empty = the default profile. On its first start the
server imports the profiles of deploy.json; afterwards the console is the source.

**New profile = a template** (v0.21). *Profiles → New profile* offers the scenarios Active Directory, Hybrid, Microsoft
Entra (user in OOBE / bulk package) and Workgroup. A scenario starts from the default profile, sets the identity and the
OOBE, the region of the Windows language, and shows what the server still lacks for it (domain in deploy.json, connector
online and its rights in the OU, Hybrid join SCP, a valid Entra package, Workspace ONE ready) with a link to set it up.
*Save as template* on a profile keeps it as a company template - without the administrator password and the VPN key.
The fields OU, WS1 Group ID, VPN check and administrator name suggest the values in use (the OU list says whether the
connector may create computer accounts there).
## 6d. LAN copy of the Windows image
Profiles with **Image source** Auto (default) take Windows from the WSODeployer server once it has a copy: the first device
asks the server for it (and installs from Microsoft), the next ones download it from the LAN. In advance, on the admin
workstation: `.\client\WSODeployer.ps1 -Action ServerImages`. Console → **Images** shows progress and the SHA-256. The
server needs about 6 GB per image and language in `server\data\os`.

## 6e. Virtual machines (VMware, Proxmox)
The device detects its platform and installs the platform applications of deploy.json "Platforms" by itself:
VMware Tools on VMware, the VirtIO guest tools + QEMU guest agent on Proxmox / KVM. `Initialize-WsoContent.ps1` downloads
them (signature / https checks). For **Proxmox** build the media with `-VirtIO`: without the VirtIO drivers WinPE does not
see a VirtIO disk, and the installed Windows would not start from it.

## 6f. Packages, Workspace ONE Drop Ship, local administrator
Console → **Packages**: upload a .ppkg. For the Workspace ONE Drop Ship package (Configuration Tool for Provisioning) also
upload its unattend.xml - it is encrypted right away. In the profile, select the packages. **Workspace ONE enrollment** =
Drop Ship package enrolls at the first logon; the computer name and the domain join still come from WSODeployer.
**Local administrator**: enable it and type the password once; it is stored encrypted and cannot be shown again.

**Windows LAPS with an own account per customer (v0.19)**: in the profile give the **account name** (e.g. `lapsadmin`,
up to 20 characters) and tick **random password**. SetupComplete creates the account (or resets it), puts it into the
Administrators group (by SID, any Windows language) and sets a random password that is never stored or logged. The
LAPS policy of Workspace ONE / Intune (CSP `./Device/Vendor/MSFT/LAPS/Policies`) must use the same name in
`AdministratorAccountName`; LAPS then rotates the password and backs it up in Entra ID or AD. Empty name = the built-in
Administrator (SID ...-500). Until the LAPS policy has run nobody knows the password - plan the WS1 profile for the
same smart group as the devices.

## 6g. Device groups
Console → **Device groups**: a group gives its devices a profile, OU and Workspace ONE group (device > group > profile).
Rules (manufacturer, model, IP subnet) put new devices into the group while they wait for approval. On **Devices**,
tick devices and move them to a group or set their profile in one step; assets.csv may carry a **Group** column.

## 6h. Device enrollment instead of deploy.pfx (v0.11)
Console → **Device enrollment** → token → copy the **join code** → `.\client\New-WsoBootMedia.ps1 -JoinCode 'WSOD1.…'`.
The media has no deploy.pfx and no PIN; WinPE makes its own key and enrolls (a new device waits in Approvals). Save the
staging account / local admin / WS1 package again so they reach the server's secret vault. Details: docs/enrollment.md.

## 6i. Cloud Connector (v0.12)
Simplest: Console → **Cloud Connectors** → **Connector configurator** - step by step (new connector or a new code for
an existing one, service account: domain user / gMSA / LocalSystem, domain, OUs, optional domain controller). At the end
it registers the connector and shows ready PowerShell for every step (trust the server certificate, gMSA, rights on the
OUs, MSI install - or only the new code on a server with the service - and the check), each with a Copy button.

Console → **Cloud Connectors** → register → copy the key. On a domain member (elevated):
`.\agent\Install-WsoAgent.ps1 -ServerUri <server> -AgentKeyFile <key file> -DelegateOU '<OU DN>'` - installs the Windows
service WSODeployerConnector (ODJ + computer account jobs). Details: docs/connector.md.

## 6j. Device identity: Entra ID / Hybrid (v0.15)
Console → **Profiles** → *Device identity and OOBE*: Workgroup, AD, Hybrid, Entra (the user joins in OOBE) or Entra
automatic. For the automatic mode make a bulk package in Windows Configuration Designer and upload it under **Packages**
as *Entra bulk package (WCD)* with its token expiry, then select it in the profile. Hybrid needs Entra Connect Sync with
Hybrid join (the connector reports the SCP). Lab preparation step by step: docs/identity-design.md.
## 6k. Technician app (v0.16)
Console → **Downloads** → *WSODeployer.Technician.zip* → unpack → **WSODeployer.Technician.exe** (asks for administrator
rights). Tab 1 checks / installs ADK, WinPE and OSD; tab 2 builds the media from the join code of the console (or only the
server address for pairing codes). Details: docs/technician.md.
## 6l. Current Windows images (v0.17)
Once a month after Patch Tuesday, on the admin workstation (elevated): `.\client\WSODeployer.ps1 -Action ServiceImage` -
the newest cumulative update goes into the Windows image of every profile and the serviced WIM to the server; devices
install it instead of the Microsoft ESD. Details: docs/images.md.
## 6m. Domain controller at the first logon (v0.20)
Supported: the device is in the LAN or **Workspace ONE Tunnel** connects it - the profile default, nothing to set. Only for a
lab or a site without WS1 Tunnel: profile option *OpenVPN before the logon* (OpenVPN MSI in "Apps", one .ovpn with the
certificates inline, stored encrypted). Details: docs/prelogon-vpn.md.
## 7. First test – Windows 11 25H2 VM
- VM: UEFI + Secure Boot, vTPM 2.0, 4–8 GB RAM, 64 GB+ disk, network with internet access and access to the WSODeployer server.
- On ESXi, an **E1000e** NIC is the simplest choice. vmxnet3 works only when the media was built with a `-CloudDriver` that includes VMware (the default `*` does).
- Put the VM serial number (`Get-CimInstance Win32_BIOS`) or MAC into `assets.csv`. Otherwise the VM waits for approval in the console.

Flow:
1. Boot from the ISO, then enter the PIN of `deploy.pfx`.
2. Identification on the server, or approval in the console (`/console` → Approvals).
3. The agent creates the computer account, and OSDCloud downloads 25H2 and installs it.
4. Specialize (ODJ), then SetupComplete (Hub, WS1 enrollment).
5. Sign in with a domain account.

Follow the run in the web console under **Deployments**. Device log: `C:\ProgramData\WSODeployer\`.
