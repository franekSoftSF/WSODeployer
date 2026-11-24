# Cloud Connector (v0.12 / v0.13)

The Cloud Connector is a Windows service on a domain member. It does the directory work for the WSODeployer server,
which runs in a Linux container and has no Active Directory rights:

- **ODJ blobs**: `djoin /provision /reuse` for a device that is being installed. The blob is encrypted to deploy.cer
  and to the server's secret vault and returned as CMS text - the server never holds a usable blob.
- **Computer account lifecycle** (Devices page → select devices → *Active Directory*):
  - *move the account to its OU* - the OU the device has now (device > group > profile > deploy.json);
  - *reset the computer account* (admins) - like "Reset Account" in ADUC; the device loses its domain trust;
  - *disable / enable the computer account* (admins);
  - retiring a device that has a computer account queues *disable* automatically (`Wso:Connector:DisableOnRetire`).

The computer name, OU and domain of an ODJ request are computed by the server, exactly as the engine does - the
connector has no naming rules of its own.

## High availability

Register several connectors (one per site or two per domain). Each request and job is claimed by one connector
(`FOR UPDATE SKIP LOCKED`); work of a connector that stops answering returns to the queue after 10 minutes.

## Console: Cloud Connectors

- **Register a connector** - a one-time **registration code** (`WSOC1.…`, valid 24 h) is shown once; **New code** for
  a reinstalled or moved service.
- **Download the connector (MSI)** - built on the workstation with `.\client\Build-WsoDownloads.ps1` (puts it into
  `data\downloads` of the server; `-Dev` for the test instance).
- State from the heartbeat (every minute): online / offline, server and **service account**, domain and domain
  controller, version and certificate expiry, last error, done / failed counters, and for every OU the server uses
  (deploy.json, profiles, device groups, devices) **whether the account may create computer accounts** there.
- **Delegation for the service account** - the command of `Grant-WsoConnectorRights.ps1` (page Downloads) with those OUs, or
  the same `dsacls` commands by hand (a domain administrator runs it once).
- Page **Downloads**: the connector MSI, the connector in PowerShell (ZIP), the delegation script and the technician tools
  (ZIP), with version, build time and SHA-256 - built by `.\client\Build-WsoDownloads.ps1`.
- **Disable** refuses the connector at once (the service stops); **Delete** removes it.
- **Computer account jobs** - queued / in progress / done / failed, who requested and which connector did it.
- v0.12 connectors with a static key and v0.2 agents with a key in appsettings (`Wso:AgentKeys`) keep working.

## Installation (MSI)

1. Console → Cloud Connectors → **Register a connector** → copy the registration code; **Download the connector (MSI)**.
2. A domain administrator delegates the rights to the service account (the commands under *Delegation*): create / delete
   computer objects, reset password, account restrictions, validated writes to the DNS host name and SPN. A gMSA works
   too (`New-ADServiceAccount`, allowed to retrieve its password on the connector server).
3. On the domain member: run the MSI, paste the code, give the account (domain + user + password, or a gMSA `name$`
   without a password; an empty user = LocalSystem, the computer account of the server). Silent:

```powershell
msiexec /i WSODeployer.Connector.msi /qn JOINCODE="WSOC1...." SERVICEDOMAIN=CORP SERVICEUSER=svc-wso SERVICEPASSWORD="..."
msiexec /i WSODeployer.Connector.msi /qn JOINCODE="WSOC1...." SERVICEDOMAIN=CORP SERVICEUSER=gmsa-wso$
```

4. The service **WSODeployerConnector** starts, makes its own RSA key (DPAPI LocalMachine, `%ProgramData%\WSODeployer\
   Connector` limited to SYSTEM, Administrators and its account), registers with the code (one use - it is deleted) and
   gets a certificate of the server CA (1 year, renewed 30 days before the end). Its requests are signed with that key,
   so no key of the server is stored and no client certificate (mTLS) is needed behind a proxy.
5. The console shows it online with its account and the OU rights. Log: `C:\ProgramData\WSODeployer\Connector\Logs\connector-yyyyMMdd.log`
   (one file a day, 14 days; the start line names the account the service runs as) and the *Application* event log, source
   *WSODeployer Connector*.

The MSI grants the account "Log on as a service" and access to the data folder and `HKLM\SOFTWARE\WSODeployer\Connector`;
it does **not** change Active Directory. An upgrade (a newer MSI) keeps the identity; a new code makes the service register
again. The server TLS certificate must be trusted by the connector server (public CA or your own CA in Trusted Root).
`agent\Install-WsoAgent.ps1` stays for scripted installs (v0.12 key) and `-Mode task` (v0.2 agent).

**The service account must stay the same.** Active Directory (KB5020276, CVE-2022-38042) lets `djoin /reuse` take over an
existing computer account only for its owner - the account the connector ran as when it created it. A connector switched to
another account (e.g. LocalSystem = `DOMAIN\HOST$`) fails a reinstall with `0xaac` ("Re-using the account was blocked by
security policy"). Either keep the account, or add the connector account(s) to the domain controller policy *Domain
controller: Allow computer account re-use during domain join* (Default Domain Controllers Policy → Security Options).
## Next (planned)

- **Entra ID**: profile join modes Workgroup / AD (ODJ) / Hybrid / Entra. Entra via a Windows Configuration Designer
  bulk-enrollment package (encrypted to the vault, applied offline) and, for customers with Intune, Autopilot
  (hardware hash uploaded by the connector through Microsoft Graph). The connector also cleans up stale Entra device
  records and adds devices to Entra groups.
