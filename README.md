# Homelab

Docker Compose definitions for my home lab, split by where each stack runs. Every
stack is a self-contained `compose.yml` you can bring up independently with
`docker compose up -d`.

## Topology

Live inspection in Portainer on 2026-10-18 confirmed that BookStack, Pi-hole,
pgAdmin, and Ollama currently run as containers on the Docker LXC. The layout
below describes the repository's split; it does not migrate those running services.
The proxy upstream variables support either placement. The Mac proxy's ignored
`.env` is configured for the destinations verified in Portainer.

| Host | Address | Verified role |
|------|---------|---------------|
| Mac (`workstation`) | `<mac-ip>` | OrbStack runs the local Caddy proxy. |
| Proxmox host (`gobbles`) | `<proxmox-ip>` | Six LXCs and one Home Assistant VM; five migrated LXCs are stopped in Archive. |
| Docker LXC | `<docker-lxc-ip>` | Portainer reports 16 running containers, listed below. |

Proxmox retains standalone Pi-hole (CT 100), Jellyfin (CT 102), Homebridge
(CT 103), Uptime Kuma (CT 104), and BookStack (CT 106). All five source LXCs
are stopped in Archive with boot startup disabled.
CT 105 is the Docker host; VM 101 runs Home Assistant. All six LXCs are
unprivileged and use local ZFS root storage.

The intended destination is Docker wherever feasible. Existing standalone data
must be migrated and verified before the source instance is retired. Preserve
source LXCs and existing Docker volumes for rollback.

> Real LAN addresses are not committed. Each stack that needs one reads it from its
> git-ignored `.env` (see the relevant `.env.example`); placeholders like `<docker-lxc-ip>`
> below stand in for them.

### How the pieces talk to each other

