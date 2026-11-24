# Windows 11 feature update (25H2 / 26H2) + latest CU (Workspace ONE UEM)

`tools/Invoke-Win11FeatureUpgrade.ps1` upgrades every managed device to a Windows 11 version (`-TargetVersion 25H2`,
default, or `26H2`) and then installs the latest cumulative update (CU). It is independent of the WSODeployer engine
(no module, no server).

## How it works

The first run (WS1 script, SYSTEM) copies the script to `%ProgramData%\WSODeployer\WinUpgrade`, saves the parameters
to `config.json`, registers the scheduled task **WSODeployer Windows Upgrade** (SYSTEM, at startup + every 4 hours) and
starts it - it returns within seconds. The task continues across restarts (T = target version):

| Device | Step |
|---|---|
| base build of T, UBR below the prerequisite | CU from Windows Update (prerequisite of the enablement package) → restart |
| base build of T, prerequisite met | "Windows 11, version T" (enablement package, one restart) from Windows Update |
| no offer, or older (23H2 / 22H2 / Windows 10) | full in-place upgrade: ISO of T downloaded to the device (`WSO_UPG_MEDIAURI`), `-MediaPath` (ISO / folder, local or UNC) or the Windows 11 Installation Assistant (only when T is the newest release) |
| build of T | CU from Windows Update → restart → rescan until nothing is offered (and UBR ≥ `-MinimumUbr`) → **Done**, task removed |
| newer than T | **Done** - never a downgrade |

