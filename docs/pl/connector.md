# Konektor domeny (Cloud Connector, v0.12 / v0.13)

Konektor to usługa Windows na serwerze w domenie. Wykonuje pracę w katalogu za serwer WSODeployer, który działa w
kontenerze Linux i nie ma żadnych praw w Active Directory:

- **Bloby ODJ**: `djoin /provision /reuse` dla instalowanej stacji. Blob jest szyfrowany do deploy.cer i do sejfu
  sekretów serwera i wraca jako tekst CMS - serwer nigdy nie ma użytecznego bloba.
- **Cykl życia kont komputerów** (strona Urządzenia → zaznacz stacje → *Active Directory*):
  - *przenieś konto do jego OU* - OU, które stacja ma teraz (stacja > grupa > profil > deploy.json);
  - *zresetuj konto komputera* (administratorzy) - jak „Resetuj konto” w ADUC; stacja traci relację zaufania;
  - *wyłącz / włącz konto komputera* (administratorzy);
  - wycofanie stacji, która ma konto komputera, samo zleca *wyłączenie* (`Wso:Connector:DisableOnRetire`).

Nazwę komputera, OU i domenę dla zlecenia ODJ liczy serwer, dokładnie tak jak silnik - konektor nie ma własnych reguł
nazewnictwa.

## Wysoka dostępność

Zarejestruj kilka konektorów (po jednym na oddział albo dwa na domenę). Każde zlecenie bierze jeden konektor
(`FOR UPDATE SKIP LOCKED`); praca konektora, który przestał odpowiadać, wraca do kolejki po 10 minutach.

## Konsola: Konektory domeny

- **Zarejestruj konektor** - jednorazowy **kod rejestracji** (`WSOC1.…`, ważny 24 h) pokazywany jeden raz; **Nowy kod**
  dla przeinstalowanej albo przeniesionej usługi.
- **Pobierz konektor (MSI)** - budowany na stacji technika przez `.\client\Build-WsoDownloads.ps1` (umieszcza go w
  `data\downloads` serwera; `-Dev` dla instancji testowej).
- Stan z sygnału (co minutę): online / offline, serwer i **konto usługi**, domena i kontroler domeny, wersja i ważność
  certyfikatu, ostatni błąd, liczniki, a dla każdej OU używanej przez serwer (deploy.json, profile, grupy stacji, stacje)
  informacja, **czy konto może tam tworzyć konta komputerów**.
- **Delegacja dla konta usługi** - polecenie `Grant-WsoConnectorRights.ps1` (strona Pobieranie) z tymi OU albo te same
  polecenia `dsacls` ręcznie (administrator domeny uruchamia je raz).
- Strona **Pobieranie**: MSI konektora, konektor w PowerShell (ZIP), skrypt delegacji i narzędzia technika (ZIP), z wersją,
  datą zbudowania i SHA-256 - budowane przez `.\client\Build-WsoDownloads.ps1`.
- **Wyłącz** od razu odrzuca konektor (usługa się zatrzymuje); **Usuń** kasuje go.
- **Operacje na kontach komputerów** - w kolejce / w toku / wykonane / błąd, kto zlecił i który konektor wykonał.
- Konektory v0.12 ze stałym kluczem i agenci v0.2 z kluczem w appsettings (`Wso:AgentKeys`) działają dalej.

## Instalacja (MSI)

1. Konsola → Konektory domeny → **Zarejestruj konektor** → skopiuj kod rejestracji; **Pobierz konektor (MSI)**.
2. Administrator domeny deleguje prawa kontu usługi (polecenia w sekcji *Delegacja*): tworzenie / usuwanie obiektów
   komputer, reset hasła, ograniczenia konta, zapis nazwy DNS i SPN. Działa też gMSA (`New-ADServiceAccount`, z prawem
   pobierania hasła na serwerze konektora).
3. Na serwerze w domenie: uruchom MSI, wklej kod, podaj konto (domena + użytkownik + hasło albo gMSA `nazwa$` bez
   hasła; pusty użytkownik = LocalSystem, czyli konto komputera serwera). Instalacja cicha:

```powershell
msiexec /i WSODeployer.Connector.msi /qn JOINCODE="WSOC1...." SERVICEDOMAIN=FIRMA SERVICEUSER=svc-wso SERVICEPASSWORD="..."
msiexec /i WSODeployer.Connector.msi /qn JOINCODE="WSOC1...." SERVICEDOMAIN=FIRMA SERVICEUSER=gmsa-wso$
```

4. Usługa **WSODeployerConnector** startuje, tworzy własny klucz RSA (DPAPI LocalMachine, `%ProgramData%\WSODeployer\
   Connector` dostępny tylko dla SYSTEM, Administratorów i jej konta), rejestruje się kodem (jednorazowo - kod jest
   usuwany) i dostaje certyfikat CA serwera (1 rok, odnawiany 30 dni przed końcem). Żądania podpisuje tym kluczem, więc
   nie przechowuje żadnego klucza serwera i nie potrzebuje certyfikatu klienta (mTLS) za proxy.
5. Konsola pokazuje ją online, z kontem i prawami w OU. Dziennik: `C:\ProgramData\WSODeployer\Connector\Logs\connector-RRRRMMDD.log`
   (plik na dzień, 14 dni; wiersz startu podaje konto, na którym działa usługa) oraz dziennik zdarzeń *Application*, źródło
   *WSODeployer Connector*.

MSI nadaje kontu prawo „Logowanie jako usługa” oraz dostęp do folderu danych i `HKLM\SOFTWARE\WSODeployer\Connector`;
**nie** zmienia Active Directory. Aktualizacja (nowszy MSI) zachowuje tożsamość; nowy kod powoduje ponowną rejestrację.
Serwer konektora musi ufać certyfikatowi TLS serwera (publiczne CA albo własne CA w Trusted Root).
`agent\Install-WsoAgent.ps1` zostaje do instalacji skryptem (klucz v0.12) i `-Mode task` (agent v0.2).

**Konto usługi musi zostać to samo.** Active Directory (KB5020276, CVE-2022-38042) pozwala `djoin /reuse` przejąć
istniejące konto komputera tylko jego właścicielowi - kontu, na którym działał konektor, gdy je tworzył. Konektor
przełączony na inne konto (np. LocalSystem = `DOMENA\HOST$`) kończy reinstalację błędem `0xaac` („Re-using the account was
blocked by security policy”). Albo zostaw konto, albo dodaj konto (konta) konektora do zasady kontrolera domeny *Domain
controller: Allow computer account re-use during domain join* (Default Domain Controllers Policy → Security Options).
## Dalej (plan)

- **Entra ID**: tryby dołączania w profilu Workgroup / AD (ODJ) / Hybrid / Entra. Entra przez pakiet bulk enrollment
  z Windows Configuration Designer (szyfrowany do sejfu, nakładany offline) oraz - dla klientów z Intune - Autopilot
  (hardware hash wgrywany przez konektor przez Microsoft Graph). Konektor usuwa też stare rekordy urządzeń w Entra i
  dodaje urządzenia do grup Entra.
