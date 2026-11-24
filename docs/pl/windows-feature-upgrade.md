# Aktualizacja funkcji Windows 11 (25H2 / 26H2) + najnowsze CU (Workspace ONE UEM)

`tools/Invoke-Win11FeatureUpgrade.ps1` podnosi każdą zarządzaną stację do wersji Windows 11 (`-TargetVersion 25H2`,
domyślnie, albo `26H2`), a potem instaluje najnowszą aktualizację zbiorczą (CU). Nie zależy od silnika WSODeployer
(bez modułu, bez serwera).

## Jak działa

Pierwsze uruchomienie (skrypt WS1, SYSTEM) kopiuje skrypt do `%ProgramData%\WSODeployer\WinUpgrade`, zapisuje
parametry w `config.json`, rejestruje zadanie **WSODeployer Windows Upgrade** (SYSTEM, przy starcie + co 4 godziny)
i je uruchamia - kończy się po kilku sekundach. Zadanie działa dalej przez kolejne restarty (T = wersja docelowa):

| Stacja | Krok |
|---|---|
| build bazowy T, UBR poniżej wymogu | CU z Windows Update (wymóg pakietu aktywacyjnego) → restart |
| build bazowy T, wymóg spełniony | „Windows 11, version T” (pakiet aktywacyjny, jeden restart) z Windows Update |
| brak oferty albo starsza (23H2 / 22H2 / Windows 10) | pełny in-place upgrade: ISO wersji T pobrane na stację (`WSO_UPG_MEDIAURI`), `-MediaPath` (ISO / folder, lokalnie lub UNC) albo Windows 11 Installation Assistant (tylko gdy T to najnowsze wydanie) |
| build T | CU z Windows Update → restart → ponowny skan, aż nic nie jest oferowane (i UBR ≥ `-MinimumUbr`) → **Done**, zadanie usunięte |
| nowsza niż T | **Done** - nigdy downgrade |

