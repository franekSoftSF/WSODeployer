# WSODeployer

Cloud OS deployment for **Omnissa Workspace ONE UEM** endpoints - physical PCs, off-network laptops and VMs. In the
spirit of PSD / MDT, built on **OSD / OSDCloud**: a device boots the WSODeployer media, gets its identity from the
server (name from the inventory number, the same name, AD object and Workspace ONE record on every re-install), joins
the domain (Hybrid via Offline Domain Join, Entra ID, workgroup) and enrolls into Workspace ONE without user action.

Polski: [README.pl.md](README.pl.md)

This repository has the **documentation, the admin package and the server images**. The source code is kept in a
private repository.

## Get it

| What | Where |
|---|---|
| Admin package `WSODeployer-0.25.0.zip` | [Releases](../../releases) - scripts of the admin workstation, domain agent and boot media, device engine, sample content, server start kit, connector MSI and technician app (`.sha256` next to it) |
| Server images | `docker pull ghcr.io/franeksoftsf/wsodeployer-api:0.25.0` and `ghcr.io/franeksoftsf/wsodeployer-nginx:0.25.0` - the server scripts of the package pull them, nothing to build |

The running server reports its version at `/api/health`.

## Documentation

| Document | About |
|---|---|
| [Installation](docs/installation.md) | step by step: server, admin workstation, content, domain agent, boot media, first VM |
| [Server and web console](docs/server.md) | server scripts, TLS, console sign-in (Workspace ONE Access / Entra ID), Workspace ONE UEM REST API |
| [Cloud Connector](docs/connector.md) | Windows service in the domain: Offline Domain Join, computer accounts |
| [Device enrollment](docs/enrollment.md) | media without deploy.pfx: join codes, approvals |
| [Device identity](docs/identity-design.md) | AD, Hybrid, Entra ID, workgroup - lab preparation |
| [Windows images](docs/images.md) | LAN copy and monthly servicing of the Windows image |
| [Technician app](docs/technician.md) | ADK / WinPE / OSD check and media builder for technicians |
| [OpenVPN before logon](docs/prelogon-vpn.md) | lab / sites without Workspace ONE Tunnel |
| [Windows feature upgrade](docs/windows-feature-upgrade.md) | in-place upgrade script for Workspace ONE |

Every document also exists in Polish under [docs/pl](docs/pl). Names of domains, accounts and servers in the
documentation are examples.

## Requirements in short

- Server: Docker (Linux or Windows with Linux containers); about 6 GB disk per Windows image and language for LAN copies.
- Admin workstation: Windows 10/11 or Server, PowerShell 5.1, Windows ADK + WinPE add-on (installed by the script).
- Domain: a domain-joined Windows server for the connector (Offline Domain Join).
- Workspace ONE UEM: Device Services host, Group ID, staging account; optional REST API OAuth client.

## Author

Szymon Frankiewicz - [digitalworkspace.pl](https://digitalworkspace.pl)

License: MIT (see [LICENSE](LICENSE)).
