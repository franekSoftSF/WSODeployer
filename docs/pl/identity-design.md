# Tożsamość urządzenia: Entra ID, Hybrid i Workspace ONE – projekt (v0.15, lab)

Cel (2026-10-03): **każde** wdrożenie kończy się urządzeniem **zarządzanym przez Workspace ONE**, w jednym z trzech
modeli tożsamości, wybieranym w profilu:

| Tryb profilu | Tożsamość | Kto dołącza | OOBE | Logowanie |
|---|---|---|---|---|
| **Entra – OOBE** | Microsoft Entra joined | **użytkownik** w klasycznym OOBE („Skonfiguruj do pracy lub nauki”) | klasyczne | konto służbowe Entra |
| **Entra – bez użytkownika (bulk)** | Microsoft Entra joined | **urządzenie samo** (token bulk z WCD), bez dotykania stacji | klasyczne albo skrócone | konto służbowe Entra na ekranie logowania |
| **Hybrid** | AD DS + Microsoft Entra hybrid joined | WSODeployer (ODJ w WinPE) + Entra Connect (rejestracja urządzenia) | klasyczne albo skrócone | konto domenowe (UPN = konto Entra) |

Workspace ONE: w każdym trybie rejestracja przez **Intelligent Hub z kontem staging w SetupComplete** (przed OOBE – działa
już dziś, sprawdzone na VM) albo – dla „Entra – OOBE” z licencją Entra ID P1 – **automatyczna rejestracja MDM** przy
dołączeniu do Entra (pakiet Drop Ship „Premium”: Hub z `ENROLL=N`).

## 1. Jak to działa – przebieg

### Entra – bez użytkownika (bulk) – pełna automatyzacja
1. **WinPE** (bez zmian): rejestracja stacji (kod dołączenia), nazwa z numeru inwentarzowego, Windows 11 z OSDCloud.
2. **Jeszcze w WinPE**: **pakiet WCD „Enroll in Azure AD”** (token bulk) jest odszyfrowany, sprawdzony i zapisany do
   chronionego folderu `Secure` instalowanego Windows – **nie** jest nakładany offline: pakiet dodany offline Windows
   przetwarza w specialize (`provtool /Turn 1`), gdzie dołączenie do Entra się nie kończyło i instalacja wisiała na
   „Przygotowywanie” (VM w labie, 2026-10-03, poprawione w 0.16.1). Opcjonalny pakiet Drop Ship WS1 nadal idzie offline (DISM).
   Nazwa komputera, region i hasło lokalnego administratora pochodzą z WSODeployer (unattend), nie z pakietów.
3. **SetupComplete (SYSTEM, przed OOBE)**: najpierw **dołączenie do Entra** – czekanie na login.microsoftonline.com,
   synchronizacja czasu, `Install-ProvisioningPackage -QuietInstall` w osobnym procesie z limitem 10 minut, sprawdzenie
   `dsregcmd` (błąd trafia do raportu, instalacja idzie dalej); potem aplikacje bazowe, **rejestracja w WS1 kontem staging**
   (urządzenie jest w WS1, zanim ktokolwiek się zaloguje – profile Wi-Fi / VPN / certyfikaty przychodzą wcześnie), raport
   do konsoli, usunięcie `Secure`.
4. **OOBE**: urządzenie jest już dołączone. Konto „package_…” z Entra nie loguje się – służy tylko do dołączenia.
5. **Ekran logowania**: użytkownik loguje się kontem służbowym Entra; WS1 przypisuje urządzenie zalogowanemu użytkownikowi
   (`ASSIGNTOLOGGEDINUSER`).

### Entra – OOBE (klasyczne, konto służbowe)
1.–3. jak wyżej, ale **bez pakietu bulk**; OOBE **nie jest ukrywane** (WSODeployer nie pomija stron konta).
4. **OOBE**: użytkownik wybiera „Skonfiguruj do pracy lub nauki” i loguje się kontem służbowym → Entra join.
5. WS1: albo Hub staging z SetupComplete (urządzenie już w WS1, przypisanie do użytkownika przy logowaniu), albo – z Entra
   ID P1 – **automatyczna rejestracja MDM do WS1** zaraz po dołączeniu (pakiet Drop Ship „Premium”, który dziś działa).
   Nigdy oba naraz (profil, *Rejestracja w Workspace ONE*: **Microsoft Entra** = automatycznie, bez Huba - domyślnie w szablonie
   Entra - OOBE od 0.22.6 - albo Hub z kontem staging): przy Hubie użytkownicy z OOBE nie mogą być w zakresie użytkownika MDM **żadnej** aplikacji MDM w Entra
   (Mobilność - sprawdź każdą, np. „AirWatch by VMware” i „Workspace ONE”); inaczej dołączenie w OOBE próbuje zarejestrować
   urządzenie drugi raz i kończy się błędem **8018000a** (już zarejestrowane) - VM na preprod 2026-10-10.
   Przy automatycznej rejestracji **reinstalacja** urządzenia wymaga najpierw usunięcia jego starego rekordu w UEM (ten sam
   numer seryjny: UEM też odpowiada 8018000a); Hub z kontem staging używa tego rekordu ponownie.

