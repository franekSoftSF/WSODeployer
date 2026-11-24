# Windows images: LAN copies and serviced images

## Why
OSDCloud installs the Windows ESD of Microsoft's catalog. Microsoft refreshes those ESDs only a few times a year: in
October 2026 the newest 25H2 ESD is still build 26200.8653 of June, so a fresh device first spends a long time in OOBE
("Getting the newest updates"). WSODeployer keeps two kinds of images on the server:

| Kind | Made by | Content |
|---|---|---|
| **esd** (LAN copy, v0.7) | the server, downloaded once from Microsoft | the catalog ESD as it is |
| **serviced** (v0.17) | `ServiceImage` on the admin workstation | ESD + the newest cumulative update, only the editions the profiles use |

A device takes the newest **serviced** image of its OS name, language, activation and edition; without one, the LAN
copy of the ESD (profile image source Auto / LAN) or Microsoft.

## ServiceImage - once a month
On the admin workstation (Install-WsoWorkstation.ps1 done, elevated PowerShell, about 25 GB free):
```powershell
.\client\WSODeployer.ps1 -Action ServiceImage                       # every profile OS of the server
.\client\WSODeployer.ps1 -Action ServiceImage -OSName 'Windows 11 25H2 x64' -Force
```
For every OS / language / activation of the server's profiles (image source Auto / LAN) with the editions they use:
1. **Microsoft Update Catalog**: the newest released cumulative update ("YYYY-MM Cumulative Update for Windows 11,
   version 25H2 for x64-based Systems (KB…) (build)") - no Preview, no Dynamic updates. Its .msu files (24H2 / 25H2: with
   the checkpoint update) are downloaded once, checked against the SHA-1 in their names and the Microsoft signature.
2. When the server already has that serviced image (same name = OS, language, activation, build, KB, all editions):
   nothing to do (`-Force` builds it again).
3. Base ESD: the server's LAN copy (fast), else Microsoft - cached in `%ProgramData%\WSODeployer\Servicing` (`-OutDir`).
4. DISM: the editions out of the ESD into a WIM, the update into each one, component cleanup, compact export
   (`W11-25H2-x64-pl-pl-Volume-26200.9457-KB5129195.wim`).
5. Upload to the server (admin key, streamed, SHA-256 compared) - the console page **Images** shows it with the badge
   *serviced*, build, KB, editions and its age (red after 35 days: the next monthly update is out).

Run it the day after Patch Tuesday (second Tuesday of the month), e.g. as a scheduled task of the admin account.
Old serviced images can be deleted on the Images page; devices always take the newest one.

## Checks
- Console **Images** → table *Profiles and their LAN copy*: each profile shows the serviced build it gets.
- Device log: `Serviced Windows image from the server: … (build …, KB…)`; `winver` after the installation.
