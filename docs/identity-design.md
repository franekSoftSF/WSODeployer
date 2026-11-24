# Device identity: Entra ID, Hybrid and Workspace ONE – design (v0.15, lab)

Goal (2026-10-03): **every** deployment ends with a device **managed by Workspace ONE**, in one of three identity models
chosen in the profile:

| Profile mode | Identity | Who joins | OOBE | Sign-in |
|---|---|---|---|---|
| **Entra – OOBE** | Microsoft Entra joined | the **user** in the classic OOBE ("Set up for work or school") | classic | Entra work account |
| **Entra – no user (bulk)** | Microsoft Entra joined | the **device itself** (bulk token from WCD), hands-off | classic or short | Entra work account on the sign-in screen |
| **Hybrid** | AD DS + Microsoft Entra hybrid joined | WSODeployer (ODJ in WinPE) + Entra Connect (device registration) | classic or short | domain account (UPN = Entra account) |

Workspace ONE: in every mode the **Intelligent Hub with the staging account in SetupComplete** (before OOBE - works
today, verified on a VM), or - for "Entra – OOBE" with Entra ID P1 - **automatic MDM enrollment** when joining Entra
(Drop Ship "Premium" package: Hub with `ENROLL=N`).

## 1. Flows

### Entra – no user (bulk) – hands-off
1. **WinPE** (unchanged): device enrollment (join code), name from the inventory number, Windows 11 through OSDCloud.
2. **Still in WinPE**: the **WCD "Enroll in Azure AD" package** (bulk token) is decrypted, checked and put into the
   protected `Secure` folder of the installed Windows - **not** applied offline: Windows applies an offline provisioning
   package in specialize (`provtool /Turn 1`), where the Entra join never finished and Setup hung on "Getting ready"
   (lab VM, 2026-10-03, fixed in 0.16.1). An optional WS1 Drop Ship package is still applied offline (DISM).
   Computer name, region and the local administrator password come from WSODeployer (unattend), not from the packages.
3. **SetupComplete (SYSTEM, before OOBE)**: first the **Entra join** - wait for login.microsoftonline.com, time sync,
   `Install-ProvisioningPackage -QuietInstall` in its own process with a 10-minute limit, `dsregcmd` check (a failure is
   reported, setup goes on); then base apps, **WS1 enrollment with the staging account** (the device is in WS1 before
   anyone signs in - Wi-Fi / VPN / certificate profiles arrive early), report to the console, `Secure` deleted.
4. **OOBE**: the device is already joined. The "package_…" account of Entra never signs in - it only joins.
5. **Sign-in screen**: the user signs in with the Entra work account; WS1 assigns the device to the signed-in user
   (`ASSIGNTOLOGGEDINUSER`).

### Entra – OOBE (classic, work account)
1.–3. as above, **without the bulk package**; the OOBE is **not hidden** (WSODeployer does not skip the account pages).
4. **OOBE**: the user chooses "Set up for work or school" and signs in with the work account → Entra join.
5. WS1: the Hub staging enrollment from SetupComplete (already in WS1, assigned at sign-in), or - with Entra ID P1 -
   **automatic MDM enrollment into WS1** right after the join (the Drop Ship "Premium" package that works today).
   Never both (profile *Workspace ONE enrollment*: **Microsoft Entra** = automatic, no Hub - the default of the Entra - OOBE
   template since 0.22.6 - or the Hub with a staging account): with the Hub the OOBE users must not be in the MDM user scope of **any** MDM app in Entra (Mobility - check
   every app, e.g. "AirWatch by VMware" and "Workspace ONE"); otherwise the OOBE join tries to enroll the device again and
   stops with **8018000a** (already enrolled) - seen on the preprod VM 2026-10-10.
   With the automatic enrollment a **reinstall** of a device needs its old UEM device record deleted first (same serial
   number: UEM answers 8018000a as well); the Hub with a staging account reuses the record instead.

### Hybrid (AD + Entra)
1.–2. WinPE: as today - the **connector** makes the computer account (`djoin /provision /reuse`) in the OU of the profile /
   group; the ODJ blob goes into the unattend (encrypted to the device).
3. SetupComplete: WS1 enrollment with the staging account.
4. First boot: domain member. **Entra Connect Sync** synchronises the computer object (OU in the sync scope), Windows
   registers with Entra through the "Automatic-Device-Join" task (the SCP in AD points to the tenant) - "Microsoft Entra
   hybrid joined" after the next sync cycle (up to ~30 min by default).
5. Domain sign-in (UPN = Entra account) - the first sign-in needs a domain controller (office network or pre-logon VPN).

## 2. Lab preparation