### Hybrid (AD + Entra)
1.–2. WinPE: jak dziś – **konektor** tworzy konto komputera (`djoin /provision /reuse`) w OU z profilu / grupy, blob ODJ
   trafia do unattend (szyfrowany do stacji).
3. SetupComplete: rejestracja w WS1 kontem staging.
4. Pierwszy start: komputer jest w domenie. **Entra Connect Sync** synchronizuje obiekt komputera (OU w zakresie
   synchronizacji), Windows rejestruje się w Entra przez zadanie „Automatic-Device-Join” (SCP w AD wskazuje tenant) –
   stan „Microsoft Entra hybrid joined” pojawia się po następnym cyklu synchronizacji (domyślnie do ~30 min).
5. Logowanie kontem domenowym (UPN = konto Entra) – pierwsze logowanie wymaga łączności z DC (w sieci firmowej albo VPN
   przed logowaniem).

## 2. Co trzeba przygotować w labie

### Entra ID (tenant)
- **Urządzenia → Ustawienia urządzeń**:
  - „Użytkownicy mogą dołączać urządzenia do Microsoft Entra” = Wszyscy (albo grupa, do której trafi konto „package_…”
    z WCD);
  - „Wymagaj MFA do rejestrowania lub dołączania urządzeń” = **Nie** (token bulk nie przejdzie MFA) – albo w Dostępie
    warunkowym wykluczyć konto „package_…” z polityki dla akcji „Zarejestruj lub dołącz urządzenia”;
  - limit urządzeń na użytkownika – sprawdzić, czy nie zablokuje konta „package_…” (lab: podnieść).
- **Mobilność (MDM i WIP) → Workspace ONE (AirWatch)**: *Zakres użytkownika MDM* = **Niektóre** (grupa prawdziwych
  użytkowników), **nigdy Wszystkie** i nigdy grupa z kontem „package_…”. Przy *Wszystkie* pakiet bulk rejestruje też urządzenie
  w WS1 jako „package_…”, WS1 nie zna tego użytkownika i Windows wycofuje całe dołączenie: `0x801900C8` w `AADJ/BPRT`
  (sprawdzone w labie). WSODeployer sam rejestruje w WS1 (Hub + staging); tryb Entra – OOBE potrzebuje zakresu dla
  prawdziwych użytkowników.
  - Zakres nie ma wykluczeń: sprawdź grupy konta „package_…” (Użytkownicy → package_… → Grupy). Grupa **dynamiczna**
    dostaje `-and -not (user.userPrincipalName -startsWith "package_")` - to obejmuje też każdy przyszły pakiet.
  - Przełącznik *Wyłącz rejestrację w MDM podczas dodawania konta służbowego* nie pomaga: dotyczy urządzeń
    *zarejestrowanych* w Entra (Ustawienia → Konta), a nie dołączenia pakietem bulk ani w OOBE.
  - Objaw: w audycie Entra *Register device* przez package_… i kilka sekund później *Unregister device*; silnik (0.22.3+)
    zapisuje to jako problem zakresu MDM, a jego zadanie ponawiające dołączy urządzenie po poprawce.
  - Ponowne ręczne nałożenie pakietu na stacji kończy się `0x800700B7` (to samo ID i wersja pakietu): wyeksportuj ten sam
    projekt WCD z wyższą wersją (np. 1.2 → 1.3); token i konto package_ zostają.
- **Hybrid**: Entra Connect **Sync** (Cloud Sync nie synchronizuje urządzeń) z opcją *Configure device options →
  Hybrid Microsoft Entra join* (tworzy SCP w AD), OU stacji w zakresie synchronizacji.
- **Opcjonalnie (Entra – OOBE z P1)**: Mobilność (MDM i WIP) → aplikacja **Workspace ONE UEM** z adresami MDM z konsoli
  WS1 (Ustawienia → Windows → integracja z Entra), zakres użytkowników MDM.

### Hybrid join w Entra Connect Sync – krok po kroku
W labie jest Entra Connect bez Hybrid; to go włącza (bez zmian dla już synchronizowanych użytkowników):
1. Na serwerze Entra Connect: **Microsoft Entra Connect** → *Configure* → **Configure device options** → logowanie
   Global Administratorem (albo Hybrid Identity Administratorem) tenanta.
