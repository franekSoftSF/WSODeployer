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
| Serwer `WSODeployer-server-0.25.1.zip` | [Releases](../../releases) - `docker-compose.yml`, `.env.example`, `run-linux.sh` / `run-windows.ps1` (klucze, certyfikat TLS, hasło admina), przykładowa treść, MSI konektora i aplikacja technika |
| Paczka administratora `WSODeployer-0.25.1.zip` | [Releases](../../releases) - serwer jak wyżej oraz skrypty stanowiska administratora, agenta domenowego i nośnika, silnik urządzenia |
| `docker-compose.yml`, `.env.example` | [Releases](../../releases) - te same dwa pliki osobno |
| Obrazy serwera | `docker pull ghcr.io/franeksoftsf/wsodeployer-api:0.25.1` i `ghcr.io/franeksoftsf/wsodeployer-nginx:0.25.1` - pobierają je skrypty serwera z paczki, nic nie budujesz |

Uruchomiony serwer podaje swoją wersję pod `/api/health`.

## Dokumentacja

| Dokument | O czym |
|---|---|
| [Instalacja serwera](docs/pl/server-setup.md) | **zacznij tutaj**: wymagania (minimalne / zalecane), porty, instalacja krok po kroku na Linuksie albo Windows (Docker Desktop), certyfikat TLS (samopodpisany albo PFX), klucze, foldery i wolumeny, kopia, pierwsze logowanie jako `admin` |
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

Szczegóły, rozmiary minimalne i zalecane: [instalacja serwera](docs/pl/server-setup.md).

- Serwer: Linux z Docker Engine albo Windows 10/11 z Docker Desktop (kontenery Linux); 2-4 vCPU, 4-8 GB RAM,
  40-100 GB dysku plus ok. 6 GB na każdy obraz i język Windows trzymany w LAN.
- Stanowisko administratora: Windows 10/11 albo Server, PowerShell 5.1, Windows ADK + dodatek WinPE (instaluje je skrypt).
- Domena: serwer Windows w domenie dla konektora (Offline Domain Join).
- Workspace ONE UEM: adres Device Services, Group ID, konto staging; opcjonalnie klient OAuth REST API.

## Autor

Szymon Frankiewicz - [digitalworkspace.pl](https://digitalworkspace.pl)

Licencja: MIT (zobacz [LICENSE](LICENSE)).
