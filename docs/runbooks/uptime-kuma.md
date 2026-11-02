# Uptime Kuma

## Scope and recorded state

Read [common precautions](common.md). [Monitoring Compose](../../server/monitoring/compose.yml)
defines stack `monitoring`, service/container `uptime-kuma`, pinned image
`louislam/uptime-kuma:2.2.1`, HTTP port 3003 to container 3001, and volume
`monitoring_uptimekuma_migrated_20261018` at `/app/data`. Override with
`UPTIMEKUMA_DATA_VOLUME` only for an intentional restore. The live source is
recorded under `/root/services/optiplex/monitoring/compose.yml` in CT 105.
The Mac route is `https://uptime.localhost`; archived CT 104 is stopped.

The migration record reports SQLite integrity, five monitors and one user retained,
a Pi-hole monitor updated to Docker, and HTTPS after restart. It does not prove
all notification delivery or monitor semantics. Kuma shares CT 105 with services
it watches; use an independent check for loss of that host.

## Diagnose

In the confirmed live stack directory:

```sh
docker compose ps uptime-kuma
docker compose logs --tail=100 uptime-kuma
```

Inspect mounts and CT disk space before restarting a database-erroring instance.
Test `http://DOCKER_LXC_IP:3003` from an authorized client and then HTTPS. If the
dashboard opens but a monitor is down, test its exact target and protocol from
Kuma's network perspective. A `.localhost` target inside Kuma resolves to Kuma's
own container, not the Mac proxy. Check target port, DNS, certificate trust,
credentials, and whether the destination moved during migration.

Distinguish a real target outage from a failed check. Do not silence all monitors
to clear a dashboard; record and use a bounded maintenance window for planned
work. Check notification provider connectivity and configuration separately from
HTTP monitor status. Logs and data may contain private URLs and notification tokens.

## Restart or recover

Restart only Kuma, leaving Grafana and Portainer in this shared stack alone:

```sh
docker compose restart uptime-kuma
```

For backup, stop only Kuma and archive the entire confirmed `/app/data` volume,
including SQLite sidecar/WAL files, plus the matching deployment source and image
identity. Restart Kuma afterward and verify checks resume. Do not copy only a live
`kuma.db` or assume a UI configuration export is a complete database backup.

Follow the common restore procedure into a fresh volume, keeping version 2.2.1
unless a separate upgrade is planned. Run a SQLite integrity check on a disposable
copy of the stopped restored database, using a compatible SQLite tool, before
starting the application. A result of `ok` checks structure, not monitor correctness.

Isolate a rehearsal from production destinations and notification providers
before first start: restored monitors may resume checks and alerts immediately.
Select the restored volume with `UPTIMEKUMA_DATA_VOLUME`, recreate only
`uptime-kuma` in the confirmed project, and inspect its `/app/data` mount. Stop on
a first-run account creation screen when expecting existing users.

## Acceptance and rollback

Verify original-account sign-in, expected monitors and history for the backup
date, current target addresses (especially Pi-hole), intervals, maintenance
windows, and notification assignments. Observe multiple check intervals. Perform
an intentional notification test to an approved destination, and confirm receipt;
record separately if notification testing is deferred. Recheck after restart.

If restore fails, stop Kuma and return to the saved compatible image and original
volume selection. Preserve failed restore data for diagnosis. Do not downgrade an
upgraded database in place. Starting archived CT 104 requires stopping Docker
Kuma first to avoid duplicate checks/alerts; its older targets and history need
review before enabling notifications.