2. *Device options*: **Configure Hybrid Microsoft Entra join**.
3. *Device operating systems*: **Windows 10 or later domain-joined devices**.
4. *SCP configuration*: zaznacz las, *Authentication Service* = **Microsoft Entra ID**, podaj Enterprise Administratora
   lasu → *Configure*. Powstaje
   `CN=62a0ff2e-97b9-4513-943f-0d221bd30080,CN=Device Registration Configuration,CN=Services,<configuration NC>` z
   `azureADName:<tenant>` i `azureADId:<id tenanta>` (bez uprawnień Enterprise Admin kreator daje skrypt PowerShell dla
   administratora lasu).
5. **Zakres synchronizacji**: Entra Connect → *Customize synchronization options* → *Domain and OU filtering* – OU
   profili / grup WSODeployer muszą być zaznaczone (obiekty komputerów spoza zakresu nigdy nie trafią do Entra).
6. Sprawdzenie: strona **Konektory domeny** pokazuje przy konektorze *SCP Hybrid join: <tenant>*, a edytor profilu po
   wyborze Hybrid – „konektor znalazł SCP”. Na wdrożonej stacji: `dsregcmd /status` → `AzureAdJoined : YES`,
   `DomainJoined : YES` (po kolejnym cyklu synchronizacji; `Start-ADSyncSyncCycle -PolicyType Delta` na serwerze Entra
   Connect przyspiesza).
7. Uwaga: polityka Dostępu warunkowego wymagająca urządzenia „hybrid joined” nie może obejmować pierwszego logowania na
   nowej stacji (stan pojawia się dopiero po synchronizacji).

### Workspace ONE z Entra P1 (użytkownik dołącza w OOBE)
- Konsola WS1 → *Groups & Settings → All Settings → Devices & Users → Windows → Windows Desktop → Directory Services /
  integracja z Microsoft Entra ID*: włącz, skopiuj **MDM enrollment URL**, **MDM terms of use URL** i **MDM discovery
  URL**.
- Entra → *Mobilność (MDM i WIP)* → **Workspace ONE UEM** (dodaj z galerii, jeśli brak): wklej trzy adresy, *Zakres
  użytkowników MDM* = Wszyscy (albo grupa użytkowników / pracowników).
- Wtedy użytkownik dołączający do Entra w OOBE jest rejestrowany w WS1 przez sam Windows; Hub z profilu (`ENROLL=N`, XML
  Premium) tylko dokłada agenta. Bez P1 zostaje rejestracja kontem staging w SetupComplete (`ENROLL=Y`).

### Sieć dla dołączenia do Entra
- Urządzenia muszą rozwiązywać i osiągać bezpośrednio `login.microsoftonline.com`, `enterpriseregistration.windows.net` i
  `device.login.microsoftonline.com`: bez przechwytywania DNS i bez inspekcji TLS przez agenta VPN / ZTNA czy proxy.
  Lab 2026-10-04: GlobalProtect na hoście odpowiadał na `login.microsoftonline.com` adresem `100.64.1.1` (nawet dla 8.8.8.8),
  VM za NAT VMware dziedziczyła to - błąd dołączenia `0xCAA70004` *No connectivity*. Pomaga sieć Bridged albo wyłączony
  GlobalProtect; zadanie ponawiające dołącza wtedy bez nowej instalacji. Log SetupComplete pokazuje odpowiedź DNS.

### Windows Configuration Designer – pakiet bulk (raz na ≤180 dni)
1. WCD (dodatek ADK „Configuration Designer” albo Microsoft Store) → **Provision desktop devices**.
2. *Set up device*: **nazwa urządzenia – zostaw puste** (nazwę nadaje WSODeployer), bez usuwania aplikacji.
3. *Set up network*: pomiń (lab: przewód) albo profil Wi-Fi, jeśli OOBE ma iść po Wi-Fi.
4. *Account management*: **Enroll in Azure AD**, *Bulk Token Expiry* (najwyżej 180 dni) → **Get Bulk Token** (logowanie
   kontem z prawem dołączania urządzeń – w Entra powstaje konto „package_<guid>”).
