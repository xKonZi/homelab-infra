# Runbook Etapu W1 — Host agentowy 24/7 (`agent_host`)

Instrukcja wdrożenia **Etapu W1** na serwerze domowym **Lenovo G580** (`g580`): uruchomienie natywnego, lekkiego demona **Antigravity Hub** działającego 24/7 w trybie headless (bez X11/`Xvfb` i bez pełnego Electrona) oraz przygotowanie centralnego workspace'u pod pracę z wielu urządzeń.

---

## 1. Cel i architektura Etapu W1

- **Natywny `language_server` 24/7:** zamiast ciężkiego środowiska graficznego wykorzystujemy binarkę `~/.antigravity-hub-server/bin/<wersja>-x64/language_server` w trybie `--standalone --enable_sidecars --persistent_mode --workspace_id agy-hub-ssh` (~`250 MiB` RAM w spoczynku).
- **Trwałość po restarcie (`systemd --user` + `linger`):** usługa `antigravity-hub.service` uruchamia się automatycznie po starcie systemu bez konieczności logowania interaktywnego, z twardymi limitami pamięci cgroup (`MemoryHigh` / `MemoryMax`).
- **Dostęp z dowolnego klienta:**
  - **Stacja robocza / laptop:** natywne *Connect over SSH* po Tailscale (wymaga `AllowTcpForwarding local` po stronie serwera) — podpina się do już działającego procesu bez duplikowania instancji.
  - **Urządzenia mobilne / przeglądarka:** pełne graficzne UI przez **Remote Control**.
- **Bezpieczeństwo sekretów:** wrapper `~/.local/bin/antigravity-hub-daemon` dynamicznie wyszukuje najnowszą wersję binarki po aktualizacjach i generuje `ANTIGRAVITY_SERVER_CSRF_TOKEN` w zmiennej środowiskowej procesu (nie w parametrach wiersza poleceń widocznych w `ps`).

---

## 2. Wymagania wstępne

- Ukończony **Etap W0** (działający węzeł `g580` w sieci Tailscale, wykonany `playbooks/site.yml` i zaliczona lista [docs/w0-verification-checklist.md](w0-verification-checklist.md)).
- Działające połączenie `ssh g580` ze stacji roboczej.

---

## 3. Krok 1 — Pierwsze podpięcie binarki i logowanie OAuth

Na świeżym serwerze binarka `language_server` nie jest jeszcze pobrana do katalogu domowego. Pierwszą inicjalizację wykonuje desktopowa aplikacja Antigravity:

1. W aplikacji Antigravity na stacji roboczej wybierz **Connect to SSH Host → `g580`**.
   - Klient automatycznie pobierze serwer do `~/.antigravity-hub-server/bin/<wersja>-x64/language_server` oraz zainicjalizuje `~/.gemini/`.
2. Zaloguj się kontem Google w otwartym oknie zdalnym.
3. W **Settings → Application** w oknie połączonym z `g580`:
   - włącz **Enable Remote Control**,
   - ustaw czytelną nazwę urządzenia w **Device Name** (np. `homelab-g580`).

> **BHP bezpieczeństwa:** Kodu QR oraz linku parowania Remote Control **nigdy** nie zapisuj w repozytorium ani nie udostępniaj na zrzutach ekranu.

---

## 4. Krok 2 — Deklaratywne wdrożenie przez Ansible (`site.yml`)

Po jednorazowej inicjalizacji binarki (lub nawet przed nią — rola jest odporna na brak pliku binarnego) uruchom pełny playbook z kontrolera Ansible na stacji roboczej:

```bash
cd ~/projects/homelab-infra/ansible
ansible-playbook playbooks/site.yml --diff -K
```

Playbook w Etapie W1 wprowadza następujące zmiany:
- **Rola `security`:** jawnie ustawia `AllowTcpForwarding local` w `/etc/ssh/sshd_config.d/00-hardening.conf` (obsługa *Connect over SSH* po `tailscale0` bez zbędnego reverse forwardingu).
- **Rola `agent_host`:**
  - instaluje narzędzia deweloperskie i runtime (`gh`, `jq`, `nodejs` dla wbudowanych sidecarów, `pipx`, `python3-venv`, `ripgrep`),
  - włącza `loginctl enable-linger <admin_user>` (idempotentnie),
  - tworzy bazowe katalogi robocze (`~/projects`, `~/ops`, `~/life`),
  - wdraża wrapper `~/.local/bin/antigravity-hub-daemon` oraz unit `~/.config/systemd/user/antigravity-hub.service` z limitami cgroup RAM (`agent_host_hub_memory_high` / `agent_host_hub_memory_max`).
- **Rola `backup`:** rozszerza `backup_paths` Restic o `/home/<admin_user>` przy wykluczeniu ciężkich cache'ów, środowisk wirtualnych oraz binarki `~/.antigravity-hub-server` (ponad `220 MB` odtwarzalnego pliku).

Po pierwszym przebiegu uruchom `ansible-playbook playbooks/site.yml --diff -K` ponownie, aby potwierdzić pełną idempotencję (`changed=0`, `failed=0`), a następnie zweryfikuj wszystkie punkty z [docs/w1-verification-checklist.md](w1-verification-checklist.md).

---

## 5. Krok 3 — Dedykowany klucz SSH serwera do GitHuba (poza Ansible)

Aby serwer mógł samodzielnie pobierać i synchronizować własne repozytoria (np. prywatne repozytoria robocze czy notatki) bez kopiowania głównego klucza ze stacji roboczej:

1. Na `g580` wygeneruj osobny klucz SSH wyłącznie dla tego hosta:
   ```bash
   ssh-keygen -t ed25519 -C "homelab-g580-github" -f ~/.ssh/id_ed25519_github_g580
   ```
2. Skonfiguruj wpis `Host github.com` w `~/.ssh/config` na `g580`, wskazujący `IdentityFile ~/.ssh/id_ed25519_github_g580`.
3. Dodaj klucz publiczny (`~/.ssh/id_ed25519_github_g580.pub`) w ustawieniach SSH konta GitHub.
4. Sklonuj wybrane własne repozytoria do odpowiednich katalogów (`~/projects/`, `~/ops/`, `~/life/`) i zweryfikuj `ssh -T git@github.com`.

---

## 6. Odtwarzanie po awarii (Disaster Recovery)

W razie reinstalacji systemu na `g580`:
1. Odtwórz bazę W0 przez `bootstrap.yml` + `site.yml`. Zadanie `agent_host` bezpiecznie wykryje brak binarki `language_server` (guard `find` + `omit`), przygotuje środowisko i włączy usługę `antigravity-hub.service` bez błędu.
2. Odtwórz potrzebne dane katalogu `/home/<admin_user>` z migawki **Restic** na Google Drive (lub wykonaj jednorazowe *Connect to SSH Host* z logowaniem OAuth).
3. Po ponownym uruchomieniu `site.yml` usługa `antigravity-hub.service` wstanie automatycznie i przejmie ciągłą pracę 24/7.
