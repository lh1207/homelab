# Common operating and restore precautions

## Establish the target

The [root README](../../README.md) records topology and dated checks. Confirm the
current host, container, image, mounts, and Compose project before making changes.
This Git checkout is not the live deployment source. Most sources are under
`/root/services/optiplex/<stack>/compose.yml` in CT 105; use container labels to
confirm. Do not deploy a reference stack alongside a running service.

For a named container, use narrowly selected output rather than dumping inspect
or resolved Compose configuration, which can expose secrets:

```sh
docker inspect --format '{{.Config.Image}} {{.Image}}' CONTAINER
docker inspect --format '{{json .Mounts}}' CONTAINER
docker inspect --format '{{index .Config.Labels "com.docker.compose.project"}}' CONTAINER
docker inspect --format '{{index .Config.Labels "com.docker.compose.project.config_files"}}' CONTAINER
docker inspect --format '{{index .Config.Labels "com.docker.compose.project.working_dir"}}' CONTAINER
```

Replace uppercase placeholders first. Run service commands in the confirmed live
stack directory, using `docker compose -p PROJECT …` with the label's project
name if it differs from the directory name. Review logs privately and redact
tokens, internal addresses, account details, and query data before sharing.

Check Proxmox, CT 105, Docker, and the application separately. If several services
fail together, investigate the common host before restarting individual services.
Use local console access when network or remote-access changes could lock you out.

## Separate the application from its proxy

Test the direct HTTP port from an authorized LAN client, then the expected HTTPS
route. `*.localhost` resolves to the client itself: the recorded Mac routes need
the Mac's Caddy and its trusted public root certificate. They do not establish LAN
DNS or remote routing. An HTTPS failure with a healthy direct backend points to
the proxy, upstream setting, or client trust. Do not disable TLS verification as
the fix. Preserve Caddy's CA volumes and keep CA private keys private.

## Before a restart, upgrade, or restore

Record the incident time, symptom, last change, selected image IDs/digests,
project, mounts, and last usable backup. Do not run `docker compose down -v`,
volume prune, or delete the retained source LXCs. Avoid `pull` as a diagnostic:
Pi-hole and BookStack references use `latest`, so recreation can change versions.
Plan upgrades separately with a backup and a version-compatible rollback.

The dated Mac backup is at `~/Documents/Backups/homelab/2026-10-18`. The root
README records 35 matching checksums and disposable SQLite checks, not a complete
host restore. No recurring schedule or retention policy was configured. Confirm
the actual available backup date and contents; do not assume newer data is covered.

For a consistent cold backup, arrange downtime and stop every writer to the
selected volumes. BookStack requires both its application and database stopped
for a cold pair. Use the existing trusted backup tooling to archive the confirmed
volumes, preserving ownership and permissions, plus the matching external Compose
sources and protected `.env`/keys. Store privately with restrictive access, verify
archive checksums, and restart services in dependency order. Avoid copying a live
SQLite database alone; include its full stopped data directory and any WAL files.

## Restore and rollback gate

1. Preserve the failed state and original backup. Verify checksums from the backup
   directory (`sha256sum -c SHA256SUMS`, or `shasum -a 256 -c SHA256SUMS` on Mac).
2. Restore into new, deliberately named volumes. Confirm archive layout, owners,
   application version, secrets, and mount destinations before starting anything.
3. Rehearse in an isolated environment without production DNS ports, outbound
   notifications, or other household side effects. Do not run two writers against
   the same database or attach production volumes to a test deployment.
4. During a cutover, stop the current instance, set only the intended volume
   overrides in the protected deployment configuration, and verify resulting
   mounts after recreation. A first-run screen is a stop signal, not an invitation
   to initialize an apparently empty application.
5. Run the service-specific acceptance checks and record results. If they fail,
   stop the replacement and restore the saved deployment configuration and
   compatible pre-change data together. Newer schemas may not support older images.

Archived sources reflect cutover time and may lose newer changes. Stop the Docker
replacement before deliberately returning to an archived LXC; confirm addresses,
startup settings, and client routing. Never start both copies to experiment.
Record the chosen recovery point and data loss before switching.

Close the incident only after client workflows pass. Record backup identity,
commands, elapsed time, checks, unresolved issues, and any changes to this runbook
in protected operational notes. Do not publish private archives or credentials.