5. *Add applications / certificates*: nic (to robią WS1 i WSODeployer); **lokalny administrator: nie** (robi to profil).
6. *Finish* → **Create** bez szyfrowania pakietu (zaszyfrowany wymaga hasła przy nakładaniu – offline nie zadziała).
7. Pakiet `.ppkg` (jest w nim token – to **hasło do dołączania urządzeń do tenanta**) → `work\` → konsola (patrz §4).

### Workspace ONE
- OG i konto staging jak dziś (np. `corp_windows`, konto staging) – używane w SetupComplete.
- Lab Hybrid: profil z domeną = AD, OU w zakresie Entra Connect.

### WSODeployer
- Konektor domeny (Hybrid) – MSI z konsoli, konto z delegacją; pakiety w konsoli; profile.

## 3. Co zbuduję w WSODeployer (v0.15)

1. **Profil → „Tożsamość urządzenia”**: Grupa robocza / AD (ODJ) / **Hybrid (AD + Entra)** / **Entra – OOBE** /
   **Entra – bulk**. Domena wynika z trybu (AD / Hybrid = ODJ, Entra = brak); walidacja:
   - Entra – bulk wymaga pakietu typu **„Entra bulk”**;
   - Entra – OOBE nie może ukrywać stron konta w OOBE;
   - Hybrid wymaga konektora i OU.
2. **Profil → „OOBE”**: **Klasyczne** (strony prywatności, konto służbowe – Windows pyta) albo **Skrócone** (jak dziś).
   Region i nazwa komputera zawsze z WSODeployer.
3. **Pakiet „Entra bulk”** (konsola → Pakiety): typ pakietu, data wygaśnięcia tokenu (konsola ostrzega 30 dni wcześniej),
   nazwa konta „package_…”. **Pakiet z tokenem nie jest publiczny**: nie trafia do `/content/packages` (dostępnego bez
   logowania), tylko jest szyfrowany do sejfu serwera i wydawany **wyłącznie zarejestrowanej stacji** przez podpisane
   żądanie (jak blob ODJ) – kopia ze sterownika USB czy z internetu nie wystarczy, żeby dołączać urządzenia do tenanta.
4. **Rejestracja WS1 w każdym trybie**: Hub staging w SetupComplete (domyślnie), pakiet Drop Ship albo „MDM z Entra (P1)”
   – kreator WinPE pokazuje w podsumowaniu: tożsamość + sposób rejestracji w WS1.
5. **Konektor – Hybrid**: w sygnale stanu sprawdza **SCP Hybrid Join** w AD (tenant, domena) – konsola pokazuje, czy las
   jest gotowy na Hybrid; ostrzega, gdy OU profilu nie ma pod SCP-skonfigurowanym lasem.
6. **Później (faza 2, konektor + Microsoft Graph)**: stan urządzenia w Entra w konsoli (joined / hybrid pending /
   hybrid joined), usuwanie starego rekordu Entra przy reinstalacji i wycofaniu, grupy Entra; Autopilot dla klientów
   z Intune.

## 4. Bezpieczeństwo
- Token bulk = prawo dołączania urządzeń do tenanta do daty wygaśnięcia. Przechowywany zaszyfrowany (sejf), wydawany
  tylko zarejestrowanej stacji, nigdy w publicznej treści; w razie wycieku: **zablokuj / usuń konto „package_…” w Entra**.
- Hasło lokalnego administratora z profilu (nie z pakietów WS1 – ich `AdministratorPassword` jest tylko kodowany).
- Pakiety WS1 Drop Ship: unattend.xml szyfrowany przy wgraniu (konto staging), jak dziś.

## 5. Plan testów w labie

| # | Scenariusz | Oczekiwany wynik |
|---|---|---|
| 1 | Entra – bulk, OOBE skrócone, Hub staging | bez dotykania stacji: Entra joined (`dsregcmd /status`: AzureAdJoined YES), w WS1 przed logowaniem, logowanie kontem Entra, urządzenie przypisane do użytkownika |
| 2 | Entra – bulk, OOBE klasyczne | jak 1, OOBE pokazuje strony Windows, ale nie pyta o konto (urządzenie już dołączone) – **do potwierdzenia** |
| 3 | Entra – OOBE, Hub staging | użytkownik dołącza w OOBE, urządzenie już w WS1, przypisanie przy logowaniu |
| 4 | Entra – OOBE, pakiet Drop Ship Premium (P1) | dołączenie w OOBE → automatyczna rejestracja MDM do WS1 |
| 5 | Hybrid, sieć firmowa | konto komputera w OU (konektor), domena po pierwszym starcie, hybrid joined po synchronizacji Entra Connect, w WS1 |
| 6 | Reinstalacja tej samej stacji (1 i 5) | ta sama nazwa / konto AD (`/reuse`) / rekord WS1; Entra: stary rekord do sprzątnięcia (faza 2) |
| 7 | Token bulk wygasły / konto „package_…” zablokowane | czytelny błąd w raporcie wdrożenia, stacja zostaje w grupie roboczej + WS1 |

Do potwierdzenia w labie (nie mam tego sprawdzonego): pakiet bulk przez Install-ProvisioningPackage w SetupComplete,
zachowanie klasycznego OOBE po dołączeniu tokenem (scenariusz 2), przypisanie urządzenia w WS1 do użytkownika Entra po
rejestracji kontem staging przed OOBE.