### Entra ID (tenant)
- **Devices → Device settings**:
  - "Users may join devices to Microsoft Entra" = All (or a group with the "package_…" account of WCD);
  - "Require MFA to register or join devices" = **No** (a bulk token cannot do MFA) - or exclude the "package_…" account
    from the Conditional Access policy for the action "Register or join devices";
  - maximum devices per user - make sure it does not block the "package_…" account (lab: raise it).
- **Mobility (MDM and WIP) → Workspace ONE (AirWatch)**: *MDM user scope* = **Some** (a group of real users), **never All**
  and never a group with the "package_…" account. With *All* the bulk package also enrolls the device in WS1 as "package_…",
  WS1 does not know that user, and Windows rolls back the whole join: `0x801900C8` at `AADJ/BPRT` (seen in the lab).
  WSODeployer enrolls in WS1 itself (Hub + staging); the Entra – OOBE mode needs the scope for its real users.
  - The scope has no exclusions: check the groups of the "package_…" account (Users → package_… → Groups). A **dynamic**
    group gets `-and -not (user.userPrincipalName -startsWith "package_")` - that also covers every future package.
  - The switch *Disable MDM enrollment when adding work or school account* does not help: it is about Entra *registered*
    devices (Settings → Accounts), not the join of the bulk package or the OOBE.
  - Sign of this case: the Entra audit shows *Register device* by package_… and *Unregister device* a few seconds later;
    the engine (0.22.3+) logs it as an MDM scope problem and its retry task joins once the scope is fixed.
  - Applying the package again by hand on a device fails with `0x800700B7` (same package id + version): export the same
    WCD project with a higher version (e.g. 1.2 → 1.3); the token and the package_ account stay.
- **Hybrid**: Entra Connect **Sync** (Cloud Sync does not synchronise devices) with *Configure device options → Hybrid
  Microsoft Entra join* (creates the SCP in AD), the workstation OU in the sync scope.
- **Optional (Entra – OOBE with P1)**: Mobility (MDM and WIP) → **Workspace ONE UEM** app with the MDM URLs of the WS1
  console (Settings → Windows → Entra integration), MDM user scope.

### Hybrid join in Entra Connect Sync – step by step
Your lab has Entra Connect without Hybrid; this turns it on (no change to the users already synchronised):
1. On the Entra Connect server: **Microsoft Entra Connect** → *Configure* → **Configure device options** → sign in with
   a Global Administrator (or Hybrid Identity Administrator) of the tenant.
2. *Device options*: **Configure Hybrid Microsoft Entra join**.
3. *Device operating systems*: **Windows 10 or later domain-joined devices**.
4. *SCP configuration*: tick the forest, *Authentication Service* = **Microsoft Entra ID**, enter an Enterprise
   Administrator of the forest → *Configure*. This creates
   `CN=62a0ff2e-97b9-4513-943f-0d221bd30080,CN=Device Registration Configuration,CN=Services,<configuration NC>` with
   `azureADName:<tenant>` and `azureADId:<tenant id>` (without Enterprise Admin rights the wizard gives a PowerShell
   script for the forest admin).
5. **Sync scope**: Entra Connect → *Customize synchronization options* → *Domain and OU filtering* - the OUs of the
   WSODeployer profiles / groups must be ticked (computer objects outside the scope never get to Entra).
6. Check: the **Cloud Connectors** page shows *Hybrid join SCP: <tenant>* per connector, and the profile editor shows
   "a connector found the SCP" when you choose Hybrid. On a deployed device: `dsregcmd /status` → `AzureAdJoined : YES`,
   `DomainJoined : YES` (after the next sync cycle; `Start-ADSyncSyncCycle -PolicyType Delta` on the Entra Connect server
   speeds it up).
7. Note: a Conditional Access policy that needs a "hybrid joined" device must not apply to the first sign-in of a new
   device (the state appears only after the sync).

### Workspace ONE with Entra P1 (the user joins in OOBE)
- WS1 console → *Groups & Settings → All Settings → Devices & Users → Windows → Windows Desktop → Directory Services /
  Microsoft Entra ID integration*: enable, copy the **MDM enrollment URL**, **MDM terms of use URL** and **MDM discovery
  URL**.
- Entra → *Mobility (MDM and WIP)* → **Workspace ONE UEM** (add from the gallery if missing): paste the three URLs, *MDM
  user scope* = All (or the group of the users / staff).
- Then a user who joins Entra in OOBE is enrolled in WS1 by Windows itself; the Hub of the profile (`ENROLL=N`, Premium
  XML) only adds the agent. Without P1 keep the staging enrollment of SetupComplete (`ENROLL=Y`).

### Network for the Entra join
- Devices must resolve and reach `login.microsoftonline.com`, `enterpriseregistration.windows.net` and
  `device.login.microsoftonline.com` directly: no DNS interception and no TLS inspection by a VPN / ZTNA agent or proxy.
  Lab 2026-10-04: GlobalProtect on the host answered `login.microsoftonline.com` with `100.64.1.1` (even for 8.8.8.8), the
  VM behind VMware NAT inherited it - join error `0xCAA70004` *No connectivity*. Bridged network or GlobalProtect off fixes
  it; the retry task then joins without a new installation. The SetupComplete log shows the DNS answer.

