# Runbook instalacji Ubuntu Server 24.04 LTS na Lenovo G580 (Etap W0)

Ten dokument opisuje minimalną ręczną instalację systemu na **Lenovo G580** z pojedynczym dyskiem **250 GB SATA SSD** (przed zakupem kieszeni caddy na `1 TB HDD`). Cała późniejsza konfiguracja trafia już deklaratywnie przez Ansible.

---

## Krok 1: Przygotowanie nośnika USB i weryfikacja ISO

1. Pobierz oficjalny obraz **Ubuntu Server 24.04 LTS (`noble`, `amd64`)** oraz plik `SHA256SUMS` (i opcjonalnie `SHA256SUMS.gpg`) ze strony `releases.ubuntu.com`.
2. Zweryfikuj sumę kontrolną pobranego obrazu na Arch PC (praktyka DevSecOps przed zapisem na USB):
   ```bash
   sha256sum -c SHA256SUMS --ignore-missing
   ```
   Oczekiwany wynik przy pliku `.iso`: `OK`.
3. Przygotuj pendrive instalacyjny metodą **GPT + FAT32 + ESP** (skopiowanie obu partycji obrazu ISO przez `losetup -fP`):

   > **Dlaczego nie Ventoy ani zwykłe `dd` na Lenovo G580?**
   > - **Ventoy:** instalator `Subiquity` / `Curtin` na Ubuntu 24.04.5 wpada w błąd mapera urządzeń (`ValueError: Invalid dep_id (disk-dm-0) not in storage config`) oraz `casper-md5check: fail`.
   > - **Hybrydowe ISO przez `dd`:** starszy BIOS Lenovo G580 nie wykrywa partycji EFI z obrazu hybrydowego i proponuje wyłącznie start **Legacy**, który wywołuje blokadę whitelisty nieoryginalnej karty Wi-Fi (`Unauthorized wireless network card is plugged in`).

   Najpierw sprawdź nazwę urządzenia pendrive'a przez `lsblk` (np. `/dev/sdX` — pomyłka wymaże dysk!) i odmontuj jego partycje, a następnie przygotuj czyste GPT + FAT32 z flagą ESP i skopiuj zawartość obu partycji ISO (`p1` dane instalatora + `p2` `/EFI/boot/bootx64.efi`):
   ```bash
   sudo umount /dev/sdX* 2>/dev/null || true
   sudo wipefs -a /dev/sdX
   sudo parted -s /dev/sdX mklabel gpt mkpart ESP fat32 1MiB 100% set 1 esp on
   sudo mkfs.vfat -F 32 -n UBUNTU_EFI /dev/sdX1

   # Podepnij ISO jako urządzenie pętli ze skanowaniem partycji (zwróci np. /dev/loop0)
   LOOP_DEV=$(sudo losetup -fP --show ubuntu-24.04.5-live-server-amd64.iso)
   sudo mkdir -p /mnt/iso_p1 /mnt/iso_p2 /mnt/usb
   sudo mount -o ro "${LOOP_DEV}p1" /mnt/iso_p1
   sudo mount -o ro "${LOOP_DEV}p2" /mnt/iso_p2
   sudo mount /dev/sdX1 /mnt/usb

   # Kropka po ukośniku (/.) kopiuje także ukryty katalog .disk wymagany przez instalator
   # (ostrzeżenia o braku obsługi symlinków na FAT32, np. 'ubuntu', można zignorować)
   sudo cp -r /mnt/iso_p1/. /mnt/usb/
   sudo cp -r /mnt/iso_p2/. /mnt/usb/
   sync

   sudo umount /mnt/usb /mnt/iso_p1 /mnt/iso_p2
   sudo losetup -d "$LOOP_DEV"
   sudo rmdir /mnt/usb /mnt/iso_p1 /mnt/iso_p2
   ```

---

## Krok 2: Ustawienia BIOS / Boot Menu na Lenovo G580

1. Wejdź do BIOS-u (klawisz `F2` / `Fn+F2` lub przycisk *Novo* na wyłączonym laptopie).
2. **Tryb startu (WYMAGANE UEFI / `EFI`):** ustaw tryb **UEFI** i w Boot Menu wybierz pozycję **`EFI`** dla pendrive'a. **Nie używaj trybu Legacy** — na G580 z zamienną kartą `Intel Centrino Advanced-N 6205` start Legacy wyzwala blokadę whitelisty BIOS-u (`Unauthorized wireless network card is plugged in`), podczas gdy boot `EFI` przechodzi POST bez problemu.
3. **Karta graficzna (`UMA Graphic`):** przestaw tryb grafiki na **`UMA Graphic`** (wyłącza dedykowaną NVIDIA GT 710M i zostawia wyłącznie zintegrowaną Intel HD 4000 — mniejsze temperatury i zużycie prądu 24/7).
4. Uruchom boot menu (`F12` / *Novo*) i wystartuj instalator Ubuntu Server z pozycji **`EFI`** USB.

