# Serwer WSODeployer – instalacja z wydania

English: [../server-setup.md](../server-setup.md)

Ten przewodnik prowadzi od pustego hosta do pierwszego logowania w konsoli WWW. Resztę (stanowisko administratora, treść,
Cloud Connector, nośnik) opisuje [installation.md](installation.md).

## 1. Co pobrać

Wszystko jest w **Releases** publicznego repozytorium `franekSoftSF/WSODeployer`. Obok każdego ZIP-a jest plik `.sha256`.

| Plik | Co to jest | Gdzie trafia |
|---|---|---|
| `WSODeployer-server-X.Y.Z.zip` | **Sam serwer**: `server\docker-compose.yml`, `server\.env.example`, `server\scripts\run-linux.sh` i `run-windows.ps1`, MSI konektora i aplikacja technika w `server\data\downloads`, przykładowa treść w `content\` | host z Dockerem |
| `WSODeployer-X.Y.Z.zip` | **Pełna paczka administratora**: serwer jak wyżej oraz skrypty stanowiska administratora (`client\`), agenta domenowego / konektora (`agent\`), silnik urządzenia (`engine\`) i rozszerzenie konsoli UEM | stanowisko administratora (i host z Dockerem, jeśli to ta sama maszyna) |
| `docker-compose.yml`, `.env.example` | Te same dwa pliki osobno, do przejrzenia przed pobraniem czegokolwiek | - |

Obrazy serwera pobierają skrypty, nic nie budujesz:
`ghcr.io/franeksoftsf/wsodeployer-api:X.Y.Z` i `ghcr.io/franeksoftsf/wsodeployer-nginx:X.Y.Z`, oraz `postgres:17-alpine`.

## 2. Wymagania

### Host

| | Minimum (lab, do ok. 50 urządzeń) | Zalecane (produkcja) |
|---|---|---|
| CPU | 2 vCPU | 4 vCPU |
| RAM | 4 GB | 8 GB |
| Dysk na serwer | 40 GB | 100 GB + ok. 6 GB na każdy obraz i język Windows trzymany w LAN |
| System | Ubuntu 22.04/24.04, Debian 12, RHEL 9 (Docker Engine) albo Windows 10/11 Pro/Enterprise z Docker Desktop | maszyna wirtualna z Linuksem (Ubuntu 24.04 LTS) |

Rozmiary to szacunki do planowania, nie limity. Zmierzone na serwerze testowym: trzy kontenery bez obciążenia zajmują ok.
200 MB RAM (API ok. 125 MB, PostgreSQL ok. 50 MB, nginx ok. 25 MB). Obrazy zajmują ok. 0,9 GB. Większość dysku zajmują
kopie obrazów Windows w LAN (`data/os`) oraz pakiety i aplikacje, które wgrywasz.

### Oprogramowanie

**Linux** (`run-linux.sh`):
- Docker Engine 24 lub nowszy z wtyczką compose (`docker compose version` musi działać). Instalacja:
  https://docs.docker.com/engine/install/.
- `curl`. `perl` jest potrzebny tylko do `--domain`.
- root albo użytkownik w grupie `docker`.

**Windows** (`run-windows.ps1`):
- **Docker Desktop** (https://www.docker.com/products/docker-desktop/) z backendem **WSL 2**. Musi być **uruchomiony** i
  ustawiony na **kontenery Linux** (ikona w zasobniku → *Switch to Linux containers…*, jeśli to proponuje).
- Windows 10/11 Pro albo Enterprise. Docker Desktop nie jest obsługiwany na Windows Server: tam użyj maszyny z Linuksem.
- PowerShell 5.1 lub 7, **uruchomiony jako administrator**. `curl.exe` jest częścią Windows.
- Jeśli ZIP pochodzi z internetu, najpierw odblokuj skrypty:
  `Get-ChildItem -Recurse .\server\scripts | Unblock-File`.

### Sieć

| Port | Kierunek | Po co |
|---|---|---|
| 443/tcp (albo Twój `WSO_HTTPS_PORT`) | urządzenia, stanowisko administratora, Cloud Connector, przeglądarki → serwer | API i konsola WWW (nginx, TLS) |
| 80/tcp (opcjonalnie) | → serwer | tylko przekierowanie na HTTPS |
| wychodzący 443 | serwer → `ghcr.io`, Docker Hub | obrazy (pierwsze uruchomienie i aktualizacje) |
| wychodzący 443 | serwer → `*.awmdm.com`, `*.uemauth.workspaceone.com` | tylko z REST API Workspace ONE UEM |

Urządzenia **nie** potrzebują dostępu do Active Directory. Pracę w domenie (Offline Domain Join, konta komputerów)
wykonuje **Cloud Connector**: usługa Windows na serwerze w domenie, która sama łączy się z serwerem WSODeployer.

## 3. Instalacja krok po kroku

### Linux
Serwer mieszka w jednym stałym folderze (`/opt/wsodeployer`, `C:\WSODeployer`), do którego kopiujesz każdą aktualizację.
```bash
unzip WSODeployer-server-X.Y.Z.zip -d /tmp/wso
sudo mkdir -p /opt/wsodeployer && sudo cp -r /tmp/wso/WSODeployer-server-X.Y.Z/. /opt/wsodeployer/
cd /opt/wsodeployer
sudo ./server/scripts/run-linux.sh --host=wso.firma.pl
```

### Windows (PowerShell jako administrator)
```powershell
Expand-Archive .\WSODeployer-server-X.Y.Z.zip $env:TEMP\wso -Force
New-Item -ItemType Directory C:\WSODeployer -Force | Out-Null
Copy-Item $env:TEMP\wso\WSODeployer-server-X.Y.Z\* C:\WSODeployer -Recurse -Force
cd C:\WSODeployer
Get-ChildItem -Recurse .\server\scripts | Unblock-File
.\server\scripts\run-windows.ps1 -PublicHost wso.firma.pl
```

### O co pyta skrypt
1. **Adres serwera** (`--host` / `-PublicHost`): nazwa DNS, której używają urządzenia i przeglądarki. Najpierw dodaj ją do DNS.
2. **Porty**: 443 / 80, jeśli są wolne, inaczej proponuje kolejne wolne.
3. **Certyfikat wdrożeniowy** `deploy.cer`: naciśnij **Enter**, jeśli jeszcze go nie masz. Utworzysz go później na
   stanowisku administratora i przekażesz serwerowi przez `--deployment-cert=` / `-DeploymentCertificate`.
4. **Certyfikat TLS**: ścieżka do **PFX** Twojej firmy / CA (potem hasło) albo **Enter** dla certyfikatu samopodpisanego.
5. Podsumowanie. Po potwierdzeniu skrypt pobiera obrazy i uruchamia serwer.

Wszystkie odpowiedzi trafiają do `.env`. Ponowne uruchomienie skryptu (w każdej chwili, np. po aktualizacji) o nic nie
pyta i nigdy nie nadpisuje kluczy ani haseł.

### Na koniec
Skrypt czeka, aż odpowie `https://<serwer>/api/health`, i wypisuje:
- adres konsoli: `https://<serwer>/console`;
- **użytkownika `admin` i jego hasło**. Hasło jest też w `server/secrets/local-admin.txt` (dostęp tylko dla Ciebie).
  Zaloguj się nim. Ten lokalny administrator istnieje zawsze, także gdy później logujesz się przez Workspace ONE Access
  albo Entra ID. Nowe hasło: `run-linux.sh local-admin` / `run-windows.ps1 -Action local-admin`.

