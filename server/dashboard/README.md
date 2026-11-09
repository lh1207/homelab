# Homepage dashboard

Homepage 2.6.0 is live on CT105 at `/root/services/optiplex/dashboard`. Its 20 cards are arranged in five groups and cover the 16 existing Docker containers, the Home Assistant VM, Proxmox, the Mac-only Reddit KB, and an undeployed Staging placeholder. MariaDB, PostgreSQL, Redis, and Caddy are native-protocol or infrastructure services; their cards link to Portainer or pgAdmin where useful rather than inventing web interfaces. Credentials are not stored in this repository.

## Access and configuration

Open `https://docker.homepage` from a client using the homelab Pi-hole DNS server. Its local DNS record points to the Docker LXC, where Caddy proxies HTTPS to Homepage. Clients must trust the server Caddy CA. Homepage allows the `docker.homepage` and legacy `homepage.localhost` Host headers; requests made directly to the container IP are intentionally rejected and should go through the proxy. The Mac proxy continues to support `https://homepage.localhost`.

The Compose stack listens on port `3004`, binding both `127.0.0.1` and `DASHBOARD_BIND_IP`. The tracked Compose file, `.env.example`, and YAML templates in `config/` are the configuration source. The private `.env` is kept only in the deployment directory at `/root/services/optiplex/dashboard`. It supplies `DASHBOARD_BIND_IP`, `HOMEPAGE_VAR_PROXMOX_URL`, and `HOMEPAGE_VAR_HOMEASSISTANT_URL`. The Compose file maps `host.docker.internal` to the host gateway; HTTP checks use that name to reach backend services.

The Home Assistant card links to the user-supplied `homeassistant.local:8123` URL. That hostname is not resolvable from this server environment, so the card has no `siteMonitor` check. Staging is not deployed; its card is a placeholder. Docker service cards use `https://docker.<service>` names; internal `siteMonitor` endpoints continue to check backend services directly. See the root README for the Pi-hole records and proxy setup.

## Deploy and roll back

Run these commands on CT105 from the deployment directory. `down` stops only this Homepage Compose stack and keeps its files and data; do not add volume-removal flags.

```sh
cd /root/services/optiplex/dashboard
docker compose up -d
docker compose ps
docker compose logs --tail=100 homepage
```

To roll back or stop Homepage:

```sh
cd /root/services/optiplex/dashboard
docker compose down
```

Homepage logs go to stdout. The empty Docker and Kubernetes configuration files disable discovery integrations; no Docker socket or service API keys are mounted.

## Status and verification

All 12 configured `siteMonitor` endpoints returned HTTP 200 during deployment verification, and the Homepage container reported healthy. Mac access through trusted HTTPS returned 200, with the expected redirect returning 308. `siteMonitor` confirms that an HTTP endpoint responds; it does not verify the full application or its dependencies. Native-protocol services are not covered by those checks.

## Why Homepage

For this setup, Homepage provides a maintainable file-based service directory, HTTP reachability indicators, and integrations in a small Compose stack. User reports praise its integrations and status checks, while noting some initial YAML setup ([r/homelab, June 4, 2026](https://www.reddit.com/r/homelab/comments/1twpczp/dashboards/); [r/selfhosted, August 11, 2024](https://www.reddit.com/r/selfhosted/comments/1epfvr4/what_dashboard_do_you_all_use/)). Homarr offers visual editing, though some users report higher memory use ([r/selfhosted, January 21, 2026](https://www.reddit.com/r/selfhosted/comments/1qj359n/my_homepage_dashboard/)).
