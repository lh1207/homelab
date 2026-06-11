# Homelab

Docker Compose definitions for my home lab, split by where each stack runs. Every
stack is a self-contained `compose.yml` you can bring up independently with
`docker compose up -d`.

## Topology

| Host            | Address           | Role                                                        |
|-----------------|-------------------|-------------------------------------------------------------|
| Mac (`workstation`) | `<mac-ip>`    | Daily-driver Mac. Runs a couple of Docker stacks + baremetal Ollama. |
| Proxmox host    | `<proxmox-ip>`    | Hypervisor. Hosts the LXCs below (created via community helper scripts). |
| Docker LXC      | `<docker-lxc-ip>` | The main container host. Runs everything under `server/`.   |
| Pi-hole LXC     | _(own LXC)_       | Network DNS / ad-blocking. Standalone LXC, not Docker.      |
| BookStack LXC   | _(own LXC)_       | Wiki. Standalone LXC, not Docker.                           |

> Real LAN addresses are not committed. Each stack that needs one reads it from its
> git-ignored `.env` (see the relevant `.env.example`); placeholders like `<docker-lxc-ip>`
> below stand in for them.

### How the pieces talk to each other

- **Reverse proxy:** A [Caddy](https://caddyserver.com/) instance on each Docker host
  serves friendly `*.localhost` hostnames and proxies them to the right service.
- **AI:** Open WebUI (Docker LXC) talks to **Ollama running baremetal on the Mac**
  at `<mac-ip>:11434` (Ollama is LAN-exposed for this reason).
  If Ollama and Open WebUI are ever moved onto the same Linux Docker host, use
  `http://host.docker.internal:11434`; the Open WebUI compose file maps that name
  to Docker's host gateway.
- **Database:** pgAdmin (Mac) connects to **Postgres in the Docker LXC** at
  `<docker-lxc-ip>:5432`.

## Repository layout

```
.
├── workstation/        # Stacks that run on the Mac (<mac-ip>)
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
└── reference/          # NOT deployed via Docker — see reference/README.md
    ├── ollama/         # Ollama (actually runs baremetal on the Mac)
    ├── dns/            # Pi-hole (actually runs as its own Proxmox LXC)
    └── wiki/           # BookStack (actually runs as its own Proxmox LXC)
```

## Services

### `workstation/` — Mac, `<mac-ip>`

| Stack        | Service | Port   | URL                  | Notes                              |
|--------------|---------|--------|----------------------|------------------------------------|
| `dev`        | pgAdmin | `5050` | `pgadmin.localhost`  | Connects to Postgres in the Docker LXC |
| `networking` | Caddy   | host   | —                    | Serves all `*.localhost` names below |
| _(baremetal)_ | Ollama | `11434`| `ollama.localhost`   | Runs on the Mac directly, not Docker |

### `server/` — Docker LXC, `<docker-lxc-ip>`

| Stack        | Service     | Port(s)        | URL                   | Notes                          |
|--------------|-------------|----------------|-----------------------|--------------------------------|
| `ai`         | Open WebUI  | `3000`         | `ai.localhost`        | Frontend for Ollama on the Mac |
| `databases`  | Postgres    | `5432`         | —                     | LAN-exposed for pgAdmin        |
| `databases`  | Redis       | `6379`         | —                     |                                |
| `staging`    | FTP + nginx | `8090`, FTP    | `staging.localhost`   | Static staging host for the production site |
| `home`       | Homebridge  | `8581`         | `homebridge.localhost`| Host networking (HomeKit mDNS) |
| `media`      | Jellyfin    | `8096`         | `jellyfin.localhost`  | Needs SMB media mount (below)  |
| `monitoring` | Grafana     | `3002`         | `grafana.localhost`   |                                |
| `monitoring` | Portainer   | `9000`         | `portainer.localhost` |                                |
| `monitoring` | Uptime Kuma | `3003`         | `uptime.localhost`    |                                |
| `networking` | Caddy       | host           | —                     | Reverse proxy for this host    |
| `services`   | Vaultwarden | `8080`         | `vault.localhost`     | Password manager               |
| `services`   | Gitea       | `3001`, `2222` | `git.localhost`       | Git server (`2222` = SSH)      |

### Runs outside Docker (`reference/`)

| Service   | Runtime                  | URL                 |
|-----------|--------------------------|---------------------|
| Ollama    | Baremetal on the Mac     | `ollama.localhost`  |
| Pi-hole   | Its own Proxmox LXC      | `pihole.localhost`  |
| BookStack | Its own Proxmox LXC      | `bookstack.localhost` |

> The `*.localhost` routes for Pi-hole and BookStack still point at placeholder
> addresses in the Caddyfiles (marked with `TODO`). Update them to each LXC's IP.

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
- **Jellyfin media** (`server/media`) expects an SMB share mounted at `/mnt/media`
  on the Docker host. Mount instructions are in the compose file comments.
- **Pi-hole on port 53** requires disabling the host's `systemd-resolved` DNS stub
  first. The reference compose publishes DNS and web UI on `0.0.0.0` and sets
  Pi-hole's Docker DNS listening mode to `all`; see comments in
  `reference/dns/compose.yml` if you ever containerize it.
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
`workstation/dev`, `reference/wiki`, `reference/dns`.