## 4. Certyfikat TLS

| Wybór | Jak | Potem |
|---|---|---|
| **Samopodpisany** (domyślnie) | Enter przy pytaniu; nginx tworzy go dla `WSO_PUBLIC_HOST` (+ `WSO_EXTRA_SAN`) | daj `server/tls/server.cer` stanowiskom administratora, Cloud Connectorowi i nośnikom (Trusted Root) – tamtejsze skrypty o niego pytają |
| **PFX** Twojej firmy / publicznego CA | `--pfx=/ścieżka/wso.pfx` / `-TlsPfx C:\ścieżka\wso.pfx` albo ścieżka przy pytaniu | z publicznym CA nic nie rozdajesz; z wewnętrznym CA jego root musi być zaufany (w domenie zwykle już jest) |

Zmiana później, bez ruszania reszty (stary certyfikat trafia do `tls/old-<czas>`):
```bash
sudo ./server/scripts/run-linux.sh tls --pfx=/root/wso.pfx      # albo: tls --self-signed
```
```powershell
.\server\scripts\run-windows.ps1 -Action tls -TlsPfx C:\certs\wso.pfx   # albo: -Action tls -SelfSigned
```
PFX jest odczytywany w kontenerze nginx (openssl), a jego hasło jest tylko w środowisku tego jednego procesu. Certyfikat
z łańcuchem trafia do `server.crt`, klucz do `server.key`, a kopia DER do `server.cer`. Skrypt sprawdza, czy klucz pasuje
do certyfikatu. PFX wyeksportowane z Windows (3DES) działają.

## 5. Klucze i sekrety (tworzy je skrypt przy pierwszym uruchomieniu)

| Co | Gdzie | Do czego |
|---|---|---|
| Hasło bazy danych | `.env` `WSO_DB_PASSWORD` | PostgreSQL |
| Klucz API administratora | `secrets/admin.key` (w `.env` tylko jego SHA-256) | stanowisko administratora (`Install-WsoWorkstation.ps1 -AdminKeyFile`) |
| Klucz agenta | `secrets/agent-1.key` (skrót w `.env`) | agent domenowy (starsze instalacje; Cloud Connector dostaje kod w konsoli) |
| Lokalny administrator konsoli | `secrets/local-admin.txt` (skrót PBKDF2 w `.env`) | pierwsze logowanie |
| Sejf sekretów + CA urządzeń | `data/ca/` (tworzy go API przy pierwszym starcie) | szyfruje konto staging, pakiety i hasła lokalnego administratora; podpisuje certyfikaty urządzeń |
| Klucze ochrony danych | wolumen Dockera `keys` | szyfrują ustawienia konsoli w bazie (sekret API UEM, certyfikat SAML, sekret OIDC) i ciasteczka logowania |
| Certyfikat TLS | `tls/` | HTTPS |

