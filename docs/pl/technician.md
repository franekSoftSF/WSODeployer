# Aplikacja technika (WSODeployer.Technician.exe)

Okno nad skryptami technika z `client\`. Nie ma własnej logiki wdrożenia: pod spodem działają `Install-WsoWorkstation.ps1`
i `New-WsoBootMedia.ps1` (Windows PowerShell 5.1), a ich komunikaty widać na żywo.

## Skąd ją wziąć
Konsola WWW → **Pobieranie** → *WSODeployer.Technician.zip*. Rozpakuj **cały** ZIP (program szuka obok siebie
`client\New-WsoBootMedia.ps1`) i uruchom `WSODeployer.Technician.exe`; poprosi o uprawnienia administratora (WinPE, ADK i
DISM ich wymagają). Język (EN / PL) jak w Windows, przycisk w nagłówku go przełącza.

## Zakładka 1 – Stanowisko
| Sprawdzenie | Brak → |
|---|---|
| Uprawnienia administratora | uruchom program ponownie (sam o nie poprosi) |
| Windows PowerShell 5.1 | część Windows |
| Windows ADK + dodatek WinPE | **Zainstaluj / napraw** (jednorazowo ok. 4 GB) |
| Moduł OSD (OSDCloud) | **Zainstaluj / napraw** |

## Zakładka 2 – Nośnik startowy
1. **Kod dołączenia** z konsoli (strona *Rejestracja urządzeń*):
   - *Token dla nośnika* → nośnik **z tokenem**: urządzenia rejestrują się same (nowa stacja nadal czeka na zatwierdzenie);
   - sam **adres serwera** (`https://…`) albo polecenie nośnika uniwersalnego z tej strony → **nośnik uniwersalny**: technik
     wpisuje w WinPE kod parowania (na nośniku nie ma sekretu).
2. **Certyfikat TLS serwera** – *Pobierz z serwera*: gdy certyfikat serwera nie jest publicznie zaufany (self-signed / własne
   CA), program zapisuje główny certyfikat łańcucha do `work\cert\server.cer` dla WinPE i pokazuje jego SHA-1 – porównaj
   z serwerem (`server\tls\server.cer`) przed budową.
3. **Opcje**: sterowniki VMware + VMware Tools (domyślnie – rozdzielczość w WinPE, vmxnet3, pvscsi; narzędzia dla urządzeń,
   których deploy.json ich nie ma), VirtIO dla Proxmox / KVM, pendrive (OSDCloud zapyta o dysk – wszystkie dane na nim
   zostaną usunięte), język kreatora WinPE, ścieżka ISO (domyślnie `work\media\WSODeployer.iso`).
4. **Zbuduj nośnik** – za pierwszym razem kilka minut; potem WinPE jest używany ponownie, dopóki jego sterowniki się nie zmienią (od nowa powstaje tylko folder WSODeployer i ISO), *Zbuduj WinPE od nowa* to wymusza; *Przerwij* kończy skrypt, *Otwórz folder* pokazuje ISO.

## Logi do analizy
**Zapisz logi do analizy** (nad dziennikiem) pakuje dziennik okna, `%TEMP%\WSODeployer.log`, logi workspace OSDCloud
(`C:\OSDCloud\WSODeployer\Logs`) i podsumowanie stanowiska do jednego ZIP-a w `work\logs`. Kody dołączenia są zamaskowane
(`WSOD1.***`); certyfikatów ani kluczy program nie zbiera - ZIP można przekazać do wsparcia.

## Dla programistów
`technician\WSODeployer.Technician` (.NET 10 WPF), testy w `technician\WSODeployer.Technician.Tests` (uruchamia je
`tests\Invoke-Tests.ps1`). `client\Build-WsoDownloads.ps1` publikuje ją jako jeden samodzielny plik (ok. 70 MB) do paczki
technika.