- **Reverse proxy:** A [Caddy](https://caddyserver.com/) instance on each Docker host
  serves friendly `https://*.localhost` hostnames and proxies them to the right service.
  Caddy issues and renews baseline certificates with its local certificate authority.
- **AI:** Open WebUI and Ollama both run on the Docker LXC and share the `ai`
  Docker network. Open WebUI now uses the verified `http://ollama:11434`
  backend in both its external Compose source and saved application setting.
  `nomic-embed-text:latest` is installed in Docker. Its four manifest-referenced
  blob hashes were validated, and a 768-dimension embedding request succeeded.
  Open WebUI's backend tags are reachable from its container; an authenticated
  WebUI workflow has not been verified. No chat model has been selected.
- **Database:** pgAdmin and Postgres both run on the Docker LXC, on separate
  Docker networks. A successful pgAdmin-to-Postgres connection is not yet verified.

## Repository layout

```
.
├── workstation/        # Repository grouping; live pgAdmin runs on the Docker LXC
│   ├── dev/            # pgAdmin
│   └── networking/     # Caddy reverse proxy + Caddyfile
├── server/             # Stacks that run in the Docker LXC (<docker-lxc-ip>)
│   ├── ai/            # Open WebUI
│   ├── databases/      # Postgres + Redis
│   ├── staging/        # Staging static host (FTP + nginx) for the production site
│   ├── home/           # Homebridge
│   ├── media/          # Jellyfin
│   ├── monitoring/     # Grafana + Portainer + Uptime Kuma
│   ├── networking/     # Caddy reverse proxy + Caddyfile
│   └── services/       # Vaultwarden + Gitea
└── reference/          # Reference definitions; live services also use Docker
    ├── ollama/         # Ollama (verified on the Docker LXC)
    ├── dns/            # Pi-hole (verified on the Docker LXC)
    └── wiki/           # BookStack (verified on the Docker LXC)
```

## Services

The service dashboard is [Homepage](https://homepage.localhost), deployed on the
Docker LXC from `/root/services/optiplex/dashboard`. Its 20 cards cover the Docker
services, Home Assistant, Proxmox, the Mac Reddit knowledge base, and the undeployed
staging definition. Twelve web services have HTTP reachability indicators; native
services link to their management tools. See [dashboard configuration and user
testimony](server/dashboard/README.md). Home Assistant uses the user-supplied
`http://homeassistant.local:8123` URL. The existing client-local hostname and CA
trust requirements below also apply to the dashboard.

### `workstation/` — repository definitions

| Stack        | Service | Port   | URL                  | Notes                              |
|--------------|---------|--------|----------------------|------------------------------------|
| `dev`        | pgAdmin | `5050` | `pgadmin.localhost`  | Live instance runs on the Docker LXC; DB connection unverified |
| `networking` | Caddy   | host   | —                    | Serves all `*.localhost` names below |

### `server/` — Docker LXC definitions

| Stack        | Service     | Port(s)        | URL                   | Notes                          |
|--------------|-------------|----------------|-----------------------|--------------------------------|
| `ai`         | Open WebUI  | `3000`         | `ai.localhost`        | Configured inference endpoint needs verification |
| `databases`  | Postgres    | `5432`         | —                     | LAN-exposed for pgAdmin        |
| `databases`  | Redis       | `6379`         | —                     |                                |
| `staging`    | FTP + nginx | `8090`, FTP    | `staging.localhost`   | Static staging host for the production site |
| `home`       | Homebridge  | `8581`         | `homebridge.localhost`| Host networking (HomeKit mDNS) |
| `media`      | Jellyfin    | `8096`         | `jellyfin.localhost`  | Migrated; read-only router media, background scan in progress |
| `monitoring` | Grafana     | `3002`         | `grafana.localhost`   |                                |
| `monitoring` | Portainer   | `9000`         | `portainer.localhost` |                                |
| `monitoring` | Uptime Kuma | `3003`         | `uptime.localhost`    |                                |
| `networking` | Caddy       | host           | —                     | Reverse proxy for this host    |
| `services`   | Vaultwarden | `8080`         | `vault.localhost`     | Password manager               |
| `services`   | Gitea       | `3001`, `2222` | `git.localhost`       | Git server (`2222` = SSH)      |

### Additional services verified on the Docker LXC

| Service | Live stack | Web port | Repository definition |
|---------|------------|----------|-----------------------|
| Ollama | `ollama` | `11434` | `reference/ollama` |
| Pi-hole | `dns` | `8888` | `reference/dns` |
| BookStack + MariaDB | `wiki` | `6875` (BookStack) | `reference/wiki` |
| pgAdmin | `dev` | `5050` | `workstation/dev` |

The live Compose sources are external to this checkout. Labels locate most at
`/root/services/optiplex/<stack>/compose.yml`; Caddy, pgAdmin, and Ollama instead
reference `/root/services/local/<stack>/compose.yml` (`networking`, `dev`, and
`ollama`). Editing this checkout does not update those files.

Both proxies have routes for every web service above, including Ollama, pgAdmin,
and staging. Postgres, Redis, FTP, Gitea SSH, and HomeKit discovery keep their
native protocols and ports; Caddy's web HTTPS routes do not replace those.

### Baseline HTTPS

Copy each networking stack's `.env.example` to `.env` and replace all placeholders
before starting Caddy. Both stacks require `BOOKSTACK_UPSTREAM`,
`PIHOLE_UPSTREAM`, `PGADMIN_UPSTREAM`, and `OLLAMA_UPSTREAM`: their actual web URLs
(including scheme and port), wherever each service runs. The Mac proxy also
requires `DOCKER_LXC_IP`. Missing values fail Compose validation.

All web routes use HTTPS, and HTTP requests redirect to HTTPS. TLS terminates at
Caddy; the existing HTTP backends continue to work behind it. No service currently
has an HTTP-only exception. If a service proves incompatible, give only its site
an explicit `http://` address in the relevant Caddyfile.

Each Caddy instance has its own local CA, persisted in its `caddy_data` volume.
After starting a networking stack, export **only the public root certificate**:

```sh
docker compose cp caddy:/data/caddy/pki/authorities/local/root.crt /tmp/homelab-caddy-root.crt
```

Import that certificate into each client's trusted root store. On macOS, import
it with Keychain Access and set it to Always Trust. Repeat for the other proxy's
root if clients use both. Do not delete `caddy_data`, export CA private keys, or
rely on clicking through browser certificate errors. See
[Caddy's local HTTPS documentation](https://caddyserver.com/docs/automatic-https#local-https).

`*.localhost` resolves to the client itself, so these names work on the Mac through
its local proxy; they are not LAN DNS names for phones or other computers. For LAN
clients, configure a real LAN DNS name (for example under `home.arpa`) pointing to
the server proxy, then add that name to the matching Caddy site and trust its CA.
See [RFC 6761](https://www.rfc-editor.org/rfc/rfc6761).

Services with a configured public/base URL must use their HTTPS route (especially
BookStack's `APP_URL`, Gitea's `ROOT_URL`, and Vaultwarden's `DOMAIN` if configured).
Update those in their deployed application configuration; the external Compose
sources are not synchronized by editing this checkout. Recreate Caddy after editing its config or `.env`
with `docker compose up -d --force-recreate caddy` (`admin off` disables reloads).

The Mac proxy is deployed from `workstation/networking` with the Compose project
name `networking`, preserving its existing certificate volumes. Use
`docker compose -p networking up -d --force-recreate caddy` from that folder.
Its CA is trusted in this Mac's login keychain; other clients require their own
trust setup.

BookStack's Docker environment was repaired by renaming `DB_USER` to
`DB_USERNAME` and `DB_PASS` to `DB_PASSWORD`, preserving the existing values and
data volume, and setting `APP_URL=https://bookstack.localhost`. The external
Compose source is `/root/services/optiplex/wiki/compose.yml`; the repository's
reference is also corrected. The initial repair preserved credentials and
mounts, with a private backup under `/root/homelab-review-backups`. The subsequent
library migration uses the source encryption key and new volumes, as recorded
below.
Its SSH host fingerprint was verified via Proxmox. The user authorized the Mac's
existing SSH key for root on CT 105, and strict SSH access is verified.


Staging has a proxy route but no deployed backend yet. Gitea's HTTPS route opens
its initial setup page; application setup remains separate from proxy setup.

## Live review status — 2026-10-18

The first Docker pass inspected all 16 containers in Portainer: all were running,
five reported healthy, all used `unless-stopped`, and all had zero recorded
restarts. Persistent mounts were present for each service. Portainer initially
listed 20 volumes and 11 networks; migration adds new volumes while retaining the
originals. The unused `ai_ollama_data` volume is retained.

The user confirmed that Docker is the intended destination for standalone
services. These migrations have been applied through the authenticated Proxmox
console:

| Service | Source | Docker destination | Verification |
|---------|--------|--------------------|--------------|
| BookStack | CT 106, v25.11.6 | v26.03.4; `wiki_bookstack_migrated_20261018` and `wiki_bookstack_db_migrated_20261018` | 7 books, 3 shelves, 10 active pages plus one deleted page; core records and users match the source; original-account sign-in and library verified; HTTPS responds after app and DB restart. |
| Uptime Kuma | CT 104, v2.2.1 | v2.2.1; `monitoring_uptimekuma_migrated_20261018` | Database integrity passed; 5 monitors and 1 user retained; Pi-hole monitor points to Docker; HTTPS responds after restart. |
| Homebridge | CT 103 | `home_homebridge_migrated_20261018` | Config, pairings, UI accounts, and plugins restored; both plugins loaded without recent logged errors; device control remains unverified. |
| Pi-hole | CT 100, FTL v6.4.1 | FTL v6.6.1; `dns_pihole_migrated_20261018` and `dns_pihole_dnsmasq_migrated_20261018` | Gravity database integrity passed; source settings restored; UDP and TCP DNS queries and HTTPS admin route respond. Source LXC is shut down and archived; clients using its old address need the Docker address or a fallback. |
| Jellyfin | CT 102, v10.11.5 | v10.11.8; four `media_jellyfin_*_migrated_20261019` volumes | 2 users and 5 libraries restored; retained watched-state keys verified; authenticated direct streaming and restart passed; cold backups saved. Metadata scan continues. |

BookStack includes the original encryption key, custom environment settings, and
uploaded-file directories. Its source LXC was put in maintenance mode, then shut
down and archived. The original
May 4 Docker volumes (`wiki_bookstack_data`, `wiki_bookstack_db_data`) remain
untouched for rollback. Their saved root password did not authenticate, so the
migration used fresh volumes rather than resetting the original database.

The standalone Pi-hole, Jellyfin, Uptime Kuma, Homebridge, and BookStack LXCs
(CT 100, 102, 104, 103, and 106) are stopped in the `Archive` resource pool and have
boot startup disabled. Pool View shows them beneath a collapsed Archive group.
Their disks and original Docker volumes are retained. The old Homebridge and
Uptime Kuma service startup settings are also disabled to prevent duplicates.
The user confirmed that fallback services are available during cutover.

The Compose templates select the migrated volumes explicitly. The
`BOOKSTACK_DATA_VOLUME`, `BOOKSTACK_DB_VOLUME`, `UPTIMEKUMA_DATA_VOLUME`,
`HOMEBRIDGE_DATA_VOLUME`, `PIHOLE_DATA_VOLUME`, and `PIHOLE_DNSMASQ_VOLUME`
overrides allow a deliberate alternate restore target.
Do not point a deployment at an old or empty volume unintentionally.

A separate route check verified trusted HTTPS and HTTP-to-HTTPS redirects for all
12 deployed web services. Staging redirects to HTTPS but returns 502 because its
backend is absent. The Mac Caddy container retains `networking_caddy_data` and
`networking_caddy_config` and mounts this checkout's Caddyfile read-only. The
server Caddy uses an external Caddyfile, now corrected to enable local HTTPS
and redirects and to reach Docker pgAdmin and Ollama through the verified host
loopback ports. All 12 routes passed HTTPS checks through both proxies. The Mac
trusts its own proxy CA; the server checks explicitly used the server CA.

Proxmox has no configured backup jobs, no backups on its configured backup
storage, and no ZFS snapshots. The requested backup destination is the Mac;
private migration backups are staged under
`/root/homelab-migration-20261018` on Proxmox, with working copies and Compose
backups under `/root/homelab-review-backups` in CT 105. The Mac destination is
`~/Documents/Backups/homelab/2026-10-18`. The Mac now holds migration archives,
a consistent backup of all Docker volumes and deployment sources, and the Mac
proxy CA/configuration backup. All 35 checksummed files matched; disposable
SQLite restore checks passed for Uptime Kuma, Pi-hole, and WebUI. A separate Mac
Ollama model archive is also saved and was transferred to CT 105. A model delta
backup created after installation is saved on the Mac at
`~/Documents/Backups/homelab/2026-10-18/ollama-model-install-20261019`; its
`SHA256SUMS` check passed. The full Docker snapshot predates this change.
This is a manual dated backup; no recurring job or retention policy is configured. Never print or commit the archives: they contain credentials and private
application data.

Standalone web endpoints responded for all five duplicate services. Standalone
Pi-hole answered DNS over UDP and TCP. Docker Pi-hole initially rejected those
queries in `LOCAL` mode; the migrated instance now answers using `ALL` mode.
The WebUI-configured Ollama LAN endpoint refused connections. Both its external
Compose source and saved application setting now use `http://ollama:11434`,
verified reachable from WebUI over the shared Docker network. The staged
`nomic-embed-text:latest` model is now installed; all four referenced blob hashes
validated, a 768-dimension embedding request passed, and backend tags are
reachable from the WebUI container. No authenticated WebUI workflow or chat
inference is verified. Jellyfin now uses a read-only SMB 2.0 router mount at
`/mnt/router-share`; migration is complete and its background scan is running.
The old CT 102 had failed RPC pipe filesystem and message-of-the-day units; it
is now stopped and archived.

Jellyfin migrated from the 10.11.5 LXC to the existing official Docker 10.11.8
image, now pinned. Separate versioned volumes preserve the native data, config,
cache, and log paths. The initial restored database passed integrity checks:
2 users, 42 UserData rows, 4,856 base items, and 48 permission rows. Users,
UserData, and Permissions row hashes matched the source before API checks.
The original Docker volumes and source LXC disks remain available for rollback.

The router accepts SMB 2.0; corrected saved credentials passed a separate
read-only mount test. CT 105 restarted successfully with a read-only router
bind. A host reboot has not been tested. The container startup guard refuses
to start Jellyfin without the real CIFS mount. The retired `/mnt/media-share`
roots were removed through Jellyfin's API; each of five libraries now has one
valid router media root. An authenticated HTTP 206 direct-stream response
matched the first 1,024 bytes of an actual video before and after restart.
Visual playback and transcoding remain untested; metadata scanning is ongoing.

At final restart, all 26 retained source watched-state keys matched. The other
16 source rows refer to six unavailable router media paths; all 42 source
watched-state records are preserved in a private export and the cold source
backup. Cold source and Docker migration backups are saved under the dated Mac
backup's `jellyfin-migration-20261019` directory. All 15 backup checksums passed;
a disposable SQLite restore from the final archive passed integrity and
retained watched-state checks. CT 102 is stopped in Archive,
with its service and boot startup disabled and its disks retained. Five source
LXCs are now archived; Docker and Home Assistant remain running.
Client DNS cutover and other application workflows remain unresolved. Gitea's
initial setup and staging's intended content still require decisions. Do not
seed them without that direction.

Second-pass inspection found all 16 containers running and matching their
external Compose sources for image, configured environment values, and mounts.
Migrated Uptime Kuma, Homebridge, and Pi-hole containers were restarted; their
web routes and Pi-hole UDP/TCP DNS checks passed afterward. BookStack original
account sign-in and the restored seven-book library were verified in Chrome.
Jellyfin source and Docker state backups are created; the restored Docker
library is scanning the verified router mount. The approved Mac SSH key now works with strict
host-key verification. The full backup stopped and restarted all 16 containers;
all returned running and all five healthchecks passed. Afterward, database counts,
Pi-hole password hash/upstreams, Homebridge identity, WebUI backend settings,
HTTPS routes, and UDP/TCP DNS checks passed. Ollama model installation is
complete. The router share is confirmed as the media location. Focused
negotiation tests found that the router accepts SMB 2.0 and rejects SMB 2.1 and
SMB 3; a separate read-only SMB 2.0 mount now works with the saved credentials.
The automatic mount after a host reboot remains untested.

## Deploying a stack

```sh
cd server/media      # or any stack folder
docker compose up -d
docker compose logs -f
```

Each folder is independent, so you only bring up what you need.

### Prerequisites & gotchas

- **Staging host** (`server/staging`) reads `FTP_USER_PASS` / `OPTIPLEX_IP` from a `.env`
  file in that folder (see the `${VAR}` references in its `compose.yml`).
- **Jellyfin media** (`server/media`) requires the real SMB share mounted at
  `/mnt/router-share` on the Docker host before Compose starts. The container
  binds that path read-only and refuses to start if it is not a CIFS mount.
  The router currently accepts SMB 2.0 only; automatic mounting after reboot
  has not been tested.
- **Pi-hole on port 53** requires disabling the host's `systemd-resolved` DNS stub
  first. The reference compose publishes DNS and web UI on `0.0.0.0` and sets
  Pi-hole's Docker DNS listening mode to `all`; see comments in
  `reference/dns/compose.yml` when managing the deployed Docker configuration.
- **Homebridge** uses host networking for HomeKit/Bonjour discovery; only its web UI
  is proxied.

## Secrets

No credentials are committed. Each stack that needs a secret reads it from a per-stack
`.env` file (git-ignored) via `${VAR}` references in its `compose.yml`. A committed
`.env.example` documents the required variables.

To bring up a stack:

```sh
cd server/databases        # any stack with a .env.example
cp .env.example .env        # then edit .env and set a strong value
docker compose up -d
```

`.env` is git-ignored, so it does **not** travel with `git push`/`pull` — create it on
each host that runs the stack. Stacks with secrets: `server/databases`, `server/staging`,
`workstation/dev`, `reference/wiki`, `reference/dns`. Also configure the address
variables in `server/ai/.env.example`, `server/networking/.env.example`, and
`workstation/networking/.env.example`
before starting those stacks.

## Security / secret hygiene

This repo is public, so it runs three independent layers to keep secrets and known
vulnerabilities out of the tree:

1. **Local pre-commit hook** ([`.pre-commit-config.yaml`](.pre-commit-config.yaml)) —
   [gitleaks](https://github.com/gitleaks/gitleaks) scans the staged diff and a local
   hook hard-fails if a real `.env` is staged. Rejects the commit before it exists.
2. **GitHub Push Protection + Secret Scanning** (repo settings, free for public repos) —
   GitHub's server rejects any `git push` containing a recognized secret pattern.
3. **CI workflow** ([`.github/workflows/security.yml`](.github/workflows/security.yml)) —
   on every push/PR: gitleaks over the full history, an `env-guard` check that no real
   `.env` is tracked, and a [KICS](https://kics.io/) misconfiguration scan of the compose
   files (report-only). Weekly (and on demand) it also runs a [Trivy](https://trivy.dev/)
   CVE scan of every container image. Findings surface in the repo **Security** tab.

### One-time local setup

```sh
brew install pre-commit gitleaks trivy
pre-commit install          # activate the commit-time hook
pre-commit run --all-files  # optional: scan the whole repo now
```

gitleaks false positives on the committed templates are suppressed in
[`.gitleaks.toml`](.gitleaks.toml).
