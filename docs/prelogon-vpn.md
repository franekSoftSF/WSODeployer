# Domain controller at the first logon – LAN, Workspace ONE Tunnel, or OpenVPN (alternative)

Offline Domain Join makes the device a domain member without a domain controller, but **the first logon with a domain
account needs the DC**. How the device reaches it is set per profile (console: *Profiles → Identity and Workspace ONE →
Domain controller at the first logon*; guided mode: the question after the OU).

| Option | When | What WSODeployer does |
|---|---|---|
| **LAN or Workspace ONE Tunnel** (default, supported) | the device is in the company network, or Workspace ONE Tunnel connects it (WS1 profile) | nothing – no VPN is installed; the device enrolls in SetupComplete, before anyone logs on, so the WS1 profiles arrive in time |
| **OpenVPN before the logon** (alternative) | lab, sites without Workspace ONE Tunnel yet | installs the OpenVPN client with its service, puts the config into `config-auto`, the service connects at every start – before the logon screen |

**Workspace ONE Tunnel before the logon**: the Windows client (`VMwareTunnelInstaller.exe` from Omnissa Customer Connect - no public download; installed with `/install /quiet /norestart`)
can go to `content/apps` as the application `WorkspaceOneTunnel` (content wizard: `-TunnelFile`) and be ticked in the profile - optional: WS1 can deliver the app itself.
The tunnel configuration comes with the WS1 Tunnel profile after the enrollment (SetupComplete, before anyone logs on); its
*Custom Configuration XML* needs `<StartTunnelPreLogon>true</StartTunnelPreLogon>` so the tunnel starts before the logon.

The alternative never replaces the default: a profile without it (every profile made before v0.20 too) installs no VPN.

## How the alternative works

1. **WinPE, before the disk is wiped**: the config is decrypted and checked – a `remote`, a client certificate
   (`<cert>` + `<key>`, `<pkcs12>` or `cryptoapicert`), no `auth-user-pass` / `askpass` (nobody can type a password before
   the logon), no separate files (`ca ca.crt` ...: `config-auto` gets one file). A wrong config stops here, the old system
   stays intact. The server checks the same when the profile is saved (`VpnRules.CheckOpenVpn` = `Test-WsoOpenVpnConfig`,
   test cases in `tests/openvpn-cases.json`).
2. **PostApply**: the OpenVPN MSI goes to `ProgramData\WSODeployer\Apps`, the config to the protected `Secure` folder;
   `deploy-state.json` gets only the type, the application and the probe – never the config.
3. **SetupComplete** (SYSTEM, after the Entra join, before the other applications): the MSI is installed, the config is
   copied to `config-auto` (path from `HKLM\SOFTWARE\OpenVPN\autostart_config_dir`, default
   `C:\Program Files\OpenVPN\config-auto`, SYSTEM + Administrators only) as `wsodeployer.ovpn`, `OpenVPNService` is set to
   start automatically and restarted. The optional probe (`host:port`, e.g. the DC `10.10.0.10:389`) is only reported – in
   the LAN it answers without the tunnel too. Result: `SetupComplete.Results.vpn` in `deploy-state.json` and the server journal.

## Setting it up

1. **OpenVPN MSI as an application** in `deploy.json` (`content/apps/`), with the *OpenVPN Service* feature:

   ```json
   "OpenVPN": {
     "File": "apps/OpenVPN-2.7.7-I001-amd64.msi",
     "Sha256": "",
     "Arguments": "ADDLOCAL=OpenVPN.Service,OpenVPN,Drivers,Drivers.OvpnDco,Drivers.TAPWindows6",
     "SuccessCodes": [0, 3010]
   }
   ```

   The content wizard (`client\Initialize-WsoContent.ps1`, step *VPN clients*) downloads this MSI from openvpn.net, checks
   the OpenVPN Inc. signature and adds the entry. The feature names are those of the 2.7 MSI (Feature table); another
   version may differ - when SetupComplete logs "The service OpenVPNService is not installed", compare its Feature table.
   `UpdateHash` fills `Sha256`.