---

## Krok 3: Przejście przez instalator (`Subiquity`)

Przechodź przez kolejne ekrany dokładnie według poniższej listy:

1. **Language:** `English` (czytelniejsze logi i komunikaty błędów).
2. **Keyboard configuration:** Layout `Polish`, Variant `Polish`.
3. **Choose type of install:** standardowy **`Ubuntu Server`** (nie minimal — łatwiejsza diagnostyka na starcie).
4. **Network connections:**
   - Wbudowana karta **Intel Centrino Advanced-N 6205** (`wlp...`) — połącz się z domową siecią Wi-Fi (preferuj pasmo **5 GHz**) z automatycznym **DHCP**.
   - Zapisz uzyskany tymczasowy adres IPv4 (np. `192.168.X.Y`) lub sprawdź go po pierwszym restarcie.
5. **Configure proxy / Ubuntu archive mirror:** zostaw domyślne wartości.
6. **Guided storage configuration (KRYTYCZNE — 250 GB SSD, bez LUKS):**
   - Wybierz `Use an entire disk` oraz zaznacz **`Set up this disk as an LVM group`**.
   - **NIE** zaznaczaj opcji szyfrowania (`Encrypt the LVM group with LUKS`) — serwer działa po Wi-Fi i bez sprawnej baterii, więc po reboocie nie powinien czekać na wpisanie hasła z klawiatury.
   - Na ekranie **Storage configuration** wybierz `ubuntu-lv` (zamontowane na `/`), kliknij **Edit** i zmniejsz rozmiar z domyślnego maksimum na **`60G`** (lub `60` GB).
   - Upewnij się, że w grupie wolumenów `ubuntu-vg` zostaje ~**160 GiB** wolnego miejsca (`250 GB` SSD = ~`223.6 GiB` surowej przestrzeni minus `1G /boot/efi`, `2G /boot` i `60G /`) — Ansible utworzy z niego osobne wolumeny dla `/var/lib/docker` (`50G`) i `/srv` (`60G`), zostawiając ~`50G` rezerwy.
7. **Profile setup:**
   - Your name: `Konrad`
   - Your server's name: **`g580`**
   - Pick a username: **`konzi`**
   - Choose a password: silne hasło lokalne (używane do `sudo` i ewentualnej konsoli awaryjnej po otwarciu klapy).
8. **Upgrade to Ubuntu Pro:** `Skip for now`.
9. **SSH Setup:**
   - Zaznacz **`Install OpenSSH server`**.
   - Opcjonalnie zaimportuj klucz publiczny SSH z GitHuba albo pomiń i wgraj swój dedykowany klucz `~/.ssh/id_ed25519_homelab.pub` przez `ssh-copy-id` zaraz po restarcie.
   - Do czasu uruchomienia Fazy 2 Ansible (`site.yml`) możesz zostawić tymczasowe logowanie hasłem na start lub od razu wgrać klucz.
10. **Featured Server Snaps:**
    - **NICZEGO NIE ZAZNACZAJ!** W szczególności **nie instaluj Snapa `docker`** — Ansible zainstaluje czyste pakiety z oficjalnego repozytorium Docker CE.
11. Dokończ instalację, wyciągnij pendrive przy komunikacie i zrestartuj laptopa.

---

## Krok 4: Pierwszy start i weryfikacja dostępu SSH z Arch PC

1. Zaloguj się lokalnie na G580 jako `konzi` i sprawdź aktualne IP interfejsu Wi-Fi:
   ```bash
   ip -br a
   ```
2. Na Arch PC (jeśli klucz nie był importowany z GitHuba albo używasz nowego klucza `~/.ssh/id_ed25519_homelab`) skopiuj klucz publiczny na serwer:
   ```bash
   ssh-copy-id -i ~/.ssh/id_ed25519_homelab.pub konzi@<LAN_IP>
   ```
