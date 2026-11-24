# Obrazy Windows: kopie LAN i obrazy serwisowane

## Po co
OSDCloud instaluje ESD Windows z katalogu Microsoftu. Microsoft odświeża te ESD tylko kilka razy w roku: w październiku 2026
najnowszy ESD 25H2 to nadal build 26200.8653 z czerwca, więc nowa stacja najpierw długo aktualizuje się w OOBE
(„Uzyskiwanie najnowszych funkcji i aktualizacji zabezpieczeń”). WSODeployer trzyma na serwerze dwa rodzaje obrazów:

| Rodzaj | Kto tworzy | Zawartość |
|---|---|---|
| **esd** (kopia LAN, v0.7) | serwer, pobrany raz od Microsoftu | ESD z katalogu bez zmian |
| **serwisowany** (v0.17) | `ServiceImage` na stanowisku admina | ESD + najnowsza aktualizacja zbiorcza, tylko edycje używane przez profile |

Urządzenie bierze najnowszy obraz **serwisowany** dla swojej nazwy systemu, języka, aktywacji i edycji; bez niego - kopię
LAN ESD (źródło obrazu w profilu Auto / LAN) albo Microsoft.

## ServiceImage - raz w miesiącu
Na stanowisku admina (po Install-WsoWorkstation.ps1, PowerShell jako administrator, ok. 25 GB wolnego miejsca):
```powershell
.\client\WSODeployer.ps1 -Action ServiceImage                       # wszystkie systemy z profili serwera
.\client\WSODeployer.ps1 -Action ServiceImage -OSName 'Windows 11 25H2 x64' -Force
```
Dla każdego systemu / języka / aktywacji z profili serwera (źródło obrazu Auto / LAN) z edycjami, których używają:
1. **Microsoft Update Catalog**: najnowsza wydana aktualizacja zbiorcza („RRRR-MM Cumulative Update for Windows 11, version
   25H2 for x64-based Systems (KB…) (build)”) - bez Preview i bez Dynamic. Jej pliki .msu (24H2 / 25H2: z aktualizacją
   checkpoint) są pobierane raz i sprawdzane z SHA-1 w nazwie oraz podpisem Microsoft.
2. Gdy serwer ma już ten obraz (ta sama nazwa = system, język, aktywacja, build, KB, wszystkie edycje): nic do zrobienia
   (`-Force` buduje go od nowa).
3. ESD bazowy: kopia LAN z serwera (szybko), w przeciwnym razie Microsoft - w pamięci podręcznej
   `%ProgramData%\WSODeployer\Servicing` (`-OutDir`).
4. DISM: edycje z ESD do WIM, aktualizacja do każdej, czyszczenie składników, eksport kompaktowy
   (`W11-25H2-x64-pl-pl-Volume-26200.9457-KB5129195.wim`).
5. Wysłanie na serwer (klucz admina, strumieniowo, SHA-256 porównany) - strona konsoli **Obrazy** pokazuje go z oznaczeniem
   *serwisowany*, buildem, KB, edycjami i wiekiem (na czerwono po 35 dniach: wyszła już kolejna miesięczna aktualizacja).

Uruchamiaj dzień po Patch Tuesday (drugi wtorek miesiąca), np. jako zadanie harmonogramu konta administratora.
Stare obrazy serwisowane można usunąć na stronie Obrazy; urządzenia zawsze biorą najnowszy.

## Sprawdzenie
- Konsola **Obrazy** → tabela *Profile i ich kopia LAN*: przy każdym profilu build serwisowany, który dostanie.
- Log urządzenia: `Serwisowany obraz Windows z serwera: … (build …, KB…)`; `winver` po instalacji.
