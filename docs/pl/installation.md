# Instalacja WSODeployer – krok po kroku (tryb serwerowy)

English: [../installation.md](../installation.md)

Kolejność: **serwer → stanowisko administratora → certyfikat na serwer → treść → agent domenowy → nośnik → test na VM**.
Wszystko uruchamiasz z folderu rozpakowanej paczki wydania (niżej); pliki lokalne trafiają do jej folderu `work\`.
Każdy skrypt pyta o brakujące wartości, pokazuje plan przed zmianami i obsługuje `-NonInteractive` oraz `-Language en|pl`.

## 0. Do pobrania (stan: 2026-10-02)

### WSODeployer
| Co | Skąd |
|---|---|
| Paczka administratora `WSODeployer-<wersja>.zip` (skrypty stanowiska administratora, agenta domenowego i nośnika, silnik urządzenia, przykładowa treść, zestaw startowy serwera, MSI konektora i aplikacja technika) | **Releases** publicznego repozytorium `franekSoftSF/WSODeployer` (obok plik `.sha256`) |
| Obrazy serwera `wsodeployer-api`, `wsodeployer-nginx` | `ghcr.io/franeksoftsf/` - pobierają je skrypty serwera z paczki, nic nie budujesz |

Rozpakuj ZIP na stanowisku administratora (np. `C:\WSODeployer`) i skopiuj jego folder `server\` na host z Dockerem (Linux albo
Windows). Folder `server\data\downloads` paczki ma już MSI konektora i aplikację technika.

### Windows ADK + dodatek WinPE (stanowisko administratora)
ADK i dodatek WinPE **muszą mieć tę samą wersję**. Aktualna wersja to **ADK 10.1.26100.9457 (wrzesień 2026)**. Zastępuje ona
10.1.26100.2454 i obsługuje Windows 11 26H2, 25H2 i 24H2 oraz Windows Server 2025/2022.

| Plik | Link |
|---|---|
| Windows ADK 10.1.26100.9457 | https://go.microsoft.com/fwlink/?linkid=2289980 |
| Dodatek Windows PE dla ADK 10.1.26100.9457 | https://go.microsoft.com/fwlink/?linkid=2289981 |
| Strona Microsoft (opis wersji) | https://learn.microsoft.com/windows-hardware/get-started/adk-install |
| Poprawki ADK (msp) | https://learn.microsoft.com/windows-hardware/get-started/adk-servicing |

- Przy instalacji ADK wystarczy zaznaczyć **Deployment Tools**. W dodatku WinPE instalujesz całość.
- **Nie używaj ADK 10.1.28000.1**. Ta wersja jest tylko dla Windows 11 26H1 Arm64, a w winget wygląda na „najnowszą”.
- Jeśli wolisz winget (`Install-WsoWorkstation.ps1` robi to sam), dostaniesz 10.1.26100.2454 dla obu składników.
  To ta sama linia wydań: doinstaluj poprawkę **KB5101684** (https://aka.ms/Windows_ADK_10.1.26100.2454_Update_KB5101684.zip).
  Zawiera poprawki bezpieczeństwa DISM i WSIM oraz oscdimg dla Secure Boot SVN 9.0. Instalacja: rozpakuj, potem w CMD jako administrator:
  `for %i in (*.msp) do start /wait msiexec.exe /qn /p "%~fi"`
- `Install-WsoWorkstation.ps1` akceptuje każdą wersję 10.1.26100.x. Jeśli najpierw zainstalujesz 9457 ręcznie, skrypt go nie nadpisze.

### Windows 11 25H2 (system dla stacji)
**Obrazu ISO nie pobierasz.** OSDCloud w WinPE sam ściąga ESD z serwerów Microsoftu (`dl.delivery.mp.microsoft.com`).
Moduł OSD 26.10.6.1 ma w katalogu Windows 11 25H2 (build 26200), także w wersji pl-pl. Wystarczy wpis w profilu `deploy.json`:
```json
"OSName": "Windows 11 25H2 x64",
"OSEdition": "Enterprise",
"OSLanguage": "pl-pl",
"OSActivation": "Volume"
```
Od tej wersji przykładowa treść ma już 25H2 w obu profilach.

ISO 25H2 przyda się tylko do ręcznej VM wzorcowej. Publiczne strony Microsoftu (https://www.microsoft.com/software-download/windows11,
Evaluation Center) oferują już **26H2**. Obraz 25H2 pobierzesz z portalu licencji zbiorowych (Microsoft 365 admin center → Volume licensing)
albo z subskrypcji Visual Studio.

### Workspace ONE i AD
| Co | Skąd |
|---|---|
| Intelligent Hub `AirwatchAgent.msi` (x64) | **pobiera go `Initialize-WsoContent.ps1`** z https://packages.omnissa.com/wsone/AirwatchAgent.msi (ten sam plik co na getwsone.com) do `workdownloads`, sprawdza podpis Omnissa; własny plik: `-HubFile` |
| Adres Device Services (np. `ds1234.awmdm.com`) | UEM → Groups & Settings → All Settings → System → Advanced → Site URLs |
| Group ID | UEM, nazwa OG |
| Konto staging (login + hasło) | UEM → Accounts → Users, rejestracja staging tylko w tej OG; Zaawansowane → Staging: **Włącz wstępną rejestrację urządzenia = Wł.** (domyślnie wyłączone) i **Urządzenia jednego użytkownika = Standard** (użytkownik loguje się po stagingu; *Urządzenia wielu użytkowników* dotyczy platform mobilnych, nie trybu wielu użytkowników Windows) - inaczej urządzenia domenowe / Entra zostają na koncie staging (log Huba: *staging mode is not SingleUnknownUserStaging*) |
| Domena (FQDN) i OU komputerów | AD |
| Lista urządzeń: numer seryjny, numer inwentarzowy (opcjonalnie profil, OU) | `assets.csv` (format jak w `content\assets.csv`) |

### Z serwera WSODeployer (folder `server\`, instancja DEV)
- `secrets-dev\admin.key` – dla stanowiska administratora;
- `secrets-dev\agent-1.key` – dla agenta domenowego;
- `tls-dev\server.cer` – certyfikat TLS serwera (stanowisko, agent, nośnik);
- `secrets-dev\local-admin.txt` – użytkownik `admin` i jego hasło do konsoli webowej (skrypt wypisuje je też na końcu).

## 1. Serwer
`server\scripts\run-windows.ps1 [-Dev]` albo `run-linux.sh [--dev]`, opis w [server/README.md](../server.md). Z paczki
wydania skrypty pobierają wydane obrazy (`docker compose pull`), w klonie repozytorium je budują. Skrypt zawsze tworzy administratora konsoli
`admin` i na końcu wypisuje jego hasło. Zaloguj się nim na `https://<serwer>/console` i skonfiguruj logowanie
(Workspace ONE Access SAML / Entra ID) na stronie **Logowanie** - bez edycji `.env`.

