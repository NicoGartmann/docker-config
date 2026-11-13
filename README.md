# `docker-config` – Modulare Docker-Compose Stacks

Zentrales Repository für alle Docker-Compose-Definitionen der Homelab- & VPS-Infrastruktur (`raspberrypi`, `parents_pi`, `strato_vps`). Ausgerollt und verwaltet über [`raspi-ansible`](https://github.com/NicoGartmann/raspi-ansible).

---

## 1. Verzeichnisstruktur & Themen-Stacks

```text
.
├── .env.example                             # Vorlage für Pfad- und Secret-Variablen
├── renovate.json                            # Renovate-Konfiguration (erbt aus NicoGartmann/renovate)
├── connectivity/
│   ├── adguard.yml                          # AdGuard Home (DNS & WebUI)
│   ├── proxy.yml                            # Nginx Proxy Manager (Reverse Proxy)
│   └── twingate.yml                         # Twingate Connector (Zero-Trust VPN)
├── documents/
│   └── docker-compose.yml                   # Paperless-ngx, PostgreSQL, Redis & Scanserver
├── media/
│   └── docker-compose.yml                   # Plex Media Server & Overseerr
├── storage/
│   └── samba.yml                            # Samba Server (servercontainers/samba)
└── monitoring/
    ├── agent.yml                            # Grafana Alloy & smartctl-exporter (Push-Agent für Pis)
    ├── server.yml                           # Prometheus, Grafana & Alloy Self-Monitoring (strato_vps)
    ├── config-agent.alloy                   # Alloy Push-Konfiguration für raspberrypi & parents_pi
    ├── config-server.alloy                  # Alloy Self-Monitoring-Konfiguration für strato_vps
    ├── prometheus/
    │   └── prometheus.yml                   # Prometheus-Konfiguration (Remote-Write Receiver)
    └── grafana/
        └── provisioning/                    # Datasources, Discord-Alerting & Dashboards als Code
```

---

## 2. Zuordnung der Stacks zu den Ziel-Hosts

| Compose-Datei | Enthaltene Services | `raspberrypi` | `parents_pi` | `strato_vps` |
| :--- | :--- | :---: | :---: | :---: |
| [`connectivity/adguard.yml`](./connectivity/adguard.yml) | `adguard` | ✅ | ✅ | – |
| [`connectivity/proxy.yml`](./connectivity/proxy.yml) | `nginx-proxy-manager` | ✅ | ✅ | ✅ |
| [`connectivity/twingate.yml`](./connectivity/twingate.yml) | `twingate-connector` | ✅ | ✅ | – |
| [`documents/docker-compose.yml`](./documents/docker-compose.yml) | `paperless`, `paperless-db`, `paperless-redis`, `scanserver` | ✅ | – | – |
| [`media/docker-compose.yml`](./media/docker-compose.yml) | `plex`, `overseerr` | – | ✅ | – |
| [`storage/samba.yml`](./storage/samba.yml) | `samba` | ✅ | ✅ | – |
| [`monitoring/agent.yml`](./monitoring/agent.yml) | `alloy`, `smartctl-exporter` | ✅ | ✅ | – |
| [`monitoring/server.yml`](./monitoring/server.yml) | `prometheus`, `grafana`, `alloy` | – | – | ✅ |

---

## 3. Umgebungsvariablen (`.env`)

Die Datei `.env` ist in `.gitignore` ausgeschlossen, damit Host-spezifische Pfade und aus Ansible-Vaults generierte Secrets bei einem `git pull` nicht überschrieben werden. Alle Standard-Verzeichnispfade besitzen direkt in den Compose-Dateien sinnvolle Fallback-Werte (`${VAR:-default}`).

Eine Übersicht aller unterstützten Variablen befindet sich in [`.env.example`](./.env.example).

### Manuelle Ausführung auf einem Ziel-Host

Da die Compose-Dateien in Themen-Unterordnern liegen, wird beim manuellen Aufruf unter `/opt/services` die zentrale `.env`-Datei über `--env-file` übergeben:

```bash
cd /opt/services
docker compose --env-file /opt/services/.env -f connectivity/adguard.yml up -d
```

Im regulären Betrieb übernimmt [`raspi-ansible`](https://github.com/NicoGartmann/raspi-ansible) das Ausrollen und Aktualisieren automatisch über:
* `ansible-playbook playbooks/docker-compose.yml --ask-vault-pass` (Erst-Setup & `.env`-Aktualisierung)
* `ansible-playbook playbooks/update.yml` (Alltags-Updates ohne Vault-Passwort)

---

## 4. Automatisierte Image-Updates (Renovate)

Alle gepinnten Container-Images werden einmal wöchentlich (montags um 06:00 Uhr) durch den zentralen Renovate-Runner aus [`NicoGartmann/renovate`](https://github.com/NicoGartmann/renovate) auf neue stabile Releases geprüft:
* Die lokale [`renovate.json`](./renovate.json) bindet das zentrale Preset `github>NicoGartmann/renovate//presets/docker-compose.json` ein.
* Pro aktualisiertem Image wird ein separater Pull Request erstellt (mit automatischem Warn-Label `database-major-upgrade` bei Major-Versionssprüngen von `postgres` oder `redis`).
* Das eigene Rolling-Image `ghcr.io/nicogartmann/scanner-webui:main` (`pull_policy: always`) ist explizit von Renovate ausgenommen.
