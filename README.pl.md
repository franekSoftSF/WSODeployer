# WSODeployer

Instalacja systemu z chmury dla urządzeń **Omnissa Workspace ONE UEM** - komputerów stacjonarnych, laptopów poza siecią
firmy i maszyn wirtualnych. W duchu PSD / MDT, oparta na **OSD / OSDCloud**: urządzenie startuje z nośnika WSODeployer,
dostaje tożsamość z serwera (nazwa z numeru inwentarzowego, ta sama nazwa, obiekt w AD i rekord w Workspace ONE przy
każdej reinstalacji), dołącza do domeny (Hybrid przez Offline Domain Join, Entra ID, grupa robocza) i rejestruje się w
Workspace ONE bez udziału użytkownika.

English: [README.md](README.md)

W tym repozytorium jest **dokumentacja, paczka administratora i obrazy serwera**. Kod źródłowy jest w prywatnym
repozytorium.

## Do pobrania

| Co | Gdzie |
|---|---|
| Paczka administratora `WSODeployer-0.25.0.zip` | [Releases](../../releases) - skrypty stanowiska administratora, agenta domenowego i nośnika, silnik urządzenia, przykładowa treść, zestaw startowy serwera, MSI konektora i aplikacja technika (obok plik `.sha256`) |
| Obrazy serwera | `docker pull ghcr.io/franeksoftsf/wsodeployer-api:0.25.0` i `ghcr.io/franeksoftsf/wsodeployer-nginx:0.25.0` - pobierają je skrypty serwera z paczki, nic nie budujesz |

Uruchomiony serwer podaje swoją wersję pod `/api/health`.

## Dokumentacja

| Dokument | O czym |
|---|---|
| [Instalacja](docs/pl/installation.md) | krok po kroku: serwer, stanowisko administratora, treść, agent domenowy, nośnik, pierwsza VM |
| [Serwer i konsola WWW](docs/server.md) (EN) | skrypty serwera, TLS, logowanie do konsoli (Workspace ONE Access / Entra ID), REST API Workspace ONE UEM |
| [Cloud Connector](docs/pl/connector.md) | usługa Windows w domenie: Offline Domain Join, konta komputerów |
| [Rejestracja urządzeń](docs/pl/enrollment.md) | nośnik bez deploy.pfx: kody dołączenia, zatwierdzanie |
| [Tożsamość urządzenia](docs/pl/identity-design.md) | AD, Hybrid, Entra ID, grupa robocza - przygotowanie labu |
| [Obrazy Windows](docs/pl/images.md) | kopia w LAN i comiesięczna aktualizacja obrazu |
| [Aplikacja technika](docs/pl/technician.md) | sprawdzenie ADK / WinPE / OSD i budowa nośnika przez techników |
| [OpenVPN przed logowaniem](docs/pl/prelogon-vpn.md) | lab / lokalizacje bez Workspace ONE Tunnel |
| [Aktualizacja wersji Windows](docs/pl/windows-feature-upgrade.md) | skrypt aktualizacji w miejscu dla Workspace ONE |

Nazwy domen, kont i serwerów w dokumentacji są przykładowe.

## Wymagania w skrócie

- Serwer: Docker (Linux albo Windows z kontenerami Linux); ok. 6 GB dysku na każdy obraz i język Windows w kopii LAN.
- Stanowisko administratora: Windows 10/11 albo Server, PowerShell 5.1, Windows ADK + dodatek WinPE (instaluje je skrypt).
- Domena: serwer Windows w domenie dla konektora (Offline Domain Join).
- Workspace ONE UEM: adres Device Services, Group ID, konto staging; opcjonalnie klient OAuth REST API.

## Autor

Szymon Frankiewicz - [digitalworkspace.pl](https://digitalworkspace.pl)

Licencja: MIT (zobacz [LICENSE](LICENSE)).