REST API Workspace ONE UEM (opcjonalnie, v0.24): strona konsoli **Workspace ONE UEM** - klient OAuth dla środowiska, potem
import komputerów z UEM jako urządzeń; po każdym wdrożeniu serwer ustawia w UEM AssetNumber i tag Workspace ONE;
**Grupy z UEM** tworzy grupy urządzeń z grup organizacyjnych UEM (server/README.md).

## 2. Stanowisko administratora (PowerShell jako administrator)
```powershell
cd C:\WSODeployer
.\client\Install-WsoWorkstation.ps1 -ServerUri https://<serwer>:8443/ -ServerCertificate .\server\tls-dev\server.cer -AdminKeyFile .\server\secrets-dev\admin.key -CertificateDir .\work\cert
```
- Instaluje ADK i dodatek WinPE (albo wykrywa już zainstalowane), moduł OSD, certyfikat serwera do Trusted Root i zaszyfrowany klucz administratora.
- Pyta o **PIN do `deploy.pfx`** (minimum 8 znaków). Technicy wpisują go w WinPE.
- Tworzy `.\work\cert\deploy.cer` i `deploy.pfx`. Ustawienia zapisuje w `%APPDATA%\WSODeployer\workstation.json`.

## 3. Certyfikat wdrożeniowy na serwer
```powershell
.\server\scripts\run-windows.ps1 -Dev -DeploymentCertificate .\work\cert\deploy.cer
```
Bez `data-dev\cert\deploy.cer` stacje nie podpiszą żądań do serwera.