2. **One .ovpn with everything inline.** Easiest: upload the client bundle ZIP of the VPN server in the profile (Zentyal:
   *VPN → Servers → Download client bundle*, Windows) – the console puts the certificate / key files into the .ovpn in
   memory, adds `data-ciphers-fallback` for a fixed `cipher` and stores the result encrypted. By hand:

   ```powershell
   $dir = 'C:\Temp\vpn-bundle'          # the unpacked bundle
   $ovpn = Get-ChildItem $dir -Filter *.ovpn | Select-Object -First 1
   $out = foreach ($line in Get-Content $ovpn.FullName) {
       if ($line -match '^\s*(ca|cert|key|tls-auth|tls-crypt)\s+"?([^"\s]+)"?(\s+(\d))?') {
           $tag = $Matches[1]; $file = Join-Path $dir (Split-Path $Matches[2] -Leaf)
           if ($tag -eq 'tls-auth' -and $Matches[4]) { "key-direction $($Matches[4])" }
           "<$tag>"; Get-Content $file | Where-Object { $_ -notmatch '^#' }; "</$tag>"
       }
       else { $line }
   }
   $out | Set-Content (Join-Path $dir 'wsodeployer.ovpn') -Encoding ascii
   ```

   Remove `auth-user-pass` if the bundle has it (the server must accept the certificate alone). An older server (Zentyal 7:
   OpenVPN 2.4) with `cipher AES-256-CBC` needs `data-ciphers-fallback AES-256-CBC` in the config for the 2.6 client.

3. **Profile**: *Domain controller at the first logon* → *OpenVPN before the logon*, the application, the probe (optional),
   paste the config. It is stored CMS-encrypted to the deployment certificate and the vault (devices only can read it);
   the field stays empty when you edit the profile again (empty = keep). Back to *LAN or Workspace ONE Tunnel* deletes the
   stored config.

4. **Test**: device on a network without the DC, after the installation restart it and log on with a domain account that
   never logged on to it – it works only when the DC was reachable before the logon. Logs: `C:\Program Files\OpenVPN\log\`,
   `ProgramData\WSODeployer\Logs\setupcomplete.log`. If the logon screen is faster than the tunnel: GPO *Always wait for the
   network at computer startup and logon*.

## Limits

- **One configuration (one client certificate and key) for N devices – test / lab ONLY, not production.** The VPN server
  cannot tell the devices apart, revoking disconnects them all at once and the key is on every disk (a leak from one device
  = access for all). For a test: a certificate for this profile only (e.g. CN `wso-lab`) and `duplicate-cn` on the server.
  Production: Workspace ONE Tunnel (per-device identity from UEM) or a VPN certificate per device – roadmap (status.md):
  key made on the device (non-exportable, `cryptoapicert`), certificate signed by the WSODeployer server, revoked when the
  device is retired (CRL on the VPN server). AD CS alone is not enough – after ODJ the device has no DC yet.
- The key stays on the device in `config-auto` (SYSTEM, Administrators and the account of the OpenVPN service – in 2.7
  `NT SERVICE\OpenVPNService`) – the service needs it at every start.
- The VPN server must push the **DNS of the domain controller** (e.g. Zentyal: First nameserver = DC, Search domain = the AD
  domain) and the **route to the DC network** – otherwise the tunnel is up and the domain logon says "domain not available".
- No trusted network detection: in the LAN the service connects too.
- Static content (without the server): `PreLogonVpn.ConfigCms` is made by the console only.
- VM test (2026-10-04, preprod): the WinPE check, the MSI and `config-auto` work; starting the service needed the access fix
  (service account of 2.7). Domain logon through the tunnel not checked yet (the Zentyal bundle had the key of another
  certificate).