### Windows Configuration Designer – the bulk package (once per ≤180 days)
1. WCD (ADK feature "Configuration Designer" or Microsoft Store) → **Provision desktop devices**.
2. *Set up device*: **device name empty** (WSODeployer names the device), no removal of apps.
3. *Set up network*: skip (lab: cable) or a Wi-Fi profile if OOBE runs on Wi-Fi.
4. *Account management*: **Enroll in Azure AD**, *Bulk Token Expiry* (180 days at most) → **Get Bulk Token** (sign in with
   an account allowed to join devices - Entra creates the account "package_<guid>").
5. *Add applications / certificates*: nothing (WS1 and WSODeployer do that); **local administrator: no** (the profile).
6. *Finish* → **Create** without package encryption (an encrypted package needs a password when applied - not offline).
7. The `.ppkg` (it holds the token - a **password for joining devices to the tenant**) → `work\` → console (see §4).

### Workspace ONE
- OG and staging account as today - used in SetupComplete.
- Lab Hybrid: a profile with domain = AD, the OU in the Entra Connect scope.

### WSODeployer
- Cloud Connector (Hybrid) - MSI from the console, delegated account; packages in the console; profiles.

## 3. What I build in WSODeployer (v0.15)

1. **Profile → "Device identity"**: Workgroup / AD (ODJ) / **Hybrid (AD + Entra)** / **Entra – OOBE** / **Entra – bulk**.
   The domain follows the mode (AD / Hybrid = ODJ, Entra = none); validation:
   - Entra – bulk needs a package of the type **"Entra bulk"**;
   - Entra – OOBE must not hide the account pages of the OOBE;
   - Hybrid needs a connector and an OU.
2. **Profile → "OOBE"**: **Classic** (privacy pages, work account - Windows asks) or **Short** (as today). Region and
   computer name always from WSODeployer.
3. **"Entra bulk" package** (console → Packages): type, token expiry (the console warns 30 days before), the
   "package_…" account. **The package with the token is not public**: not in `/content/packages` (reachable without
   sign-in) but encrypted to the server vault and handed **only to an enrolled device** through a signed request (like the
   ODJ blob).
4. **WS1 enrollment in every mode**: Hub staging in SetupComplete (default), Drop Ship package, or "MDM from Entra (P1)" -
   the WinPE wizard summary shows the identity + the WS1 enrollment.
5. **Connector – Hybrid**: the heartbeat checks the **Hybrid Join SCP** in AD (tenant, domain) - the console shows whether
   the forest is ready for Hybrid.
6. **Later (phase 2, connector + Microsoft Graph)**: Entra state of the device in the console (joined / hybrid pending /
   hybrid joined), removal of the old Entra record on reinstall and retire, Entra groups; Autopilot for Intune customers.

## 4. Security
- Bulk token = the right to join devices to the tenant until it expires. Stored encrypted (vault), handed only to an
  enrolled device, never in public content; if leaked: **block / delete the "package_…" account in Entra**.
- Local administrator password from the profile (not from the WS1 packages - their `AdministratorPassword` is only encoded).
- WS1 Drop Ship packages: the unattend.xml is encrypted on upload (staging account), as today.

## 5. Lab test plan

| # | Scenario | Expected |
|---|---|---|
| 1 | Entra – bulk, short OOBE, Hub staging | hands-off: Entra joined (`dsregcmd /status`: AzureAdJoined YES), in WS1 before sign-in, Entra sign-in, device assigned to the user |
| 2 | Entra – bulk, classic OOBE | as 1, OOBE shows the Windows pages but no account page (already joined) - **to confirm** |
| 3 | Entra – OOBE, Hub staging | the user joins in OOBE, the device is already in WS1, assigned at sign-in |
| 4 | Entra – OOBE, Drop Ship Premium (P1) | join in OOBE → automatic MDM enrollment into WS1 |
| 5 | Hybrid, office network | computer account in the OU (connector), domain member after the first boot, hybrid joined after the Entra Connect sync, in WS1 |
| 6 | Reinstall of the same device (1 and 5) | same name / AD account (`/reuse`) / WS1 record; Entra: old record to clean up (phase 2) |
| 7 | Bulk token expired / "package_…" blocked | clear error in the deployment report, the device stays in a workgroup + WS1 |

To confirm in the lab (not verified by me): the bulk package through Install-ProvisioningPackage in SetupComplete, the classic OOBE
after the token join (scenario 2), WS1 assignment to the Entra user after the staging enrollment before OOBE.
