# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) and AI coding assistants when working with code in this repository.

## What this repo is

Modular, topic-based Docker Compose configurations deployed to `/opt/services` across a fleet of Raspberry Pis and a rented Strato VPS via the companion repository [`raspi-ansible`](https://github.com/NicoGartmann/raspi-ansible):

- **`raspberrypi`** (`192.168.178.10` — Home Pi with external drive `/mnt/drive`):
  - Runs: `connectivity/adguard.yml`, `connectivity/proxy.yml`, `connectivity/twingate.yml`, `documents/docker-compose.yml`, `storage/samba.yml`, `monitoring/agent.yml`
- **`parents_pi`** (`192.168.178.46` — Off-site Pi at parents' house with external drives `/mnt/mediahdd` and `/mnt/oldhdd`):
  - Runs: `connectivity/adguard.yml`, `connectivity/proxy.yml`, `connectivity/twingate.yml`, `media/docker-compose.yml`, `storage/samba.yml`, `monitoring/agent.yml`
- **`strato_vps`** (`217.154.166.48` — Public Ubuntu VPS, central monitoring server):
  - Runs: `connectivity/proxy.yml`, `monitoring/server.yml`

Each host subscribes only to the specific Compose files listed in its `compose_files` variable inside `raspi-ansible` (`inventory/host_vars/<hostname>/vars.yml`).

## Commands & Validation

There is no CI pipeline configured. Always validate every modified or added Compose file locally against `.env.example` before committing:

```bash
# Validate a single Compose file
docker compose --env-file .env.example -f connectivity/adguard.yml config -q

# Validate all Compose files in the repository
for f in */*.yml; do
  echo "Checking $f"
  docker compose --env-file .env.example -f "$f" config -q && echo "OK: $f"
done
```

When running Compose commands manually on a target host (where the repo is cloned to `/opt/services`), always pass `--env-file /opt/services/.env` because the YAML files reside in topic subdirectories:

```bash
cd /opt/services
docker compose --env-file /opt/services/.env -f connectivity/adguard.yml up -d
```

## Architecture & Conventions (Guardrails)

1. **Top-Level `name:` Attribute (No `version:`)**:
   - Every Compose file **must** declare a unique top-level `name: <stack-name>` attribute (e.g., `name: adguard`, `name: proxy`, `name: twingate`). Because multiple YAML files share directories like `connectivity/`, omitting `name:` would cause Docker Compose to default all of them to the directory name (`connectivity`) and clash on project state.
   - Never add the obsolete `version: "3.x"` attribute.

2. **Image Version Pinning**:
   - Pin all images to **explicit, official stable release versions** (e.g., `adguard/adguardhome:v0.107.77`, `twingate/connector:1.95.0`, `lscr.io/linuxserver/plex:1.43.5`).
   - Do **not** use `:latest` for application containers (exception: custom rolling images like `ghcr.io/nicogartmann/scanner-webui:main`) and do **not** use pre-release / release-candidate (`-rc`) tags.
   - *Note*: On `ghcr.io/servercontainers/samba`, the `-r0` suffix in `smbd-only-a3.24.1-s4.23.8-r0` denotes the stable Alpine Linux APK package revision (`a` = Alpine version, `s` = Samba version, `r0` = package build 0), not a release candidate.

3. **`.env`, `.env.example`, and Variable Fallbacks**:
   - **Never commit `.env` or secrets to Git**. `.env` is ignored via `.gitignore` so that `git pull` (`force: true`) during `raspi-ansible` updates never overwrites host-specific paths or Ansible Vault secrets in `/opt/services/.env`.
   - Whenever you introduce a new environment variable in any Compose file, document it in `.env.example` and `README.md`.
   - Use `${VAR:-/default/path}` fallbacks in Compose files for non-secret paths and ports, and `${VAR}` (without plaintext secret defaults) for sensitive credentials (`TWINGATE_ACCESS_TOKEN`, `TWINGATE_REFRESH_TOKEN`, `SAMBA_PASSWORD`) injected per host via `raspi-ansible` Vaults (`vaults/<hostname>.yml`).

4. **Storage Paths & Production Data Protection**:
   - **App Configurations (`/opt/<service>`)**: All service configuration directories reside outside the Git repository under `/opt/<service>` (e.g., `/opt/adguard`, `/opt/nginx-proxy-manager`, `/opt/plex`, `/opt/overseerr`). Never commit runtime data directories or `.gitkeep` placeholders into this repository.
   - **Protected Production Paths (Do Not Alter)**:
     - On `raspberrypi`: Paperless-ngx data (`${PAPERLESS_DATA:-/mnt/drive/paperless}/{pgdata,data,media,consume,export}`) and its internal Postgres credentials (`paperless`) are actively used in production.
     - On `parents_pi`: Plex media library mounts (`/mnt/mediahdd/Serien:/tv`, `/mnt/mediahdd/Filme:/movies`, `/mnt/oldhdd/Bilder:/pictures`) are actively used in production.

5. **Language & Commit Style**:
   - Documentation (`README.md`, design docs in `docs/`) and inline explanations are written in German; `CLAUDE.md` is kept in English.
   - Use Conventional Commits with ticket scope when applicable (e.g., `feat(SUP-146): ...`, `refactor(SUP-146): ...`, `docs(SUP-146): ...`).
