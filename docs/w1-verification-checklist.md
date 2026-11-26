# Lista kontrolna weryfikacji Etapu W1 (`g580`)

Przejdź przez wszystkie 6 punktów **po** wdrożeniu `playbooks/site.yml` (z rolą `agent_host`) i uruchomieniu kroków z [docs/w1-agent-host-runbook.md](w1-agent-host-runbook.md).

---

## 1. Idempotencja Ansible

Uruchom playbook ponownie ze stacji roboczej:

```bash
cd ~/projects/homelab-infra/ansible
ansible-playbook playbooks/site.yml --diff -K
```

- **Oczekiwany wynik:** `changed=0` oraz `failed=0` w podsumowaniu `PLAY RECAP`.

---

## 2. Hardening SSH z lokalnym tunelowaniem

Na **G580** sprawdź zarówno plik konfiguracyjny, jak i efektywną konfigurację uruchomionego demona `sshd`:

```bash
grep -i AllowTcpForwarding /etc/ssh/sshd_config.d/00-hardening.conf
sudo sshd -T | grep -i allowtcpforwarding
```

- **Oczekiwany wynik:**
  - `AllowTcpForwarding local`
  - `allowtcpforwarding local`

---

## 3. Środowisko `agent_host` i `linger`

Na **G580** sprawdź `linger`, zainstalowane pakiety oraz katalogi robocze:

```bash
loginctl show-user <ADMIN_USER> -p Linger
node -v
gh --version | head -n 1
ls -ld ~/projects ~/ops ~/life
```

- **Oczekiwany wynik:**
  - `Linger=yes`
  - obecne `node` (np. `v18.x`), `gh`, a także `jq`, `rg`, `pipx`
  - katalogi `~/projects`, `~/ops`, `~/life` należą do `<ADMIN_USER>:<ADMIN_USER>`.

---

## 4. Autostart po `sudo reboot` bez udziału PC

1. Zrestartuj serwer (`sudo reboot`) i **nie** uruchamiaj *Connect over SSH* na stacji roboczej.
2. Po ~45 sekundach sprawdź na urządzeniu mobilnym / w przeglądarce (`Remote Control`), czy host (np. `homelab-g580`) sam pojawił się jako **Online**.
3. Po zalogowaniu na **G580** przez zwykłe `ssh g580` sprawdź status usługi użytkownika:
   ```bash
   systemctl --user status antigravity-hub.service --no-pager
   ```

- **Oczekiwany wynik:**
  - `Active: active (running)` w cgroupie `user@<UID>.service` (nie w `session-*.scope`),
  - limity pamięci aktywne (np. `high: 2.0G max: 3.0G`) i zużycie w normie (~`200–300M`),
  - w logach brak błędu brakującego `node` dla sidecara oraz obecne potwierdzenie poprawnej sesji (`get_auth_status valid=true`).

---

## 5. Brak duplikacji procesów przy *Connect over SSH*

1. Na **G580** sprawdź PID działającego procesu przed połączeniem ze stacji roboczej:
   ```bash
   ps aux | grep -i language_server
   ```
2. Na stacji roboczej uruchom **Connect to SSH Host → `g580`**.
3. Na **G580** ponownie wykonaj `ps aux | grep -i language_server`.

- **Oczekiwany wynik:** dokładnie **1** proces `language_server` z tym samym PID przed i po połączeniu (brak duplikatów).

---

## 6. Rozszerzony backup Restic (`/home/<ADMIN_USER>`)

Na **G580** wyzwól ręcznie usługę backupu i zweryfikuj zawartość migawki:

```bash
sudo systemctl start restic-backup.service
sudo journalctl -u restic-backup.service -n 25 --no-pager
sudo /usr/local/sbin/restic-env snapshots --latest 2
sudo /usr/local/sbin/restic-env ls latest /home/<ADMIN_USER> | grep -E '(\.gemini|\.antigravity-hub-server|projects)' | head -n 10
```

- **Oczekiwany wynik:**
  - `no errors were found` w logach `restic-backup.service`,
  - w `snapshots` najnowsza migawka zawiera ścieżki `/etc`, `/home/<ADMIN_USER>` oraz `/srv`,
  - `ls latest` pokazuje `/home/<ADMIN_USER>/.gemini` oraz `/home/<ADMIN_USER>/projects`, natomiast **wyklucza** `/home/<ADMIN_USER>/.antigravity-hub-server`.