3. Przetestuj logowanie kluczem i uprawnienia `sudo`:
   ```bash
   ssh -i ~/.ssh/id_ed25519_homelab konzi@<LAN_IP>
   sudo -v
   ```

---

## Krok 5: Uruchomienie konfiguracji Ansible (Faza C i D)

Wszystkie polecenia wykonujesz z katalogu `~/projects/homelab-infra/ansible` na Arch PC:

1. Przygotuj lokalny plik `inventory/hosts.yml` na bazie `inventory/hosts.yml.example`.
2. **Faza 1 — `bootstrap.yml` (jednorazowo po LAN):**
   ```bash
   ansible-playbook playbooks/bootstrap.yml --check --diff -e ansible_host=<LAN_IP> -K
   ansible-playbook playbooks/bootstrap.yml -e ansible_host=<LAN_IP> -K
   ```
3. **Spięcie tunelu Tailscale (na G580):**
   ```bash
   ssh -i ~/.ssh/id_ed25519_homelab konzi@<LAN_IP>
   sudo tailscale up
   ```
   Otwórz wyświetlony link autoryzacyjny w przeglądarce i zaakceptuj urządzenie `g580`. Następnie z Arch PC sprawdź nowe połączenie po MagicDNS:
   ```bash
   ssh -i ~/.ssh/id_ed25519_homelab konzi@g580
   ```
4. **Przygotowanie backupów przed Fazą 2 (własny OAuth Client ID zamiast wygaszonego wspólnego ID rclone):**
   - W Google Cloud Console wejdź bezpośrednio pod adres `https://console.cloud.google.com/projectcreate` i utwórz osobny projekt (np. `homelab-backup`) — bezpośrednie wejście przez `projectcreate` omija formularz podpięcia karty z triala 300 USD.
   - Włącz **Google Drive API** (*APIs & Services → Library*).
   - W **Google Auth Platform**:
     - Ustaw odbiorców jako **External**, uzupełnij wymagane dane oraz **Stronę marki (Branding)** (np. `konzi.dev`), aby odblokować przycisk **Opublikuj aplikację**.
     - **KRYTYCZNE:** zanim przejdziesz do autoryzacji w `rclone`, zmień status publikacji z `Testing` na **`In production`** (*Opublikuj aplikację* — bez wysyłania aplikacji do formalnej weryfikacji Google). W trybie `Testing` refresh tokeny wygasają twardo po 7 dniach (`invalid_grant`).
     - W sekcji **Clients** utwórz klienta OAuth typu **`Desktop app`** i skopiuj `Client ID` oraz `Client Secret`.
   - Z katalogu `~/projects/homelab-infra` na Arch PC uruchom kreator wskazując docelowy plik (gitignorowany):
     ```bash
     rclone config --config ansible/files/rclone.conf
     chmod 600 ansible/files/rclone.conf
     ```
     Utwórz remote o nazwie **`gdrive`** (`drive`), podaj własne `client_id` i `client_secret`, wybierz scope **`drive.file`** (*Access to files created by rclone only*) i zautoryzuj w przeglądarce (*Advanced → Go to ... (unsafe)*). **Nie zakładaj folderu `homelab-backups` ręcznie w przeglądarce** — przy scope `drive.file` `rclone`/`restic` musi utworzyć go sam.
   - Utwórz `ansible/inventory/group_vars/all/secrets.yml` z `secrets.yml.example` (`chmod 600`), wstaw wygenerowane silne hasło do Restic (`openssl rand -base64 32`) i zapisz jego kopię również poza serwerem G580 (np. w menedżerze haseł).
   - *(Na przyszłość: podczas planowanej rotacji poświadczeń OAuth przekaż `-e backup_rclone_force_update=true`, ponieważ na co dzień Ansible nie nadpisuje odświeżanych automatycznie tokenów w `/root/.config/rclone/rclone.conf`.)*
5. **Faza 2 — `site.yml` (już wyłącznie po Tailscale):**
   ```bash
   # Opcjonalny dry-run samego hardeningu SSH/UFW przed zamknięciem LAN
   # (pełne --check na świeżym hoście nie zadziała zanim istnieją pakiety Docker CE i unity systemd):
   ansible-playbook playbooks/site.yml --check --diff --tags security -K

   # Pełne wdrożenie po Tailscale:
   ansible-playbook playbooks/site.yml --diff -K
   ```
6. Na końcu przejdź przez listę testów odbiorczych w [docs/w0-verification-checklist.md](w0-verification-checklist.md).
