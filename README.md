# `homelab-infra`

Deklaratywna konfiguracja domowego serwera **Lenovo G580** (Ubuntu Server 24.04 LTS) zarządzana za pomocą **Ansible** i **Docker Compose**.

Projekt stanowi fundament infrastruktury **Etapu W0** pod lekkie usługi produkcyjne 24/7 (`PROD`) działające wyłącznie po Wi-Fi i bez dostępu administracyjnego do routera.

---

## Sprzęt i ograniczenia (stan Etapu W0)

- **Serwer:** Lenovo G580 — Intel Core i7-3720QM (4C/8T), 8 GB DDR3 (`2x 4 GB`), **250 GB SATA SSD** (docelowy `1 TB HDD` dojdzie później w zatoce caddy jako osobny punkt montowania `/mnt/hdd`), karta Wi-Fi Intel Centrino Advanced-N 6205, wyłączona NVIDIA GT 710M.
- **Sieć:** wyłącznie Wi-Fi, brak przekierowania portów i statycznego DHCP na routerze -> pełna komunikacja prywatna przez **Tailscale (`MagicDNS`)**.
- **Zasilanie:** aktualnie bez sprawnej baterii (planowany zakup nowej baterii w roli UPS). Brak LUKS na dysku oraz brak automatycznych restartów po aktualizacjach minimalizują ryzyko zablokowanego startu po zaniku zasilania.

---

## Architektura Etapu W0

| Obszar | Rozwiązanie |
| :--- | :--- |
| **System bazowy** | Ubuntu Server 24.04 LTS (`g580`, użytkownik `konzi`, strefa `Europe/Warsaw`) |
| **Dyski (SSD 250 GB)** | LVM bez LUKS (`ubuntu-vg`): `60G` `/` (instalator), `50G` `/var/lib/docker`, `60G` `/srv`, ~`50G` rezerwy w VG |
| **Laptop jako serwer** | Zamknięta klapa ignorowana (`systemd-logind`), `consoleblank=60`, `iwlwifi power_save=0`, `thermald`, `zram` (`min(ram / 2, 4096)`, `zstd`) |
| **Sieć i hardening** | **Zero Trust po Wi-Fi:** UFW `deny incoming` w LAN, utwardzone OpenSSH (`00-hardening.conf`, tylko klucze, bez roota) wyłącznie na `tailscale0` |
| **Aktualizacje** | `unattended-upgrades` dla łatek bezpieczeństwa, **bez** automatycznego rebootu |
| **Kontenery** | Oficjalne repo Docker CE + `docker-compose-plugin`, rotacja logów (`10m` x 3), `live-restore: true` |
| **Backupy** | `Restic` + `rclone` na Google Drive (`homelab-backups/g580/restic`), szyfrowanie po stronie klienta, codzienny `systemd timer` (`03:30`) |
| **Sekrety (W0)** | Zero sekretów w repo (`.gitignore` + `.example`); docelowa migracja na `SOPS + age` w Etapie W1/W2 |

---

## Struktura repozytorium

```text
homelab-infra/
├── README.md
├── .gitignore
├── docs/
│   ├── w0-ubuntu-install-runbook.md     # Ręczna instalacja Ubuntu 24.04 LTS na G580
│   └── w0-verification-checklist.md     # Testy odbiorcze po wdrożeniu Etapu W0
├── ansible/
│   ├── ansible.cfg
│   ├── requirements.yml
│   ├── files/.gitkeep                   # Miejsce na lokalny, ignorowany rclone.conf
│   ├── inventory/
│   │   ├── hosts.yml.example
│   │   └── group_vars/all/
│   │       ├── vars.yml
│   │       └── secrets.yml.example
│   ├── playbooks/
│   │   ├── bootstrap.yml                # Faza 1 (po LAN): base + storage + tailscale
│   │   └── site.yml                     # Faza 2 i utrzymanie (po Tailscale): pełny stan + hardening + Docker + backup
│   └── roles/
│       ├── base/
│       ├── storage/
│       ├── tailscale/
│       ├── security/
│       ├── docker/
│       └── backup/
└── stacks/
    └── README.md                        # Przyszłe stosy Docker Compose (Etap W2)
```

---

## Szybki start (Etap W0 w 2 fazach)

1. Zainstaluj bazowy system na Lenovo G580 zgodnie z [docs/w0-ubuntu-install-runbook.md](docs/w0-ubuntu-install-runbook.md).
2. Na stacji roboczej (Arch PC) przygotuj lokalne pliki nieśledzone przez Git na bazie szablonów `.example`:
   - `ansible/inventory/hosts.yml`
   - `ansible/inventory/group_vars/all/secrets.yml`
   - `ansible/files/rclone.conf` (token OAuth dla Google Drive z własnym klientem `Desktop app` z Google Cloud i zakresem `drive.file`)
3. **Faza 1 (po tymczasowym IP w LAN):** uruchom `bootstrap.yml`, aby skonfigurować system bazowy, wolumeny LVM i zainstalować Tailscale:
   ```bash
   cd ansible
   ansible-playbook playbooks/bootstrap.yml -e ansible_host=<LAN_IP> -K
   ```
4. **Spięcie tunelu:** zaloguj się na G580, uruchom `sudo tailscale up`, autoryzuj węzeł i zweryfikuj `ssh g580` z Arch PC.
5. **Faza 2 (wyłącznie po Tailscale):** uruchom pełny, idempotentny playbook `site.yml` (utwardza SSH/UFW, instaluje Docker CE i konfiguruje backupy Restic):
   ```bash
   ansible-playbook playbooks/site.yml -K
   ```
6. Zweryfikuj wszystkie punkty z [docs/w0-verification-checklist.md](docs/w0-verification-checklist.md) (w tym test odtwarzania pliku przez Restic).

---

## Autorstwo

- **KonZi (`Konrad`)**: architektura systemu, decyzje projektowe (LVM bez LUKS, Zero Trust na `tailscale0`, dwufazowy bootstrap, strategia backupów), przeglądy kodu, wdrożenie na sprzęcie i testy odbiorcze.
- **Antigravity**: wygenerowanie struktury ról Ansible, szablonów konfiguracyjnych i roboczej dokumentacji pod nadzorem i według specyfikacji KonZiego.