## 4. Treść wdrożeniowa
**Workspace ONE z konsoli** (najprościej): Konsola → **Workspace ONE** - serwer device services, Group ID, własność,
**konto staging** (tylko do zapisu, serwer od razu szyfruje je do certyfikatu wdrożeniowego i sejfu) i **Intelligent Hub**
(pobranie aktualnego z Omnissa albo wgranie MSI). Kreator poniżej jest wtedy potrzebny tylko dla aplikacji i `assets.csv`.

**Kilka środowisk Workspace ONE** (v0.22, np. tenant testowy i produkcyjny): ustawienia powyżej to środowisko domyślne;
*Kolejne środowiska Workspace ONE* dodaje następne, każde z własnym serwerem, Group ID, własnością i kontem staging
(`secrets/staging-<nazwa>.cms`), zapisane w deploy.json `Ws1Environments`. Profil wybiera swoje środowisko
(*Środowisko Workspace ONE*, puste = domyślne); Hub jest wspólny. Środowiska używanego przez profil nie można usunąć.
Stacja sprawdza środowisko swojego profilu przed wyczyszczeniem dysku.

```powershell
.\client\Initialize-WsoContent.ps1 -ContentRoot .\work\content
```
Treść trafia na serwer przez jego API (klucz administratora z `Install-WsoWorkstation.ps1`): wysyłane są tylko zmienione
pliki (`deploy.json`, `assets.csv`, `hub\*.msi`, `apps\*`, zaszyfrowane `secrets\*.cms`), `deploy.json` na końcu. Serwer
sprawdza SHA-256, przyjmuje sekrety tylko jako zaszyfrowany CMS i zostawia poprzedni `deploy.json` jako `deploy.json.bak-*`.
Samo wysłanie (np. po zmianie pliku w `apps\`): `.\client\WSODeployer.ps1 -Action ServerPublish -ContentRoot .\work\content`.
Z `-PublishTo <folder>` kreator kopiuje treść do folderu (serwer na tym samym komputerze, np.
`.\server\data-dev\content`).
Kreator pyta o:
- organizację i prefiks nazwy;
- metodę dołączania do domeny (`ODJ`), domenę i OU;
- serwer WS1, Group ID i własność CD/CS;
- Hub: pobranie aktualnego Huba x64 (`t`, około 300 MB, ponownie używany, dopóki aktualny) albo `-HubFile <msi>`;
- konto staging (odpowiedz `t`).

Na końcu liczy sumy SHA-256, sprawdza treść, wysyła ją na serwer i importuje `assets.csv`.

Przed kreatorem:
- wpisz swoje urządzenia do `work\content\assets.csv` (folder powstaje przy pierwszym uruchomieniu; możesz je przerwać i uruchomić ponownie);
- przykładowa aplikacja `VCRedist`: wrzuć `VC_redist.x64.exe` do `apps\` albo usuń jej wpis z `deploy.json`;
- profile (wersja Windows, język, strefa czasowa, aplikacje) edytujesz w `deploy.json` → `Profiles`.

## 5. Agent domenowy (serwer Windows w domenie, PowerShell jako administrator)
Skopiuj na ten serwer:
- foldery `agent\` i `engine\` z repozytorium;
- `deploy.cer`;
- `server.cer`;
- `agent-1.key`.

Potem uruchom:
```powershell
.\agent\Install-WsoAgent.ps1 -ServerUri https://<serwer>:8443/ -AgentKeyFile C:\temp\agent-1.key -DeploymentCertificate C:\temp\deploy.cer -ServerCertificate C:\temp\server.cer -DelegateOU "OU=Workstations,OU=Computers,DC=firma,DC=pl"
.\agent\Install-WsoAgent.ps1 -Action test
```
- Domyślnie zadanie działa jako SYSTEM, czyli konto komputera serwera, i to ono dostaje uprawnienia do OU.
  Opcja `-GmsaName` ustawia konto gMSA (wymaga klucza KDS root).
- Serwer musi mieć dostęp do portu 8443 serwera WSODeployer.
- Po instalacji usuń `C:\temp\agent-1.key`, bo agent ma własną zaszyfrowaną kopię.

## 6. Nośnik startowy (PowerShell jako administrator)
```powershell
.\client\New-WsoBootMedia.ps1 -OutFile .\work\media\WSODeployer.iso
```
Tworzy `work\media\WSODeployer.iso`. Z `-Usb` zapisuje też pendrive: OSDCloud pyta o dysk i **czyści go**.

## 6b. Kreator na stacji
WinPE uruchamia **graficzny kreator** (jak w MDT): powitanie + PIN → urządzenie (rozpoznanie / zatwierdzenie) → podsumowanie
(**Instaluj**) → postęp z logiem na żywo → zakończenie (restart za 15 s). Przed kliknięciem **Instaluj** nic nie jest zmieniane.
Konsola zamiast okna: `D:\WSODeployer\Start-WsoDeploy.ps1 -NoGui` (albo `"Gui": false` w `media.json`). Czarne okno konsoli za
kreatorem pokazuje to samo wyjście - nie zamykaj go.

## 6c. Profile (konsola WWW)
O tym, co zostanie zainstalowane, decyduje profil w konsoli: **Profile** → Nowy / edycja. Profil obejmuje:
- wersję Windows, edycję, język i aktywację;
- aplikacje;
- OU, grupę WS1 i własność;
- strefę czasową, klawiaturę i ustawienia regionalne;
- paczkę sterowników i Windows Update.

**Nowy profil = szablon** (v0.21). *Profile → Nowy profil* proponuje scenariusze Active Directory, Hybryda, Microsoft
Entra (użytkownik w OOBE / pakiet bulk) i Grupa robocza. Scenariusz startuje z profilu domyślnego, ustawia tożsamość
i OOBE, region według języka Windows i pokazuje, czego serwerowi jeszcze brakuje (domena w deploy.json, konektor online
i jego prawa w OU, SCP hybrid join, ważny pakiet Entra, gotowy Workspace ONE) z linkiem do konfiguracji.
*Zapisz jako szablon* przy profilu zachowuje go jako szablon firmy - bez hasła administratora i klucza VPN. Pola OU,
Group ID WS1, sprawdzenie VPN i nazwa administratora podpowiadają używane wartości (lista OU mówi, czy konektor może tam
tworzyć konta komputerów).
Profil wybiera rekord urządzenia (zatwierdzenie / assets.csv); puste = profil domyślny. Przy pierwszym starcie serwer
importuje profile z deploy.json; potem źródłem jest konsola.

## 6d. Kopia lokalna obrazu Windows (LAN)
Profile ze **Źródłem obrazu** Auto (domyślnie) biorą Windows z serwera WSODeployer, gdy ten ma kopię: pierwsze urządzenie
zgłasza ją serwerowi (i instaluje od Microsoftu), następne pobierają z LAN. Zawczasu, na stanowisku administratora:
`.\client\WSODeployer.ps1 -Action ServerImages`. Konsola → **Obrazy** pokazuje postęp i SHA-256. Serwer potrzebuje ok. 6 GB
na obraz i język w `server\data\os`.

## 6e. Maszyny wirtualne (VMware, Proxmox)
Urządzenie samo wykrywa platformę i instaluje aplikacje platformy z sekcji "Platforms" w deploy.json: VMware Tools na
VMware, narzędzia VirtIO + QEMU Guest Agent na Proxmox / KVM. `Initialize-WsoContent.ps1` je pobiera (sprawdza podpis / https).
Dla **Proxmox** zbuduj nośnik z `-VirtIO`: bez sterowników VirtIO WinPE nie zobaczy dysku VirtIO, a zainstalowany Windows
by z niego nie wystartował.

## 6f. Pakiety, Workspace ONE Drop Ship, lokalny administrator
Konsola → **Pakiety**: wgraj plik .ppkg. Dla pakietu Workspace ONE Drop Ship (Configuration Tool for Provisioning) wgraj
też jego unattend.xml - zostanie od razu zaszyfrowany. W profilu zaznacz pakiety. **Rejestracja w Workspace ONE** = pakiet
Drop Ship rejestruje przy pierwszym logowaniu; nazwa komputera i dołączenie do domeny dalej pochodzą z WSODeployer.
**Lokalny administrator**: włącz i wpisz hasło jeden raz; jest przechowywane zaszyfrowane i nie da się go już wyświetlić.

**Windows LAPS z własnym kontem klienta (v0.19)**: w profilu podaj **nazwę konta** (np. `lapsadmin`, do 20 znaków) i zaznacz
**losowe hasło**. SetupComplete tworzy konto (albo resetuje istniejące), dodaje je do grupy Administratorzy (po SID, w każdej
wersji językowej Windows) i ustawia losowe hasło, którego nigdzie nie zapisuje ani nie loguje. Zasada LAPS z Workspace ONE /
Intune (CSP `./Device/Vendor/MSFT/LAPS/Policies`) musi mieć tę samą nazwę w `AdministratorAccountName`; LAPS zmienia wtedy
hasło i przechowuje je w Entra ID albo AD. Pusta nazwa = wbudowane konto Administrator (SID ...-500). Dopóki zasada LAPS
się nie wykona, nikt nie zna hasła - profil WS1 przypisz do tej samej grupy smart co urządzenia.

## 6g. Grupy stacji
Konsola → **Grupy stacji**: grupa nadaje swoim stacjom profil, OU i grupę Workspace ONE (stacja > grupa > profil).
Reguły (producent, model, podsieć IP) umieszczają nowe stacje w grupie, gdy czekają na zatwierdzenie. Na stronie
**Urządzenia** zaznacz stacje i jednym krokiem przenieś je do grupy albo ustaw profil; assets.csv może mieć kolumnę **Group**.

## 6h. Rejestracja urządzeń zamiast deploy.pfx (v0.11)
Konsola → **Rejestracja urządzeń** → token → skopiuj **kod dołączenia** → `.\client\New-WsoBootMedia.ps1 -JoinCode 'WSOD1.…'`.
Nośnik nie ma deploy.pfx ani PIN-u; WinPE tworzy własny klucz i się rejestruje (nowa stacja czeka w Zatwierdzaniu).
Zapisz ponownie konto staging / lokalnego admina / pakiet WS1, aby trafiły do sejfu sekretów serwera. Szczegóły:
docs/pl/enrollment.md.

## 6i. Konektor domeny (v0.12)
Najprościej: Konsola → **Cloud Connectors** → **Konfigurator konektora** - krok po kroku (nowy konektor albo nowy kod dla
istniejącego, konto usługi: użytkownik domenowy / gMSA / LocalSystem, domena, OU, opcjonalnie kontroler domeny). Na końcu
rejestruje konektor i pokazuje gotowe polecenia PowerShell do każdego kroku (zaufanie do certyfikatu serwera, gMSA, prawa
w OU, instalacja MSI - albo tylko nowy kod na serwerze z usługą - i sprawdzenie), każde z przyciskiem Kopiuj.

Konsola → **Konektory domeny** → zarejestruj → skopiuj klucz. Na serwerze w domenie (jako administrator):
`.\agent\Install-WsoAgent.ps1 -ServerUri <serwer> -AgentKeyFile <plik klucza> -DelegateOU '<DN OU>'` - instaluje usługę
Windows WSODeployerConnector (ODJ + operacje na kontach komputerów). Szczegóły: docs/pl/connector.md.

## 6j. Tożsamość urządzenia: Entra ID / Hybrid (v0.15)
Konsola → **Profile** → *Tożsamość urządzenia i OOBE*: Grupa robocza, AD, Hybrid, Entra (użytkownik dołącza w OOBE) albo
Entra automatycznie. Dla trybu automatycznego zrób pakiet bulk w Windows Configuration Designer i wgraj go w **Pakietach**
jako *Pakiet Entra bulk (WCD)* z datą wygaśnięcia tokenu, potem zaznacz go w profilu. Hybrid wymaga Entra Connect Sync z
Hybrid join (konektor raportuje SCP). Przygotowanie labu krok po kroku: docs/pl/identity-design.md.
## 6k. Aplikacja technika (v0.16)
Konsola → **Pobieranie** → *WSODeployer.Technician.zip* → rozpakuj → **WSODeployer.Technician.exe** (prosi o uprawnienia
administratora). Zakładka 1 sprawdza / instaluje ADK, WinPE i OSD; zakładka 2 buduje nośnik z kodu dołączenia z konsoli
(albo samego adresu serwera dla kodów parowania). Szczegóły: docs/pl/technician.md.
## 6l. Aktualne obrazy Windows (v0.17)
Raz w miesiącu po Patch Tuesday, na stanowisku admina (jako administrator): `.\client\WSODeployer.ps1 -Action ServiceImage` -
najnowsza aktualizacja zbiorcza trafia do obrazu Windows każdego profilu, a serwisowany WIM na serwer; urządzenia instalują
go zamiast ESD Microsoftu. Szczegóły: docs/pl/images.md.
## 6m. Kontroler domeny przy pierwszym logowaniu (v0.20)
Wspierane: urządzenie jest w LAN albo łączy je **Workspace ONE Tunnel** - domyślne ustawienie profilu, nic nie trzeba
ustawiać. Tylko dla laboratorium albo miejsca bez WS1 Tunnel: opcja profilu *OpenVPN przed logowaniem* (MSI OpenVPN w "Apps",
jeden plik .ovpn z certyfikatami w środku, zapisany zaszyfrowany). Szczegóły: docs/pl/prelogon-vpn.md.
## 7. Pierwszy test – VM z Windows 11 25H2
- VM: UEFI + Secure Boot, vTPM 2.0, 4–8 GB RAM, dysk ≥ 64 GB, sieć z dostępem do internetu i do serwera WSODeployer.
- Na ESXi najprościej użyć karty **E1000e**. Sterownik vmxnet3 jest w WinPE tylko wtedy, gdy nośnik budowano z `-CloudDriver` obejmującym VMware (domyślne `*` go obejmuje).
- Numer seryjny VM (`Get-CimInstance Win32_BIOS`) albo MAC wpisz do `assets.csv`. Bez tego VM czeka na zatwierdzenie w konsoli.

Przebieg:
1. Uruchomienie z ISO, potem PIN do `deploy.pfx`.
2. Identyfikacja na serwerze, ewentualnie zatwierdzenie w konsoli (`/console` → Zatwierdzenia).
3. Agent tworzy konto komputera, a OSDCloud pobiera 25H2 i instaluje system.
4. Specialize (ODJ), SetupComplete (Hub, rejestracja WS1).
5. Logowanie kontem domenowym.

Przebieg śledzisz w konsoli WWW w zakładce **Wdrożenia**. Log na stacji: `C:\ProgramData\WSODeployer\`.