Nie ma osobnego klucza głównego (KEK), który trzeba wpisywać albo przechowywać osobno: **kluczami są sejf sekretów w
`data/ca` i klucze ochrony danych w wolumenie `keys`**. Kopia bazy bez nich jest bezużyteczna, bo zaszyfrowanych sekretów
nie da się odczytać. Przechowuj je jak klucze: zapisuje je akcja `backup` (niżej), a kto ma kopię, odczyta sekrety.

## 6. Foldery i wolumeny

Wszystkie foldery są obok `docker-compose.yml` (`server/`). Ścieżki zmieniasz przez `WSO_DATA` / `WSO_TLS` w `.env`.

| Folder / wolumen | Kontener | Zawartość | Rozmiar | Kopia |
|---|---|---|---|---|
| `data/content` | api `/data/content` | `deploy.json`, `assets.csv`, Hub, aplikacje, zaszyfrowane sekrety (`secrets/*.cms`) | Hub ok. 300 MB + Twoje aplikacje | **tak** |
| `data/ca` | api `/data/ca` | sejf sekretów + CA urządzeń | KB | **tak – krytyczne** |
| `data/cert` | api (tylko odczyt) | `deploy.cer` (publiczna część certyfikatu wdrożeniowego) | KB | tak |
| `data/os` | api `/data/os` | kopie obrazów Windows w LAN | ok. 6 GB na obraz i język | nie (powstają ponownie z Microsoftu) |
| `data/packages` | api `/data/packages` | pakiety provisioning wgrane w konsoli | Twoje pakiety | tak |
| `data/downloads` | api (tylko odczyt) | MSI konektora, aplikacja technika (konsola → Pobieranie) | ok. 100 MB | nie (są w ZIP wydania) |
| `tls/` | nginx `/etc/nginx/certs` | `server.crt`, `server.key`, `server.cer` | KB | tak |
| wolumen `pgdata` | db | baza: urządzenia, wdrożenia, profile, audyt | dziesiątki MB | **tak** (`pg_dump`) |
| wolumen `keys` | api `/keys` | klucze ochrony danych | KB | **tak – krytyczne** |
| `secrets/`, `.env` | – (tylko host) | klucze i hasła z sekcji 5 | KB | **tak** |

## 7. Kopia i odtwarzanie

```bash
sudo ./server/scripts/run-linux.sh backup
```
```powershell
.\server\scripts\run-windows.ps1 -Action backup
```
Powstaje `backups/wso-<czas>.dump` (baza) i `backups/wso-<czas>-keys/` (`data/ca`, `data/cert`, `.env`, `secrets`, `tls`,
klucze ochrony danych). Skopiuj oba poza host. `data/content` i `data/packages` to zwykłe pliki: dodaj je do kopii plików.

Odtwarzanie na nowym hoście:
1. Rozpakuj to samo wydanie. Skopiuj z powrotem `.env`, `secrets/`, `tls/` i `data/` (razem z `ca` i `cert` z kopii kluczy).
2. Uruchom skrypt raz: utworzy kontenery i zachowa wszystkie wartości.
3. Przywróć klucze ochrony danych do wolumenu `keys`, a bazę przez `pg_restore` (w `server/`):
   ```bash
   docker compose -p wsodeployer cp backups/wso-<czas>-keys/dataprotection-keys/. api:/keys/
   docker compose -p wsodeployer exec -T db pg_restore -U wso -d wso --clean < backups/wso-<czas>.dump
   docker compose -p wsodeployer restart api
   ```

## 8. Active Directory

Serwer nie dołącza do domeny. Pracę w domenie wykonuje **Cloud Connector**: konsola → **Cloud Connectors** →
**Konfigurator konektora** (konto usługi, domena, OU, MSI na serwerze w domenie). Zobacz [connector.md](connector.md).
Opcjonalny skrót przy pierwszym uruchomieniu: `--domain=ad.firma.pl --ou="OU=Komputery,DC=ad,DC=firma,DC=pl"`
(`-Domain` / `-DefaultOU`) wpisuje domenę i domyślne OU komputerów do `deploy.json` serwera.

## 9. Aktualizacja

Pobierz nowy `WSODeployer-server-X.Y.Z.zip` i skopiuj jego zawartość do tego samego folderu co w sekcji 3. ZIP nigdy nie
zawiera `.env`, `secrets/`, `tls/` ani Twoich danych; podmieniane są tylko `server/data/downloads` i skrypty. Potem
uruchom skrypt ponownie. Skrypt pobierze nowe obrazy i zrestartuje kontenery, a API przy starcie
zaktualizuje bazę. Wcześniej zrób kopię.

## 10. Dalej

Zaloguj się jako `admin` i przejdź do [installation.md](installation.md): stanowisko administratora, certyfikat
wdrożeniowy, treść, Cloud Connector, nośnik, pierwsze urządzenie.