| T | Build | Pakiet aktywacyjny | Buildy bazowe i wymóg |
|---|---|---|---|
| 25H2 | 26200 | [KB5054156](https://support.microsoft.com/help/5054156) | 24H2 ≥ 26100.5074 |
| 26H2 | 26300 | [KB5121794](https://support.microsoft.com/help/5121794) | 24H2 ≥ 26100.9546, 25H2 ≥ 26200.9546 (KB5124010) |

Pakietów aktywacyjnych nie ma w Microsoft Update Catalog - oferuje je tylko Windows Update / WSUS (klasyfikacja
Upgrades). **Installation Assistant zawsze instaluje najnowsze wydanie** (dziś 26H2), więc przy `-TargetVersion 25H2`
pełny upgrade wymaga `-MediaPath` z nośnikiem 25H2 - skrypt nie przeskoczy wersji docelowej.
Każda faza ma limit prób (`-MaxAttempts`, domyślnie 3); potem stan **Failed** i zadanie jest usuwane.

### ISO na stacji (pełny upgrade)

Z `WSO_UPG_MEDIAURI` stacja sama pobiera ISO - tylko gdy pełny upgrade jest naprawdę potrzebny (stacje 24H2 z pakietem
aktywacyjnym nigdy go nie pobierają):

- **Gdzie**: `C:\Windows\Temp\WsoWinUpgrade\Win11_<T>.iso` (`WSO_UPG_MEDIACACHE`); folder ukryty + systemowy, dostęp
  tylko SYSTEM i Administratorzy. Skrypt usuwa tylko swoje pliki (`Win11_*.iso`, niedokończone pobieranie) po
  **Done** albo przy zmianie celu; po **Failed** ISO zostaje na ponowną próbę.
- **Jak**: BITS - wznawia po restarcie / przy następnym uruchomieniu zadania (co 4 h), tylko w sieciach bez limitu
  (bez LTE / hotspota), maks. 5 h na jedno uruchomienie. Bez BITS: zwykłe pobieranie HTTPS. Wymaga 35 GB wolnego na C:.
- **Kontrola**: `WSO_UPG_MEDIASHA256` (zalecane) - po pobraniu i przed każdym ponownym użyciem; niezgodność usuwa
  plik. Potem sprawdzane są też setup.exe (podpis Microsoft) i build obrazu (musi być T).
- **Źródło**: adres HTTPS hostowany u was - linki Microsoftu do ISO wygasają po 24 h. Laptopy poza siecią potrzebują
  adresu dostępnego z internetu: własny serwer WWW albo blob storage z tokenem SAS tylko do odczytu (Azure Blob / S3).
  Token jest tylko w zmiennej WS1 i w `config.json` (folder tylko dla SYSTEM / Administratorów); w logu adres bez
  query stringu. SHA-256 ISO: `Get-FileHash .\Win11_25H2_Polish_x64.iso`.
- ISO w języku stacji (i z edycją stacji: ISO wieloedycyjne, Pro / Enterprise) - setup zachowa aplikacje i dane tylko
  przy tym samym języku.

### Przejście na kolejną wersję (25H2 → 26H2)

1. Profil WS1 Windows Updates: `TargetReleaseVersion` = `26H2` (inaczej stacja nigdy nie dostanie oferty 26H2).
2. Zmienna skryptu WS1 `WSO_UPG_TARGETVERSION` = `26H2`, uruchom skrypt ponownie: zmiana celu sama czyści stan
   (także po Done).
3. Stacje 25H2 dostają pakiet aktywacyjny 26H2 (jeden restart), stacje 24H2 też (bez 25H2 po drodze).

Kolejne wydanie w przyszłości: dodaj wiersz do `$script:Releases` w skrypcie (build, KB, buildy bazowe z minimalnym
UBR z artykułu KB), wartość do `ValidateSet` parametru `-TargetVersion` i przenieś `Latest`.

## Profil WS1 „Windows Updates”

Profil (Policy CSP `Update/*`) decyduje, co oferuje Windows Update. Skrypt nigdy go nie nadpisuje:

- **Wersja docelowa**: w profilu *Windows 11* / T. Gdy profil wskazuje inną wersję, skrypt loguje ostrzeżenie
  i pomija pakiet aktywacyjny (zamiast tego pełny upgrade). Bez wersji docelowej w profilu skrypt sam ustawia
  politykę rejestru `TargetReleaseVersion` = T (`-NoPolicy` to wyłącza).
- **Odroczenie / wstrzymanie aktualizacji funkcji**: 0 dni, bez pauzy - inaczej T nie zostanie zaoferowane.
- **Odroczenie / wstrzymanie aktualizacji jakościowych**: CU pojawi się dopiero po odroczeniu; dla „najnowsze CU
  od razu” ustaw 0 dni (albo krótkie odroczenie i `-MinimumUbr`).
- Skan idzie do Windows Update online (`-UpdateSource WindowsUpdate`, domyślnie) nawet przy WSUS;
  `-UpdateSource Default` używa skonfigurowanego źródła.
- Safeguard holds: Microsoft może blokować wydanie na części sprzętu; wtedy rusza pełny upgrade (setup.exe
  `/compat ignorewarning` pomija tylko ostrzeżenia - twarde blokady nadal go zatrzymują, patrz `setup-logs`).

### Zalecane ustawienia profilu

Policy CSP `./Device/Vendor/MSFT/Policy/Config/Update/...` ([dokumentacja](https://learn.microsoft.com/en-us/windows/client-management/mdm/policy-csp-update)):

| Ustawienie | Wartość | Dlaczego |
|---|---|---|
| `ProductVersion` / `TargetReleaseVersion` | `Windows 11` / T | oferowane T, nic nowszego; skrypt nie zapisuje wtedy polityki w rejestrze |
| `BranchReadinessLevel` | 16 | kanał General Availability |
| `ManagePreviewBuilds` | 0 | bez buildów Insider |
| `DeferFeatureUpdatesPeriodInDays`, `DeferQualityUpdatesPeriodInDays`, `Pause*` | nieustawione (0) | T i najnowsze CU oferowane od razu |
| `ConfigureDeadlineForFeatureUpdates` | 5 (na czas wdrożenia: 2-5) | dni do wymuszonej instalacji |
| `ConfigureDeadlineForQualityUpdates` | 2-5 | |
| `ConfigureDeadlineGracePeriod` | 1-2 | dni od instalacji do wymuszonego restartu (jakościowe; także funkcji, gdy brak `ConfigureDeadlineGracePeriodForFeatureUpdates`) |
| `ConfigureDeadlineNoAutoReboot` | **0** | 1 = brak automatycznego restartu poza godzinami aktywności przed terminem + karencją (do 7 dni czekania na restart) |
| `SetDisablePauseUXAccess` | **1** | użytkownik nie wstrzyma aktualizacji w Ustawieniach |
| `AllowAutoWindowsUpdateDownloadOverMeteredNetwork` | **0** | bez pobierania CU / upgrade'u przez LTE / hotspot |
| `IgnoreMOUpdateDownloadLimit`, `IgnoreMOAppDownloadLimit` | nieustawione | legacy, nie działają w Windows 11 |
| `AllowAutoUpdate` | 1 albo nieustawione | przy terminach ignorowane (pobieranie / instalacja / restart) |
| `DisableWUfBSafeguards` | nieustawione (0) | zostaw safeguard holds; tam gdzie blokują, skrypt robi pełny upgrade |

Poza profilem Update: Delivery Optimization (`./Device/Vendor/MSFT/Policy/Config/DeliveryOptimization/DODownloadMode`
= 1 LAN albo 2 grupa), żeby CU pobierało się z internetu raz na lokalizację.

Fragment Custom Settings dla dwóch zmian, których najczęściej brakuje:

```xml
<Replace><CmdID>1</CmdID><Item><Target><LocURI>./Device/Vendor/MSFT/Policy/Config/Update/ConfigureDeadlineNoAutoReboot</LocURI></Target><Meta><Format xmlns="syncml:metinf">int</Format></Meta><Data>0</Data></Item></Replace>
<Replace><CmdID>2</CmdID><Item><Target><LocURI>./Device/Vendor/MSFT/Policy/Config/Update/SetDisablePauseUXAccess</LocURI></Target><Meta><Format xmlns="syncml:metinf">int</Format></Meta><Data>1</Data></Item></Replace>
```

## Wdrożenie w WS1 UEM

1. **Resources → Scripts → Add → Windows**: język PowerShell, kontekst **System**, timeout 300 s, treść =
   `tools/Invoke-Win11FeatureUpgrade.ps1` (bez zmian, bez parametrów).
2. **Variables** skryptu (WS1 nie przekazuje parametrów - każdą zmienną ustawia jako zmienną środowiskową, skrypt
   czyta `WSO_UPG_<nazwa parametru>`), np.:

   | Klucz | Wartość |
   |---|---|
   | `WSO_UPG_TARGETVERSION` | `25H2` (później `26H2`) |
   | `WSO_UPG_RESTARTDELAYMINUTES` | `120` |
   | `WSO_UPG_MEDIAURI` | `https://pliki.example.com/win11/Win11_25H2_x64.iso` (do pełnego upgrade'u, patrz wyżej) |
   | `WSO_UPG_MEDIASHA256` | SHA-256 tego ISO |

   Przełączniki przyjmują `true` / `false` (`1` / `0`), np. `WSO_UPG_NORESTART` = `true`. Bez zmiennej obowiązuje
   wartość domyślna. Błędna wartość kończy skrypt błędem (exit 1 = *Failed* w WS1, szczegóły w `upgrade.log`).
   Przejście na 26H2 = zmiana `WSO_UPG_TARGETVERSION` i ponowne uruchomienie skryptu.
3. Przypisz do smart group (najpierw pilotaż), wyzwalacz *On demand* albo *Scheduled*.
4. Opcjonalny sensor (PowerShell, System, String) do raportowania:

   ```powershell
   $p = Get-ItemProperty 'HKLM:\SOFTWARE\WSODeployer\WinUpgrade' -ErrorAction SilentlyContinue
   if ($p) { '{0} {1} {2}.{3} {4}' -f $p.TargetVersion, $p.State, $p.Build, $p.Ubr, $p.Detail } else { 'NotStarted' }
   ```

## Ustawienia

Zmienna WS1 (zmienna środowiskowa) albo, przy ręcznym uruchomieniu, parametr o tej samej nazwie (parametr wygrywa).

| Zmienna WS1 | Parametr | Domyślnie | |
|---|---|---|---|
| `WSO_UPG_TARGETVERSION` | `-TargetVersion` | `25H2` | `25H2` albo `26H2` |
| `WSO_UPG_MEDIAURI` | `-MediaUri` | – | adres HTTPS ISO wersji T, pobierane do lokalnego cache (ma pierwszeństwo przed MediaPath) |
| `WSO_UPG_MEDIASHA256` | `-MediaSha256` | – | SHA-256 ISO (64 znaki hex) |
| `WSO_UPG_MEDIACACHE` | `-MediaCache` | `C:\Windows\Temp\WsoWinUpgrade` | ukryty lokalny folder na ISO |
| `WSO_UPG_MEDIAPATH` | `-MediaPath` | – | `.iso` albo folder z `setup.exe` wersji T, lokalnie lub UNC (ISO z UNC jest kopiowane do cache) |
| `WSO_UPG_NOINSTALLATIONASSISTANT` | `-NoInstallationAssistant` | false | nigdy nie pobieraj Installation Assistant |
| `WSO_UPG_QUALITYSCOPE` | `-QualityScope` | `Cumulative` | `All` = każda aktualizacja software bez previews (nigdy sterowniki) |
| `WSO_UPG_UPDATESOURCE` | `-UpdateSource` | `WindowsUpdate` | `Default` = WSUS / skonfigurowane źródło |
| `WSO_UPG_RESTARTDELAYMINUTES` | `-RestartDelayMinutes` | 60 | zapowiedziany restart, gdy ktoś jest zalogowany; bez użytkownika: od razu |
| `WSO_UPG_NORESTART` | `-NoRestart` | false | nigdy nie restartuj - kontynuacja po następnym restarcie |
| `WSO_UPG_RESTARTMESSAGE` | `-RestartMessage` | tekst EN | komunikat przy restarcie (można podać po polsku) |
| `WSO_UPG_MINIMUMUBR` | `-MinimumUbr` | 0 | Done dopiero od <build T>.<UBR> |
| `WSO_UPG_MAXATTEMPTS` | `-MaxAttempts` | 3 | na fazę |
| `WSO_UPG_NOPOLICY` | `-NoPolicy` | false | nigdy nie zapisuj polityki rejestru TargetReleaseVersion |
| `WSO_UPG_RESET` | `-Reset` | false | wyczyść stan (także po Done / Failed) - potem usuń zmienną |

Stan: `HKLM\SOFTWARE\WSODeployer\WinUpgrade` (`TargetVersion`, `State` = Scheduled / Running / WaitingForRestart /
Done / Failed, `Detail`, `Build`, `Ubr`, `LastRun`); log `%ProgramData%\WSODeployer\WinUpgrade\upgrade.log`.

## Testy

Tylko statycznie (parser, PSScriptAnalyzer, filtry aktualizacji na przykładowych danych). Nie testowane na stacji:
skan / instalacja Windows Update, pakiety aktywacyjne, setup.exe i Installation Assistant. Najpierw VM / grupa
pilotażowa.