| T | Build | Enablement package | Base builds and prerequisite |
|---|---|---|---|
| 25H2 | 26200 | [KB5054156](https://support.microsoft.com/help/5054156) | 24H2 ≥ 26100.5074 |
| 26H2 | 26300 | [KB5121794](https://support.microsoft.com/help/5121794) | 24H2 ≥ 26100.9546, 25H2 ≥ 26200.9546 (KB5124010) |

The enablement packages are not in the Microsoft Update Catalog - only Windows Update / WSUS (classification
Upgrades) offer them. The **Installation Assistant always installs the newest release** (today 26H2), so with
`-TargetVersion 25H2` a full upgrade needs `-MediaPath` with 25H2 media - the script refuses to overshoot.
Every phase has an attempt limit (`-MaxAttempts`, default 3); then the state is **Failed** and the task is removed.

### ISO on the device (full upgrade)

With `WSO_UPG_MEDIAURI` the device downloads the ISO itself - only when a full upgrade is really needed (24H2 devices
with the enablement package never download it):

- **Where**: `C:\Windows\Temp\WsoWinUpgrade\Win11_<T>.iso` (`WSO_UPG_MEDIACACHE`); the folder is hidden + system and
  only SYSTEM and Administrators can open it. The script deletes only its own files (`Win11_*.iso`, the partial
  download), after **Done** or when the target changes; after **Failed** the ISO stays for a retry.
- **How**: BITS - continues after a restart / the next task run (every 4 h), only on unmetered networks (no LTE /
  hotspot), at most 5 h per task run. Without BITS: a plain HTTPS download. Needs 35 GB free on C:.
- **Check**: `WSO_UPG_MEDIASHA256` (recommended) - after the download and before every reuse; a mismatch deletes the
  file. Then setup.exe (Microsoft signature) and the image build (must be T) are checked too.
- **Source**: an HTTPS address you host - the Microsoft ISO links expire after 24 h. Off-network laptops need an
  internet-reachable address: an own web server, or blob storage with a read-only SAS token (Azure Blob / S3). The
  token is only in the WS1 Variable and in `config.json` (folder only for SYSTEM / Administrators); the log shows the
  address without the query string. SHA-256 of the ISO: `Get-FileHash .\Win11_25H2_Polish_x64.iso`.
- Use the ISO in the devices' language (and edition: a multi-edition ISO, Pro / Enterprise) - setup keeps apps and
  data only with the same language.

### Moving to the next version (25H2 → 26H2)

1. WS1 Windows Updates profile: `TargetReleaseVersion` = `26H2` (otherwise the device is never offered 26H2).
2. WS1 script Variable `WSO_UPG_TARGETVERSION` = `26H2`, run the script again: a changed target resets the state
   (also after Done).
3. 25H2 devices take the 26H2 enablement package (one restart), 24H2 devices too (no 25H2 in between).

A new release later: add a row to `$script:Releases` in the script (build, KB, base builds with the minimum UBR from
the KB article), the value to `-TargetVersion`'s `ValidateSet`, and move `Latest`.

## The WS1 "Windows Updates" profile

The profile (Policy CSP `Update/*`) decides what Windows Update offers. The script never overrides it:

- **Target version**: *Windows 11* / T in the profile. When the profile pins another version the script logs a warning
  and skips the enablement package (full upgrade instead). Without a target version in the profile the script sets
  the registry policy `TargetReleaseVersion` = T itself (`-NoPolicy` turns that off).
- **Feature update deferral / pause**: 0 days, not paused - otherwise T is not offered.
- **Quality update deferral / pause**: the CU is offered only after the deferral; for "the newest CU now" use 0 days
  (or a short deferral and `-MinimumUbr`).
- The scan goes to Windows Update online (`-UpdateSource WindowsUpdate`, default) even on a WSUS-configured device;
  `-UpdateSource Default` uses the configured source.
- Safeguard holds: Microsoft may block a release on some hardware; then the full upgrade runs (setup.exe
  `/compat ignorewarning` ignores only dismissible warnings - hard blocks still stop it, see `setup-logs`).

### Recommended profile settings

Policy CSP `./Device/Vendor/MSFT/Policy/Config/Update/...` ([reference](https://learn.microsoft.com/en-us/windows/client-management/mdm/policy-csp-update)):

| Setting | Value | Why |
|---|---|---|
| `ProductVersion` / `TargetReleaseVersion` | `Windows 11` / T | T offered, nothing newer; the script then writes no registry policy |
| `BranchReadinessLevel` | 16 | General Availability Channel |
| `ManagePreviewBuilds` | 0 | no Insider builds |
| `DeferFeatureUpdatesPeriodInDays`, `DeferQualityUpdatesPeriodInDays`, `Pause*` | not set (0) | T and the newest CU offered at once |
| `ConfigureDeadlineForFeatureUpdates` | 5 (rollout: 2-5) | days until forced install |
| `ConfigureDeadlineForQualityUpdates` | 2-5 | |
| `ConfigureDeadlineGracePeriod` | 1-2 | days from install to forced restart (quality; also feature when `ConfigureDeadlineGracePeriodForFeatureUpdates` is not set) |
| `ConfigureDeadlineNoAutoReboot` | **0** | 1 = no automatic restart outside active hours before deadline + grace (up to 7 days waiting for a restart) |
| `SetDisablePauseUXAccess` | **1** | users cannot pause updates in Settings |
| `AllowAutoWindowsUpdateDownloadOverMeteredNetwork` | **0** | no CU / upgrade downloads over LTE / hotspots |
| `IgnoreMOUpdateDownloadLimit`, `IgnoreMOAppDownloadLimit` | not set | legacy, not applicable to Windows 11 |
| `AllowAutoUpdate` | 1 or not set | ignored for download / install / restart when deadlines are configured |
| `DisableWUfBSafeguards` | not set (0) | keep safeguard holds; the script uses the full upgrade where they block |

Outside the Update profile: Delivery Optimization (`./Device/Vendor/MSFT/Policy/Config/DeliveryOptimization/DODownloadMode`
= 1 LAN or 2 group) so a CU is downloaded from the internet once per site.

Custom Settings snippet for the two changes most profiles miss:

```xml
<Replace><CmdID>1</CmdID><Item><Target><LocURI>./Device/Vendor/MSFT/Policy/Config/Update/ConfigureDeadlineNoAutoReboot</LocURI></Target><Meta><Format xmlns="syncml:metinf">int</Format></Meta><Data>0</Data></Item></Replace>
<Replace><CmdID>2</CmdID><Item><Target><LocURI>./Device/Vendor/MSFT/Policy/Config/Update/SetDisablePauseUXAccess</LocURI></Target><Meta><Format xmlns="syncml:metinf">int</Format></Meta><Data>1</Data></Item></Replace>
```

## Deployment in WS1 UEM

1. **Resources → Scripts → Add → Windows**: language PowerShell, execution context **System**, timeout 300 s,
   script = the content of `tools/Invoke-Win11FeatureUpgrade.ps1` (unchanged, no parameters).
2. **Variables** of the script (WS1 passes no parameters - it sets each Variable as an environment variable, the
   script reads `WSO_UPG_<parameter name>`), e.g.:

   | Key | Value |
   |---|---|
   | `WSO_UPG_TARGETVERSION` | `25H2` (later `26H2`) |
   | `WSO_UPG_RESTARTDELAYMINUTES` | `120` |
   | `WSO_UPG_MEDIAURI` | `https://files.example.com/win11/Win11_25H2_x64.iso` (for the full upgrade, see above) |
   | `WSO_UPG_MEDIASHA256` | SHA-256 of that ISO |

   Switches take `true` / `false` (`1` / `0`), e.g. `WSO_UPG_NORESTART` = `true`. Without a Variable the default
   applies. An invalid value fails the script (exit 1 = *Failed* in WS1, details in `upgrade.log`).
   Moving to 26H2 = change `WSO_UPG_TARGETVERSION` and run the script again.
3. Assign it to a smart group (pilot first), trigger *On demand* or *Scheduled*.
4. Optional sensor (PowerShell, System, String) for reporting:

   ```powershell
   $p = Get-ItemProperty 'HKLM:\SOFTWARE\WSODeployer\WinUpgrade' -ErrorAction SilentlyContinue
   if ($p) { '{0} {1} {2}.{3} {4}' -f $p.TargetVersion, $p.State, $p.Build, $p.Ubr, $p.Detail } else { 'NotStarted' }
   ```

## Settings

WS1 Variable (environment variable) or, when run by hand, the parameter of the same name (the parameter wins).

| WS1 Variable | Parameter | Default | |
|---|---|---|---|
| `WSO_UPG_TARGETVERSION` | `-TargetVersion` | `25H2` | `25H2` or `26H2` |
| `WSO_UPG_MEDIAURI` | `-MediaUri` | – | HTTPS address of the ISO of T, downloaded into the media cache (wins over MediaPath) |
| `WSO_UPG_MEDIASHA256` | `-MediaSha256` | – | SHA-256 of the ISO (64 hex characters) |
| `WSO_UPG_MEDIACACHE` | `-MediaCache` | `C:\Windows\Temp\WsoWinUpgrade` | hidden local folder for the ISO |
| `WSO_UPG_MEDIAPATH` | `-MediaPath` | – | `.iso` or folder with `setup.exe` of T, local or UNC (a UNC ISO is copied into the media cache) |
| `WSO_UPG_NOINSTALLATIONASSISTANT` | `-NoInstallationAssistant` | false | never download the Installation Assistant |
| `WSO_UPG_QUALITYSCOPE` | `-QualityScope` | `Cumulative` | `All` = every non-preview software update (never drivers) |
| `WSO_UPG_UPDATESOURCE` | `-UpdateSource` | `WindowsUpdate` | `Default` = WSUS / configured source |
| `WSO_UPG_RESTARTDELAYMINUTES` | `-RestartDelayMinutes` | 60 | announced restart when a user is signed in; without a user: at once |
| `WSO_UPG_NORESTART` | `-NoRestart` | false | never restart - continue after the next restart |
| `WSO_UPG_RESTARTMESSAGE` | `-RestartMessage` | English text | message shown with the restart |
| `WSO_UPG_MINIMUMUBR` | `-MinimumUbr` | 0 | Done only from <build of T>.<UBR> |
| `WSO_UPG_MAXATTEMPTS` | `-MaxAttempts` | 3 | per phase |
| `WSO_UPG_NOPOLICY` | `-NoPolicy` | false | never write the registry TargetReleaseVersion policy |
| `WSO_UPG_RESET` | `-Reset` | false | clear the state (also after Done / Failed) - remove the Variable afterwards |

Status: `HKLM\SOFTWARE\WSODeployer\WinUpgrade` (`TargetVersion`, `State` = Scheduled / Running / WaitingForRestart /
Done / Failed, `Detail`, `Build`, `Ubr`, `LastRun`); log `%ProgramData%\WSODeployer\WinUpgrade\upgrade.log`.

## Tested

Static checks only (parser, PSScriptAnalyzer, update filters on sample data). Not tested on a device: Windows Update
scan / install, the enablement packages, setup.exe and the Installation Assistant. Test on a VM / pilot group first.
