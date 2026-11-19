# Stosy Docker Compose (`stacks/`)

Ten katalog jest zarezerwowany na definicje usług aplikacyjnych wdrażanych w **Etapie W2** (pilot QoL: `Radicale`, `Vikunja`, `ntfy`).

## Obowiązkowa zasada bezpieczeństwa sieciowego

Docker domyślnie manipuluje regułami `iptables`, przez co opublikowanie portu jako `-p 8080:80` wystawia kontener na wszystkich interfejsach i **omija zaporę UFW** na karcie Wi-Fi.

Aby zachować model **Zero Trust** na Lenovo G580:
- Każdy publikowany port w `compose.yml` **musi** być przypięty wyłącznie do `127.0.0.1` (np. `"127.0.0.1:5232:5232"`) lub do konkretnego adresu IP interfejsu `tailscale0`.
- **Nigdy** nie publikujemy portów bez jawnego adresu powiązania (`0.0.0.0`).
