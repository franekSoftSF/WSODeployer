# Kontroler domeny przy pierwszym logowaniu – LAN, Workspace ONE Tunnel albo OpenVPN (alternatywa)

Offline Domain Join robi z urządzenia członka domeny bez kontrolera domeny, ale **pierwsze logowanie kontem domenowym
wymaga DC**. Sposób połączenia z DC ustawia się w profilu (konsola: *Profile → Tożsamość i Workspace ONE → Kontroler
domeny przy pierwszym logowaniu*; kreator: pytanie po OU).

| Opcja | Kiedy | Co robi WSODeployer |
|---|---|---|
| **LAN albo Workspace ONE Tunnel** (domyślnie, wspierane) | urządzenie jest w sieci firmowej albo łączy je Workspace ONE Tunnel (profil WS1) | nic – nie instaluje VPN; urządzenie rejestruje się w WS1 w SetupComplete, zanim ktokolwiek się zaloguje, więc profile WS1 docierają na czas |
| **OpenVPN przed logowaniem** (alternatywa) | laboratorium, miejsca jeszcze bez Workspace ONE Tunnel | instaluje klienta OpenVPN z usługą, kładzie konfigurację do `config-auto`, usługa łączy się przy każdym starcie – przed ekranem logowania |

**Workspace ONE Tunnel przed logowaniem**: klient dla Windows (`VMwareTunnelInstaller.exe` z Omnissa Customer Connect -
nie ma publicznego pobierania; instalacja `/install /quiet /norestart`) może trafić do `content/apps` jako aplikacja
`WorkspaceOneTunnel` (kreator treści: `-TunnelFile`) i zostać zaznaczony w profilu - opcjonalnie, WS1 może dostarczyć
aplikację sam. Konfiguracja tunelu przychodzi z profilem WS1 Tunnel po rejestracji (SetupComplete, zanim ktokolwiek się zaloguje); jego
*Custom Configuration XML* musi mieć `<StartTunnelPreLogon>true</StartTunnelPreLogon>`, żeby tunel startował przed logowaniem.

Alternatywa nigdy nie zastępuje ścieżki domyślnej: profil bez niej (także każdy profil sprzed v0.20) nie instaluje VPN.

## Jak działa alternatywa

1. **WinPE, przed wyczyszczeniem dysku**: konfiguracja jest odszyfrowywana i sprawdzana – musi mieć `remote` i certyfikat
   klienta (`<cert>` + `<key>`, `<pkcs12>` albo `cryptoapicert`), nie może mieć `auth-user-pass` / `askpass` (przed
   logowaniem nikt nie wpisze hasła) ani osobnych plików (`ca ca.crt` ...: do `config-auto` trafia jeden plik). Zła
   konfiguracja zatrzymuje wdrożenie tutaj, stary system zostaje nietknięty. Serwer sprawdza to samo przy zapisie profilu
   (`VpnRules.CheckOpenVpn` = `Test-WsoOpenVpnConfig`, przypadki testowe w `tests/openvpn-cases.json`).
2. **PostApply**: MSI OpenVPN trafia do `ProgramData\WSODeployer\Apps`, konfiguracja do chronionego folderu `Secure`;
   w `deploy-state.json` jest tylko typ, aplikacja i sprawdzenie – nigdy konfiguracja.
3. **SetupComplete** (SYSTEM, po dołączeniu do Entra, przed pozostałymi aplikacjami): instalacja MSI, kopia konfiguracji
   do `config-auto` (ścieżka z `HKLM\SOFTWARE\OpenVPN\autostart_config_dir`, domyślnie `C:\Program Files\OpenVPN\config-auto`,
   tylko SYSTEM + Administratorzy) jako `wsodeployer.ovpn`, `OpenVPNService` ustawiona na start automatyczny i
   zrestartowana. Opcjonalne sprawdzenie (`host:port`, np. DC `10.10.0.10:389`) trafia tylko do logu – w LAN odpowiada też
   bez tunelu. Wynik: `SetupComplete.Results.vpn` w `deploy-state.json` i w dzienniku serwera.

## Konfiguracja

