# Lista kontrolna weryfikacji Etapu W0 (`g580`)

Przejdź przez wszystkie 9 punktów **po** uruchomieniu `playbooks/site.yml`. Zakończenie tej listy bez błędów zamyka Etap W0 i pozwala bezpiecznie scalić gałąź do `main`.

---

## 1. Idempotencja Ansible

Uruchom playbook ponownie z Arch PC:

```bash
cd ~/projects/homelab-infra/ansible
ansible-playbook playbooks/site.yml -K
```

- **Oczekiwany wynik:** `changed=0` oraz `failed=0` w podsumowaniu `PLAY RECAP`.

---

## 2. Sieć Zero Trust (UFW + zamknięty LAN)

Z **Arch PC** przeskanuj port SSH na **lokalnym adresie Wi-Fi** serwera (`<LAN_IP>`):

```bash
nc -zv -w 3 <LAN_IP> 22
# lub: nmap -Pn -p 22 <LAN_IP>
```

- **Oczekiwany wynik z LAN:** połączenie odrzucone / timeout (`filtered` lub brak odpowiedzi).

Na **G580** sprawdź aktywne reguły zapory:

```bash
sudo ufw status verbose
```

- **Oczekiwany wynik na G580:**
  - `Status: active`
  - `Default: deny (incoming), allow (outgoing), deny (routed)` (lub `disabled (routed)`)
  - Wyłącznie reguła `22/tcp on tailscale0 -> ALLOW IN`.

---

## 3. Hardening OpenSSH (`00-hardening.conf`)

Na **G580** sprawdź efektywną konfigurację uruchomionego demona `sshd`:

```bash
sudo sshd -T | grep -E '^(passwordauthentication|permitrootlogin|pubkeyauthentication|kbdinteractiveauthentication|allowusers)'
```

- **Oczekiwany wynik:**
  - `passwordauthentication no`
  - `permitrootlogin no`
  - `pubkeyauthentication yes`
  - `kbdinteractiveauthentication no`
  - `allowusers konzi`

---

## 4. Tailscale i MagicDNS

Z **Arch PC** zweryfikuj stan tunelu oraz krótkie połączenie SSH po MagicDNS:

```bash
tailscale status
ssh -i ~/.ssh/id_ed25519_homelab konzi@g580 hostname
```

- **Oczekiwany wynik:** węzeł `g580` widoczny w `tailscale status`, a `ssh` zwraca `g580` bez pytania o hasło konta.

---

## 5. Układ dysków LVM (`250 GB SSD` + rezerwa VG)

Na **G580** sprawdź rozkład wolumenów i punktów montowania:

```bash
lsblk
sudo lvs
sudo vgs
df -h / /var/lib/docker /srv
ls -ld /srv
```

- **Oczekiwany wynik:**
  - `/` ~ `60G` (`ubuntu-vg/ubuntu-lv`)
  - `/var/lib/docker` ~ `50G` (`ubuntu-vg/lv_docker`)
  - `/srv` ~ `60G` (`ubuntu-vg/lv_srv`, właściciel `konzi:konzi`)
  - W `ubuntu-vg` pozostaje ~`50G` nieprzydzielonej rezerwy (`VFree`).

---

## 6. Tryb „laptop jako serwer” (klapa, Wi-Fi, zram, temperatury)

1. Na **G580** sprawdź brak oszczędzania energii Wi-Fi, aktywność `zram`, `thermald` oraz odczyt temperatur:
   ```bash
   iw dev
   iw dev <INTERFEJS_WIFI> get power_save
   zramctl
   swapon --show
   systemctl is-active thermald
   sensors
   ```
   - **Oczekiwany wynik:** `Power save: off`, widoczny swap na `/dev/zram0` (`zstd`, do 4 GB), `thermald` = `active`, temperatury w spoczynku w normie.
2. **Test fizyczny klapy:** otwórz na Arch PC ciągły `ping g580` i aktywną sesję SSH, następnie **zamknij pokrywę Lenovo G580** i odczekaj ~60 sekund.
   - **Oczekiwany wynik:** ping i sesja SSH działają bez przerwania, ekran laptopa wygaszony po bezczynności (`consoleblank=60`).

---

## 7. Aktualizacje bezpieczeństwa (bez auto-restartu)

Na **G580**:

```bash
systemctl is-active unattended-upgrades
apt-config dump APT::Periodic Unattended-Upgrade::Automatic-Reboot
```

- **Oczekiwany wynik:** usługa `active`, `Unattended-Upgrade::Automatic-Reboot "false";`.

---

## 8. Docker CE i rotacja logów

Na **G580** (w **nowej sesji SSH**, aby aktywowała się grupa `docker` przypisana w `site.yml` — jako `konzi`, bez `sudo`):

```bash
id
docker --version
docker compose version
docker info --format 'Logging={{.LoggingDriver}} LiveRestore={{.LiveRestoreEnabled}}'
```

- **Oczekiwany wynik:** `Logging=json-file LiveRestore=true`, brak błędu uprawnień grupy `docker`.

Szybki test izolacji sieciowej kontenera (bindowanie wyłącznie na `127.0.0.1`; krótkie `sleep 2` daje `nginx` czas na start zanim uderzy `curl`):

```bash
docker run --rm -d --name w0-nginx-test -p 127.0.0.1:18080:80 nginx:alpine
sleep 2 && curl -I http://127.0.0.1:18080
docker stop w0-nginx-test
```

---

## 9. Obowiązkowy test backupu i odtwarzania (`Restic` -> Google Drive)

Na **G580** przetestuj pełny cykl zapisu i odzyskania danych przed pojawieniem się realnych usług:

1. Utwórz plik testowy w `/srv`:
   ```bash
   echo "homelab-w0-backup-ok-$(date +%F)" > /srv/backup-test.txt
   ```
2. Ręcznie wyzwól usługę backupu i sprawdź logi:
   ```bash
   sudo systemctl start restic-backup.service
   sudo journalctl -u restic-backup.service -n 50 --no-pager
   ```
3. Potwierdź obecność migawki w repozytorium Google Drive:
   ```bash
   sudo /usr/local/sbin/restic-env snapshots
   ```
4. Usuń plik lokalny i odtwórz go z repozytorium Restic do `/tmp/restore-test` (`restic restore` zakłada katalog docelowy z uprawnieniami `0700 root:root`, dlatego odczyt pliku wymaga `sudo cat`):
   ```bash
   rm /srv/backup-test.txt
   sudo /usr/local/sbin/restic-env restore latest --target /tmp/restore-test --include /srv/backup-test.txt
   sudo cat /tmp/restore-test/srv/backup-test.txt
   ```
5. Posprzątaj pliki testowe i sprawdź harmonogram `systemd timer`:
   ```bash
   sudo rm -rf /tmp/restore-test
   systemctl list-timers restic-backup.timer
   ```

- **Oczekiwany wynik:** odtworzony plik zawiera dokładnie wpisany ciąg znaków, a `restic-backup.timer` ma zaplanowane kolejne uruchomienie w oknie `03:30–03:45` (`RandomizedDelaySec=15m`).
