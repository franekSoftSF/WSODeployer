# Technician app (WSODeployer.Technician.exe)

A window over the technician scripts of `client\`. It has no deployment logic of its own: `Install-WsoWorkstation.ps1`
and `New-WsoBootMedia.ps1` run underneath (Windows PowerShell 5.1) and their output is shown live.

## Getting it
Web console → **Downloads** → *WSODeployer.Technician.zip*. Unpack the **whole** ZIP (the program looks for
`client\New-WsoBootMedia.ps1` next to itself) and start `WSODeployer.Technician.exe`; it asks for administrator rights
(WinPE, ADK and DISM need them). English / Polish follow Windows, the button in the header switches.

## Tab 1 – Workstation
| Check | Missing → |
|---|---|
| Administrator rights | start the program again (it asks for them) |
| Windows PowerShell 5.1 | part of Windows |
| Windows ADK + WinPE add-on | **Install / repair** (about 4 GB once) |
| OSD module (OSDCloud) | **Install / repair** |

## Tab 2 – Boot media
1. **Join code** from the console (page *Device enrollment*):
   - *Token for media* → media **with a token**: devices enroll on their own (a new device still waits for approval);
   - only the **server address** (`https://…`) or the universal command of that page → **universal media**: the
     technician types a pairing code in WinPE (no secret on the media).
2. **TLS certificate of the server** – *Get it from the server*: when the server certificate is not publicly trusted
   (self-signed / own CA) the program saves the root of its chain to `work\cert\server.cer` for the WinPE trust store and
   shows its SHA-1 - compare it with the server (`server\tls\server.cer`) before you build.
3. **Options**: VMware drivers + VMware Tools (default - display resolution in WinPE, vmxnet3, pvscsi; the tools for
   devices whose deploy.json has none), VirtIO for Proxmox / KVM, USB drive (OSDCloud asks for the disk - all data on it
   is erased), language of the WinPE wizard, ISO path (default `work\media\WSODeployer.iso`).
4. **Build the media** – the first time several minutes; WinPE is then reused while its drivers do not change (only the WSODeployer folder and the ISO are made again), *Build WinPE again* forces it; *Stop* ends the script, *Open the folder* shows the ISO.

## Logs for analysis
**Save logs for analysis** (above the log) packs the window log, `%TEMP%\WSODeployer.log`, the OSDCloud workspace logs
(`C:\OSDCloud\WSODeployer\Logs`) and a summary of the workstation into one ZIP in `work\logs`. Join codes are masked
(`WSOD1.***`); certificates and keys are never collected - the ZIP can go to support.

## For developers
`technician\WSODeployer.Technician` (.NET 10 WPF), tests in `technician\WSODeployer.Technician.Tests`
(`tests\Invoke-Tests.ps1` runs them). `client\Build-WsoDownloads.ps1` publishes it self-contained as one file (about
70 MB) into the technician ZIP.