1. **MSI OpenVPN jako aplikacja** w `deploy.json` (`content/apps/`), z funkcją *OpenVPN Service*:

   ```json
   "OpenVPN": {
     "File": "apps/OpenVPN-2.7.7-I001-amd64.msi",
     "Sha256": "",
     "Arguments": "ADDLOCAL=OpenVPN.Service,OpenVPN,Drivers,Drivers.OvpnDco,Drivers.TAPWindows6",
     "SuccessCodes": [0, 3010]
   }
   ```

   Kreator treści (`client\Initialize-WsoContent.ps1`, krok *Klienci VPN*) pobiera to MSI z openvpn.net, sprawdza podpis
   OpenVPN Inc. i dodaje wpis. Nazwy funkcji pochodzą z MSI 2.7 (tabela Feature); inna wersja może mieć inne – jeśli
   SetupComplete zapisze „Usługa OpenVPNService nie jest zainstalowana”, porównaj jej tabelę Feature.
   `UpdateHash` uzupełnia `Sha256`.

2. **Jeden plik .ovpn ze wszystkim w środku.** Najprościej: wgraj w profilu ZIP z paczką klienta serwera VPN (Zentyal:
   *VPN → Servers → Download client bundle*, Windows) – konsola wstawia pliki certyfikatu / klucza do .ovpn w pamięci,
   dodaje `data-ciphers-fallback` dla stałego `cipher` i zapisuje wynik zaszyfrowany. Ręcznie:

   ```powershell
   $dir = 'C:\Temp\vpn-bundle'          # rozpakowana paczka
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

   Usuń `auth-user-pass`, jeśli paczka je ma (serwer musi przyjmować sam certyfikat). Starszy serwer (Zentyal 7:
   OpenVPN 2.4) z `cipher AES-256-CBC` wymaga w konfiguracji `data-ciphers-fallback AES-256-CBC` dla klienta 2.6.

3. **Profil**: *Kontroler domeny przy pierwszym logowaniu* → *OpenVPN przed logowaniem*, aplikacja, sprawdzenie
   (opcjonalne), wklejona konfiguracja. Zapisuje się zaszyfrowana CMS certyfikatem wdrożeniowym i sejfem (odczytają ją
   tylko urządzenia); przy ponownej edycji pole jest puste (puste = bez zmian). Powrót do *LAN albo Workspace ONE Tunnel*
   usuwa zapisaną konfigurację.

4. **Test**: urządzenie w sieci bez DC; po instalacji restart i logowanie kontem domenowym, które nigdy się na nim nie
   logowało – uda się tylko wtedy, gdy DC był osiągalny przed logowaniem. Logi: `C:\Program Files\OpenVPN\log\`,
   `ProgramData\WSODeployer\Logs\setupcomplete.log`. Jeśli ekran logowania jest szybszy niż tunel: GPO *Always wait for the
   network at computer startup and logon*.

## Ograniczenia

- **Jedna konfiguracja (jeden certyfikat i klucz klienta) dla N urządzeń – TYLKO test / lab, nie produkcja.** Serwer VPN
  nie odróżnia stacji, unieważnienie odcina wszystkie naraz, a klucz leży na każdym dysku (wyciek z jednej stacji = dostęp
  dla wszystkich). Na test: certyfikat tylko dla tego profilu (np. CN `wso-lab`) i `duplicate-cn` na serwerze.
  Produkcja: Workspace ONE Tunnel (tożsamość per urządzenie z UEM) albo certyfikat VPN per stacja – roadmapa
  (status.md): klucz tworzony na stacji (nieeksportowalny, `cryptoapicert`), certyfikat podpisany przez serwer WSODeployer,
  odwołanie przy wycofaniu urządzenia (CRL na serwerze VPN). AD CS sam nie wystarczy – po ODJ stacja nie ma jeszcze DC.
- Klucz zostaje na urządzeniu w `config-auto` (SYSTEM, Administratorzy i konto usługi OpenVPN – w 2.7
  `NT SERVICE\OpenVPNService`) – usługa potrzebuje go przy każdym starcie.
- Serwer VPN musi przekazać klientowi **DNS kontrolera domeny** (np. Zentyal: First nameserver = DC, Search domain = domena AD)
  i **trasę do sieci DC** – inaczej tunel działa, a logowanie domenowe kończy się „domena jest niedostępna”.
- Brak wykrywania sieci zaufanej: w LAN usługa też się łączy.
- Treść statyczna (bez serwera): `PreLogonVpn.ConfigCms` tworzy tylko konsola.
- Test na VM (2026-10-04, preprod): sprawdzenie w WinPE, instalacja MSI i `config-auto` działają; start usługi wymagał
  poprawki uprawnień (konto usługi 2.7). Logowanie domenowe przez tunel jeszcze nie sprawdzone (paczka z Zentyala miała klucz
  z innego certyfikatu).
