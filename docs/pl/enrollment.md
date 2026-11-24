# Rejestracja urządzeń (v0.11)

Do wersji 0.10 każdy nośnik zawierał **deploy.pfx** (klucz prywatny certyfikatu wdrożeniowego), a technik wpisywał w
WinPE jego **PIN**. Ten sam klucz podpisywał żądania wszystkich urządzeń i odszyfrowywał wszystkie sekrety: zgubiony
pendrive plus znany PIN odsłaniały wszystko, a zmiana klucza oznaczała przebudowę wszystkich nośników.

Od v0.11 urządzenie może się **zarejestrować** (model z projektu Winch):

1. **Konsola → Rejestracja urządzeń**: administrator tworzy
   - **token** dla nośnika: konsola pokazuje jeden raz **kod dołączenia** (`WSOD1.…` = adres serwera + token);
   - albo **kod parowania** (`ABCD-EFGH`, musi wygasać, domyślnie najwyżej 168 h): nośnik uniwersalny ma kod
     dołączenia bez tokenu, a technik wpisuje kod w kreatorze WinPE.
   Token / kod może kierować nowe stacje do **grupy stacji**, mieć limit czasu i użyć, można go w każdej chwili
   odwołać. Przechowywany jest tylko jego skrót SHA-256.
2. **Nośnik**: `.\client\New-WsoBootMedia.ps1 -JoinCode 'WSOD1.…'` zapisuje kod dołączenia w `media.json`; bez
   deploy.pfx i bez PIN-u.
3. **WinPE**: silnik tworzy klucz RSA 2048 **w pamięci** (nigdy nie opuszcza RAM-dysku), wysyła CSR z numerem
   seryjnym, adresami MAC, SMBIOS UUID, modelem i adresami IP (`POST /api/enroll`).
   - znana, zatwierdzona stacja od razu dostaje certyfikat **CA serwera**;
   - nowa stacja pojawia się w **Zatwierdzaniu** (grupa z tokenu albo z reguł) i kreator czeka, jak dotąd;
   - certyfikat żyje **24 h** (`Wso:Enrollment:CertificateHours`) - jedno wdrożenie.
4. Stacja podpisuje żądania tym kluczem (`X-Wso-Credential` + zwykłe nagłówki podpisu). Serwer przyjmuje go
   **tylko dla jej własnego rekordu** (identify, ODJ).
5. **Sekrety** (blob ODJ, konto staging WS1, hasło lokalnego administratora, unattend WS1 Drop Ship) są zapisane
   zaszyfrowane do certyfikatu wdrożeniowego **i** do **sejfu sekretów** serwera. Zarejestrowana stacja prosi serwer
   o kopię zaszyfrowaną jej kluczem (`POST /api/device/reprotect`; endpoint ODJ robi to sam).

## Klucze serwera

`Wso:Enrollment:KeyPath` (Docker: `data/ca`, tworzone przy pierwszym starcie, czytelne tylko dla API):
`ca.crt` / `ca.key` (CA urządzeń) oraz `vault.crt` / `vault.key` (sejf sekretów). **Rób ich kopię razem z bazą** -
utrata klucza sejfu sprawia, że zapisane sekrety są nieczytelne dla zarejestrowanych stacji (nośniki z deploy.pfx
działają dalej). Publiczny certyfikat sejfu: `/content/vault.cer`.

## Przejście z deploy.pfx

Sekrety zapisane przed v0.11 są zaszyfrowane tylko do deploy.cer. Dla zarejestrowanych stacji zapisz je ponownie:
- **konto staging**: `.\client\Initialize-WsoContent.ps1` (krok 4) ze znanym adresem serwera;
- **lokalny administrator**: otwórz profil, wpisz hasło jeszcze raz, zapisz;
- **pakiet WS1 Drop Ship**: wgraj go ponownie;
- **bloby ODJ**: zaktualizuj agenta domenowego (szyfruje do sejfu z work itemu) - nowe bloby są w porządku.
Stary sekret daje na stacji czytelny błąd (`secret.notInVault`). Gdy wszystkie nośniki używają kodów dołączenia,
ustaw `WSO_ALLOW_DEPLOY_PFX=false` (`Wso:Enrollment:AllowDeploymentCertificate`): żądania podpisane deploy.pfx są
odrzucane.

## Bezpieczeństwo

- Token w kodzie dołączenia działa jak hasło okaziciela: kto ma nośnik, może poprosić o rejestrację. **Nowa** stacja
  nadal wymaga zatwierdzenia; **znana** stacja dostaje certyfikat od razu - ograniczaj tokeny liczbą użyć / czasem i
  odwołuj zgubione nośniki.
- CA podpisuje tylko klucz publiczny z CSR (którego podpis dowodzi posiadania klucza); nazwę i rozszerzenia ustala
  serwer, więc stacja nie może podać się za inną.
- Odwołanie poświadczenia (Rejestracja urządzeń → Certyfikaty urządzeń) zatrzymuje trwające wdrożenie przy jego
  następnym żądaniu.
