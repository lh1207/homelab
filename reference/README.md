# reference/ — reference Compose definitions

Live Portainer inspection on 2026-10-18 confirmed Docker LXC containers for all
three services below. These files are reference definitions; the live Compose
sources are external to this checkout and may differ.

| Folder | Service | Verified live stack | External Compose source |
|--------|---------|---------------------|-------------------------|
| `ollama` | Ollama | `ollama` | `/root/services/local/ollama/compose.yml` |
| `dns` | Pi-hole | `dns` | `/root/services/optiplex/dns/compose.yml` |
| `wiki` | BookStack + MariaDB | `wiki` | `/root/services/optiplex/wiki/compose.yml` |

Proxmox also runs standalone Pi-hole (CT 100) and BookStack (CT 106). The intended
destination is Docker; preserve standalone data during migration. The standalone
BookStack database contains 7 books, 3 shelves, and 10 active pages.

BookStack is migrated into new Docker volumes with its library, uploaded files,
custom settings, and original encryption key. Core records and users match the
source; original-account sign-in and the seven-book library are verified. The original Docker volumes and source
LXC remain available for rollback; the source is shut down in the Archive pool.
Pi-hole settings are also migrated into new Docker volumes, and its old LXC is
shut down and archived after successful UDP/TCP DNS checks. Clients using its old
address must switch to the Docker host or an existing fallback.

See the top-level [`README.md`](../README.md) for review findings and open checks.
