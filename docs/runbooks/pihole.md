# Pi-hole

## Scope and recorded state

Read [common precautions](common.md). [Reference Compose](../../reference/dns/compose.yml)
is not the live source, recorded at `/root/services/optiplex/dns/compose.yml`.
Stack `dns`, service/container `pihole`, runs in CT 105. The migration record
reports FTL v6.6.1, UDP/TCP DNS and HTTPS checks; it does not confirm household
client DNS cutover. CT 100 is stopped in Archive.

| Persistent mount | Selected migrated volume |
| --- | --- |
| `/etc/pihole` | `dns_pihole_migrated_20261018` |
| `/etc/dnsmasq.d` | `dns_pihole_dnsmasq_migrated_20261018` |

Overrides are `PIHOLE_DATA_VOLUME` and `PIHOLE_DNSMASQ_VOLUME`. DNS publishes
TCP/UDP 53; web publishes 8888. The Mac route is `https://pihole.localhost/admin/`.

## Diagnose

On CT 105, inspect mounts/project as described in common precautions, then:

```sh
docker compose ps pihole
docker compose logs --tail=100 pihole
ss -lntup 'sport = :53'
docker exec pihole pihole status
```

From a client intended to use this resolver, replace `DNS_SERVER_IP`:

```sh
dig @DNS_SERVER_IP example.com A +time=2 +tries=1
dig @DNS_SERVER_IP example.com A +tcp +time=2 +tries=1
```

Both must return a successful DNS response. Check expected local records and a
known configured blocklist entry separately. A working admin page does not prove
DNS works. Compare explicit-server queries with the client's normal resolver;
check DHCP/router and IPv6 resolver assignments if clients still use CT 100 or
bypass Pi-hole. A cached answer alone does not prove current upstream access.

If DNS times out, check host reachability, firewall permissions for both protocols,
port ownership, and upstream reachability. The record says `LOCAL` mode rejected
Docker queries and `ALL` resolved that issue. `all` plus bindings on `0.0.0.0`
requires LAN/tailnet firewall scoping; do not expose an open resolver to the Internet.
If port 53 is occupied by systemd-resolved, first arrange a working host DNS
fallback and inspect its configuration and `/etc/resolv.conf` target. Disable only
the conflicting stub deliberately; do not blindly stop the resolver or copy the
reference's `sed` command without checking the current host.

## Restart or recover

Ensure the documented household DNS fallback actually works before downtime;
record its original settings so clients can be returned afterward. With correct
mounts and no port conflict, restart only this service:

```sh
docker compose restart pihole
```

For restore, follow the common cold-backup procedure for both volumes and matching
configuration. Preserve the migrated `pihole.toml` password hash, upstreams, local
records, gravity data, and dnsmasq settings. The live migration omits the password
environment override; the reference requires a password variable and can overwrite
that behavior. Do not deploy it unchanged against migrated data. Keep the intended
advertised host IP and listening mode; never print the password/hash in evidence.

Restore both archives to new volumes and select them with the two overrides in
the confirmed live configuration. Verify mounts before client cutover. Do not
pull a new `latest` image during recovery; use the recorded compatible image.

## Acceptance and rollback

Require direct UDP and TCP answers, expected local resolution and blocking,
admin sign-in, and queries from a real client using its normal DNS settings.
Recheck after a service restart. Verify intended LAN and IPv6 resolver settings;
return clients from temporary fallback only after checks pass. Record whether a
host reboot was tested separately.

If checks fail, retain fallback DNS, stop the failed replacement, and restore the
saved volume selection/configuration. CT 100 is a dated last-resort source, not
an automatic failover: stop Docker Pi-hole before starting it and deliberately
return client resolver addresses. Preserve any newer configuration first.
