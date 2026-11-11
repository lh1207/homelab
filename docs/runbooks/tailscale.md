# Tailscale runbook

> **Status: deployed; iPhone cellular DNS and trusted HTTPS operator-confirmed November 11, 2026.** Tailscale 1.104.6 was installed directly in Docker LXC CT 105 (Debian 13) from the official stable APT repository on November 10. On November 11, the operator approved TUN passthrough and a stop/start. `/dev/net/tun` persisted afterward, and `tailscaled` was active and enabled at boot. Interactive enrollment completed: the device was online, backend state was `Running`, and no health warnings were reported. CT 105 received three Tailscale ping replies from the online iPhone through a relay. The operator confirmed Jellyfin loaded on the iPhone over cellular with Wi-Fi off, using CT 105's Tailscale IPv4 address and port 8096.

Use the shared [runbook conventions](common.md) for incident notes and change records.

## Deployment and verification record

The SSH host fingerprint was compared with the authenticated Proxmox CT 105
console before installation. No existing Tailscale installation or state
directory was found in the preflight checks.

The approved TUN device was added as `dev2` through Proxmox, preserving `dev0`
(`/dev/dri/renderD128,gid=992`) and `dev1` (`/dev/dri/card0,gid=44`). CT 105
remains unprivileged. See the
[official LXC instructions](https://tailscale.com/docs/features/containers/lxc/lxc-unprivileged)
for the device mechanism.

After the stop/start, trusted LAN SSH worked and all 17 Docker containers were
running. Jellyfin and Open WebUI health endpoints returned HTTP 200; BookStack
and Pi-hole web endpoints returned HTTP 302. From the Mac, the existing HTTPS
Homepage route returned 200 and Jellyfin returned 302 with certificate validation
enabled. The media share remained mounted read-only as CIFS. These checks prove
basic recovery, not authenticated workflows or tailnet access.

Enrollment used `tailscale up --accept-dns=false --timeout=15s` to preserve
existing host DNS. The command timed out awaiting sign-in, then the operator
completed authentication and a subsequent status check confirmed `Running`.
Do not store authentication URLs or keys in this repository.

`tailscale netcheck` confirmed UDP connectivity and relay reachability. At
enrollment, the device advertised no subnet routes or exit node, selected no exit
node, and had Tailscale SSH disabled. Jellyfin's health endpoint returned 200
when addressed through the device's own Tailscale IPv4 address; this is a local
binding check and does not establish access from another device.

The iPhone subsequently appeared online in the same tailnet. Three
`tailscale ping --c 3 --timeout=5s` probes received replies through the Chicago
DERP relay. The command exited 1 because a direct connection was not established;
the received relay replies prove peer connectivity, not application access.

The operator confirmed Jellyfin loaded on the iPhone over cellular with Wi-Fi
off, using CT 105's Tailscale address on port 8096. This verifies external
application reachability; authenticated playback was not tested. This direct
HTTP application route travels inside the encrypted tailnet. The subsequent DNS,
routing, and iPhone certificate setup is recorded below. Enrollment occurred
after the stop/start, so
post-enrollment reboot recovery has not been tested.

## Existing service names over cellular — November 11, 2026

Live inspection confirmed Pi-hole still maps the `docker.<service>` names to
CT 105's LAN IPv4 address. Caddy uses host networking and its existing private
CA; Pi-hole publishes TCP/UDP 53 on all host addresses with listening mode `ALL`.
The deployed sources remain external: `/root/services/optiplex/dns/compose.yml`
and `/root/services/local/networking/compose.yml`. Neither was changed.

### Applied configuration

- Advertise and approve **only CT 105's LAN IPv4 `/32`**. No whole-LAN route,
  exit node, or additional IPv6 route was enabled. This preserves the existing
  Pi-hole answers and Caddy names without maintaining a second DNS record set.
- Set CT 105's Tailscale IPv4 address as the sole **global nameserver**, with
  **Override DNS servers enabled**. Do not restrict it to a domain: these names
  use different suffixes (`.jellyfin`, `.homepage`, etc.), and ordinary DNS must
  also use Pi-hole for filtering. Public fallback resolvers would bypass that
  filtering and cannot resolve these private names.
- Preserve MagicDNS, the existing search domain, and tailnet HTTPS settings.
  CT 105 retains `--accept-dns=false`, `--accept-routes=false`, no exit-node
  selection, and Tailscale SSH disabled. IPv4 forwarding was already `1`;
  no sysctl or firewall changes were necessary.
- Preserve the existing policy. Inspection showed an unrestricted grant
  (`src: ["*"], dst: ["*"], ip: ["*"]`). The operator explicitly approved the
  route and DNS change after being told this permits tailnet members to reach
  CT 105's listening ports, not just Caddy. A `/32` limits destination hosts,
  not ports. Restricting the policy is a separate administrative change.

The live command was `tailscale set --advertise-routes=CT105_LAN_IPV4/32`
with the actual address substituted, followed by approval in the device's
**Edit route settings**. The console was rechecked: exactly that route is
approved and exit-node use is off. See [Tailscale subnet routing](https://tailscale.com/docs/features/subnet-routers)
and [tailnet DNS](https://tailscale.com/docs/reference/dns-in-tailscale).
Do not repeat the literal placeholder command or replace pre-existing route
lists on a future deployment without inspecting them first.

### Public root certificate and iPhone setup

Only `/data/caddy/pki/authorities/local/root.crt` was exported from the running
server Caddy container. No CA private key was copied or changed. The certificate
is `Caddy Local Authority - 2026 ECC Root`, valid October 19, 2026 through
August 27, 2036. Its SHA-256 fingerprint is:

```text
76:8B:29:40:32:01:D4:D6:48:C1:0D:83:40:0B:04:D6:D7:1D:9D:11:07:8D:91:59:71:1A:F1:24:7E:74:54:7E
```

A certificate-only, unsigned configuration profile was prepared on this Mac at
`/private/tmp/homelab-caddy-iphone.mobileconfig`; its plist passed `plutil -lint`.
It contains one `com.apple.security.root` payload with the public certificate,
no private keys, VPN settings, or device-management enrollment. The plain public
certificate is `/private/tmp/homelab-iphone-caddy-root.crt`. Temporary Mac files
are conveniences, not durable backups; re-export the public certificate from
Caddy or use its protected backup if they disappear.

For a new iPhone, transfer the public certificate/profile through a trusted
channel such as AirDrop, verify the fingerprint, and install it in
**Settings → General → VPN & Device Management**. Then enable its full SSL trust
under **General → About → Certificate Trust Settings**. Installation alone does
not enable SSL trust; see [Apple's instructions](https://support.apple.com/en-us/102390).
Keep Tailscale connected and **Use Tailscale DNS** enabled. Do not click through
certificate warnings or install another proxy's root by mistake.

### Verification and limits

- In CT 105, explicit queries to Pi-hole's Tailscale IPv4 address returned the
  existing LAN address for Jellyfin over UDP and Homepage over TCP. Public
  `example.com` resolution succeeded over both protocols; `doubleclick.net`,
  present on the configured blocklist, returned `0.0.0.0`. These are local
  binding/resolver checks, not independent remote-client packet tests.
- After the route and DNS changes, the operator was asked to install/trust the
  profile, disable iPhone Wi-Fi, connect Tailscale with its DNS enabled, and test
  `https://docker.jellyfin/` and `https://docker.homepage/` without certificate
  warnings, including Jellyfin sign-in. The response was **“works”**. Record this
  as operator-confirmed end-to-end name resolution, trusted HTTPS, Jellyfin,
  and a second Caddy application over cellular. No separate iPhone DNS diagnostic
  output or media-playback test was collected.
- The Mac's post-change LAN checks used the exported server CA and explicit
  server-address resolution: all 13 deployed Caddy names returned HTTP 200,
  302, or 308 with TLS verification result `0`. This verifies LAN routing,
  Caddy certificates, and basic backend responses, not every authenticated flow.
- All 17 Docker containers remained running; all six configured healthchecks
  were healthy. LAN SSH remained available. The authenticated Proxmox UI
  confirmed preserved GPU devices `dev0`/`dev1` and TUN `dev2`; Proxmox SSH did
  not authenticate, so do not assume it is an available recovery method.
  Tailscaled remained active/enabled and the read-only CIFS mount was retained.
- A single post-change Tailscale ping to the iPhone timed out. The operator's
  successful application test is the acceptance evidence; direct peer transport
  was not established by that probe. No restart, host reboot, or failover test
  was performed in this extension.

Working cellular URLs confirmed by the operator: `https://docker.jellyfin/`
and `https://docker.homepage/`. Other deployed aliases keep the same DNS/Caddy
path; they passed LAN checks but were not individually tested on the iPhone.
Staging remains undeployed. Proxmox, Home Assistant, and other LAN hosts are
outside this route. The iPhone needs no further setup for these confirmed URLs;
additional clients need their own Tailscale DNS and server-CA trust setup.

Global DNS override applies to every DNS-accepting tailnet client and makes DNS
dependent on this homelab while connected. Applications using their own encrypted
DNS can bypass Pi-hole. Device key expiry remains enabled, as the route dialog
warned; expiry can interrupt access until reauthentication. Existing LAN clients
continue using the same records and CA. The `/32` can also win over a client's
connected LAN route to this one host, so test both Wi-Fi and cellular after
future client/network changes.

### Backup and rollback

Before advertisement, a mode-0700 directory was created inside CT 105:
`/root/homelab-review-backups/tailscale-dns-20261111`. It contains restricted
`prefs-before.json`, routes/forwarding evidence, the admin DNS baseline, Caddyfile,
and the public root certificate. Do not publish the preference backup or copy
Tailscale identity state. Baseline: no advertised routes/custom nameservers,
DNS override off, MagicDNS on. Caddy and Pi-hole configurations were not changed.

Use LAN SSH or the authenticated Proxmox console for recovery:

1. In tailnet DNS, turn **Override DNS servers off**, then remove only the newly
   added CT 105 Tailscale nameserver. Preserve MagicDNS/search domains and any
   later unrelated settings. This restores normal client DNS first.
2. In CT 105's route settings, unapprove its LAN `/32`. Inside CT 105 run
   `tailscale set --advertise-routes=` to restore this change's empty route list.
   If routes have been added since, remove only this `/32` instead.
3. Leave host DNS, sysctls, firewall, LXC devices, Docker, and Caddy CA volumes
   unchanged. No service restart is required. If desired, remove only the
   Homelab Caddy certificate profile from the iPhone; this removes its HTTPS trust.
4. Recheck LAN SSH, Pi-hole TCP/UDP, the Caddy URLs with the correct CA, Docker
   health, and the retained direct Tailscale Jellyfin address/port. Record the
   actual client result; never delete identity state or re-enroll as rollback.

## Diagnose access

First determine whether the issue affects one device or multiple devices, and whether local LAN access still works. Record the affected device and the time the issue began. Do not share device keys, auth keys, or other credentials in notes.

If the affected machine is confirmed to be a Linux host running Tailscale under systemd, these checks are read-only:

```sh
sudo systemctl status tailscaled --no-pager
tailscale status
tailscale ip -4
sudo journalctl -u tailscaled --since '30 minutes ago' --no-pager
```

Run `tailscale` commands inside CT 105 as an account with access to the daemon. If diagnosing a different device, first confirm its placement and use that platform's documented status and log view.

Check whether the daemon is running, whether the affected peer appears in status, and whether recent logs show connectivity or authentication errors. A reachable Tailscale peer does not by itself confirm that any particular service or route is available.

For a confirmed installation, replace `PEER` with an intended peer's known Tailscale IP or name:

```sh
tailscale ping PEER
tailscale netcheck
```

These send diagnostic traffic. A relayed connection can still be functional; a direct connection is not an acceptance requirement. If peer connectivity works but an application fails, test the application's actual port and check its listening address, host firewall, and tailnet access policy. Check device expiry or pending approval in the administrator's device inventory when authentication is the failure. Inspect local DNS and any configured tailnet DNS separately if an IP works but a name fails. A `*.localhost` application URL still resolves to the client itself; Tailscale does not turn it into a remote service name.

## Restart

Restart only after confirming the affected host and service manager. A restart briefly interrupts that device's Tailscale connectivity. If this is your only remote path to the host, do not restart remotely: arrange local or out-of-band access first, or defer until someone can recover the host locally.

For a confirmed systemd-managed Linux installation, the restart command is:

```sh
sudo systemctl restart tailscaled
```

Then check `systemctl status tailscaled` and `tailscale status` again, and verify access to the specific required peer or service. If connectivity does not return, use the available local or out-of-band console to inspect logs and restore access. Do not repeatedly restart or change routes as a troubleshooting shortcut.

## Identity and recovery

Treat Tailscale state directories, machine identity material, and authentication keys as secrets. Do not copy them into a general-purpose backup, ticket, or chat. Before a planned host rebuild or state removal, confirm the approved backup method, who controls the tailnet, and the recovery procedure. This repository does not document those details.

Record the installed version, service manager, required peers, non-secret routing/DNS settings, and access policy ownership in protected inventory. If identity backup is authorized, use encrypted restricted storage and the platform's consistent backup procedure. Never start an original host and a restored clone with the same machine identity; isolate a recovery rehearsal and use a deliberate enrollment plan.

Do not delete Tailscale state, revoke a device, generate or use an auth key, re-enroll a machine, alter route approvals, or change exit-node settings without authorization from the tailnet administrator. These actions can change access for this device or other users. If identity state is lost, preserve the host and logs where practical, then ask the administrator to approve the recovery or re-enrollment path.

After an approved recovery, verify that the intended device is online and that required peer or service access works. Confirm any required routes with the administrator. Keep the previous working access path available until verification succeeds; if it fails, use the approved local or out-of-band recovery path and restore the prior configuration where possible.

Record peer connectivity, the actual application workflow, required DNS lookups, and any intentionally configured subnet or exit-node behavior separately. Test from the client that needs access and retain normal LAN/console access. A daemon restart does not prove recovery after host reboot; record that check separately when performed.
