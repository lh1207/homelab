# Service runbooks

Procedures for the documented homelab, based on repository main `6f8e0a2` and
its October 18–19, 2026 migration record. Written November 2, 2026. These are
operator instructions, not evidence that a recovery drill or a new live check
has passed.

| Service | Runbook | Recorded placement |
| --- | --- | --- |
| Pi-hole | [DNS diagnosis and recovery](pihole.md) | Docker LXC CT 105, stack `dns` |
| Tailscale | [Deployment, access diagnosis and recovery](tailscale.md) | CT 105 single-host route and Pi-hole DNS; iPhone trusted Jellyfin/Homepage HTTPS confirmed November 11 |
| BookStack | [Library diagnosis and recovery](bookstack.md) | CT 105, stack `wiki` |
| Uptime Kuma | [Monitoring diagnosis and recovery](uptime-kuma.md) | CT 105, stack `monitoring` |

Read [common operating and restore precautions](common.md) first. Keep a protected
offline copy with the private inventory; BookStack must not be the only place
its own recovery instructions are available.
