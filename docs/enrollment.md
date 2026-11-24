# Device enrollment (v0.11)

Before v0.11 every boot media carried **deploy.pfx** (the private key of the deployment certificate) and technicians
typed its **PIN** in WinPE. The same key signed the requests of every device and decrypted every secret: a lost USB
stick plus a known PIN exposed all of them, and rotating the key meant rebuilding all media.

From v0.11 a device can **enroll** instead (the model of the Winch project):

1. **Console → Device enrollment**: an admin creates
   - a **token** for boot media: the console shows a **join code** once (`WSOD1.…` = server address + token);
   - or a **pairing code** (`ABCD-EFGH`, must expire, at most 168 h by default): universal media carry a join code
     without a token, and the technician types the code in the WinPE wizard.
   A token / code may put new devices into a **device group**; it can be limited in time and number of uses and
   revoked at any time. Only its SHA-256 is stored.
2. **Media**: `.\client\New-WsoBootMedia.ps1 -JoinCode 'WSOD1.…'` writes the join code to `media.json`; no deploy.pfx,
   no PIN.
3. **WinPE**: the engine makes an RSA 2048 key **in memory** (it never leaves the RAM disk), sends a CSR with the
   serial number, MAC addresses, SMBIOS UUID, model and IP addresses (`POST /api/enroll`).
   - a known, approved device gets a certificate of the **server CA** at once;
   - a new device appears in **Approvals** (group by token or group rules) and the wizard waits, as before;
   - the certificate lives **24 h** (`Wso:Enrollment:CertificateHours`) - one deployment.
4. The device signs its requests with that key (`X-Wso-Credential` + the usual signature headers). The server accepts
   it for **its own asset only** (identify, ODJ).
5. **Secrets** (ODJ blob, WS1 staging account, local administrator password, WS1 Drop Ship unattend) are encrypted at
   rest to the deployment certificate **and** to the server's **secret vault**. An enrolled device asks the server for
   a copy encrypted to its own key (`POST /api/device/reprotect`; the ODJ endpoint does it by itself).

## Server keys

`Wso:Enrollment:KeyPath` (Docker: `data/ca`, created on the first start, readable by the API only):
`ca.crt` / `ca.key` (device CA) and `vault.crt` / `vault.key` (secret vault). **Back them up with the database** - a
lost vault key makes the stored secrets unreadable for enrolled devices (media with deploy.pfx still work).
The public vault certificate is `/content/vault.cer`.

## Moving from deploy.pfx

Secrets saved before v0.11 are encrypted to deploy.cer only. For enrolled devices save them again:
- **staging account**: `.\client\Initialize-WsoContent.ps1` (step 4) with the server address known;
- **local administrator**: open the profile, type the password again, save;
- **WS1 Drop Ship package**: upload it again;
- **ODJ blobs**: update the domain agent (it encrypts to the vault from the work item) - new blobs are fine.
An old secret gives the device a clear error (`secret.notInVault`). When all media use join codes, set
`WSO_ALLOW_DEPLOY_PFX=false` (`Wso:Enrollment:AllowDeploymentCertificate`): requests signed with deploy.pfx are refused.

## Security notes

- The token in a join code is a bearer secret: whoever has the media may ask to enroll. A **new** device still needs
  an approval; a **known** device gets a certificate right away - limit tokens by uses / time and revoke lost media.
- The CA signs only the public key of the CSR (whose signature proves the key); subject and extensions are decided by
  the server, so a device cannot claim another identity.
- A revoked credential (Device enrollment → Device certificates) stops a running deployment at its next request.
