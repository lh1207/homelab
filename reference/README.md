# reference/ — not deployed via Docker

These Compose files are **not part of the live deployment**. They were earlier
attempts to run these services in Docker that were set aside in favor of other
runtimes. They're kept here for reference in case they're revisited later.

| Folder   | Service   | How it actually runs today                       |
|----------|-----------|--------------------------------------------------|
| `ollama` | Ollama    | Baremetal on the Mac (`workstation`), port 11434 |
| `dns`    | Pi-hole   | Its own Proxmox LXC                              |
| `wiki`   | BookStack | Its own Proxmox LXC                              |

See the top-level [`README.md`](../README.md) for the full topology.
