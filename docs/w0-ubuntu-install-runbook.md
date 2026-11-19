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
3. Przygotuj pendrive instalacyjny:
   - **Ścieżka A (zalecana — Ventoy):** Jeśli masz pendrive z **Ventoy**, wystarczy skopiować zweryfikowany plik `.iso` bezpośrednio na główną partycję pendrive'a (bez ponownego formatowania).
   - **Ścieżka B (klasyczne `dd`):** Jeśli zapisujesz obraz bezpośrednio, najpierw upewnij się przez `lsblk`, które urządzenie to pendrive (np. `/dev/sdX` — pomyłka wymaże dysk!):
     ```bash
     sudo dd bs=4M if=ubuntu-24.04.x-live-server-amd64.iso of=/dev/sdX status=progress oflag=sync
     ```

---

## Krok 2: Ustawienia BIOS / Boot Menu na Lenovo G580

1. Wejdź do BIOS-u (klawisz `F2` / `Fn+F2` lub przycisk *Novo* na wyłączonym laptopie).
2. **Tryb startu:** preferowany **UEFI**. Jeśli starszy BIOS G580 będzie miał problem z bootem wybranego pendrive'a w UEFI, użyj trybu **Legacy** jako fallbacku.
3. **Karta graficzna:** potwierdź wyłączoną dedykowaną **NVIDIA GT 710M** (zostawiamy aktywną wyłącznie zintegrowaną Intel HD 4000 — mniejsze temperatury i zużycie prądu 24/7).
4. Uruchom boot menu (`F12` / *Novo*) i wystartuj instalator Ubuntu Server z USB.

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
   - Upewnij się, że w grupie wolumenów `ubuntu-vg` zostaje ~**169 GiB** wolnego miejsca — Ansible utworzy z niego osobne wolumeny dla `/var/lib/docker` (`50G`) i `/srv` (`60G`), zostawiając ~`59G` rezerwy.
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
3. **Spiecie tunelu Tailscale (na G580):**
   ```bash
   ssh -i ~/.ssh/id_ed25519_homelab konzi@<LAN_IP>
   sudo tailscale up
   ```
   Otwórz wyświetlony link autoryzacyjny w przeglądarce i zaakceptuj urządzenie `g580`. Następnie z Arch PC sprawdź nowe połączenie po MagicDNS:
   ```bash
   ssh -i ~/.ssh/id_ed25519_homelab konzi@g580
   ```
4. **Przygotowanie backupów przed Fazą 2:**
   - Wygeneruj token OAuth Google Drive na Arch PC (`rclone authorize "drive"`) i zapisz plik `ansible/files/rclone.conf` z sekcją `[gdrive]`.
   - Utwórz `ansible/group_vars/all/secrets.yml` z `secrets.yml.example` i wstaw wygenerowane silne hasło do Restic (`openssl rand -base64 32` — zapisz kopię w menedżerze haseł!).
5. **Faza 2 — `site.yml` (już wyłącznie po Tailscale):**
   ```bash
   ansible-playbook playbooks/site.yml --check --diff -K
   ansible-playbook playbooks/site.yml -K
   ```
6. Na końcu przejdź przez listę testów odbiorczych w [docs/w0-verification-checklist.md](w0-verification-checklist.md).
