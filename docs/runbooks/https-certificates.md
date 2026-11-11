# HTTPS certificates on Mac and iPhone

Use the [common precautions](common.md) and [baseline HTTPS setup](../../README.md#baseline-https)
to confirm the proxy and deployment before exporting a certificate.

## Select the correct CA

The server Caddy proxy serves `https://docker.<service>` names. The Mac proxy
serves the legacy `*.localhost` routes. They have separate local certificate
authorities (CAs). Each client needs to trust the public root of the proxy it
uses, once per CA, rather than installing a certificate for each service.

Trusting a CA lets applications using that trust store accept certificates it
signs. Confirm the source and fingerprint before approving that persistent
change. Preserve Caddy's `caddy_data` volume; never export its private keys.
See [Caddy's local HTTPS documentation](https://caddyserver.com/docs/automatic-https#local-https).

## Export and identify the public root

On the owning Docker host, in the confirmed live networking stack directory,
use the correct Compose project name if it differs from the directory name:

```sh
docker compose cp caddy:/data/caddy/pki/authorities/local/root.crt /tmp/homelab-caddy-root.crt
openssl x509 -in /tmp/homelab-caddy-root.crt -noout -subject -dates -fingerprint -sha256
```

Transfer only this public certificate through an existing trusted channel.
Compare its SHA-256 fingerprint with the root on the verified source host.
The display name alone is insufficient: both proxies can use similar names.
Keep certificates and private deployment addresses out of this public repository.

## Trust on macOS

After approving the CA trust change, install the verified public root into the
current user's login keychain with trust scoped to SSL:

```sh
security add-trusted-cert -r trustRoot -p ssl \
  -k "$HOME/Library/Keychains/login.keychain-db" /tmp/homelab-caddy-root.crt
```

Approve any macOS authorization prompt locally. The trust applies to applications
using this user's keychain; other users or applications with independent trust
stores may need separate configuration.

Verify actual HTTPS with the macOS curl, without a certificate override:

```sh
/usr/bin/curl --silent --show-error --output /dev/null \
  --write-out 'HTTP %{http_code} TLS=%{ssl_verify_result}\n' https://docker.homepage/
```

An expected HTTP response and `TLS=0` confirm this request passed TLS validation.
Do not use `--insecure` as a fix. A check with `--cacert` proves the supplied
chain works, but does not prove the keychain trusts it. A generic
`security verify-cert` check under its default policy also does not establish
the result of an SSL-scoped trust installation; test the intended HTTPS route.

## Trust on iPhone

Convert the verified public root to DER on the Mac:

```sh
openssl x509 -in /tmp/homelab-caddy-root.crt -outform DER \
  -out /tmp/homelab-caddy-root.cer
```

1. AirDrop the `.cer` file to the iPhone and accept it.
2. In Settings, install the downloaded profile. If needed, find it under
   **General → VPN & Device Management**.
3. Open **Settings → General → About → Certificate Trust Settings** and enable
   full trust for the verified Caddy root. Installing the profile alone does not
   enable SSL trust. See [Apple's instructions](https://support.apple.com/en-us/102390).
4. On home Wi-Fi, open `https://docker.homepage` in Safari. Confirm that it loads
   without a privacy warning; do not bypass the warning.

## Separate DNS, connectivity, and trust failures

The client must also resolve `docker.<service>` through Pi-hole. Under the
iPhone Wi-Fi network's **Configure DNS**, confirm the configured server is the
Docker Pi-hole address. An additional resolver that lacks the local records
can return “name not found”; it is not a reliable fallback for these names.
Do not assume the iPhone uses Pi-hole just because another client does.

| Symptom | Focused check |
| --- | --- |
| Direct IP web access fails | Confirm home Wi-Fi and LAN reachability before changing certificates. |
| Direct IP works, hostname cannot be found | Query Pi-hole for that name and inspect the client's DNS configuration. |
| Hostname loads with a privacy warning | Check the selected CA, full trust, certificate validity, and hostname. |
| HTTPS validates but returns 502 | Check Caddy's upstream and backend; client trust has passed. |

For example, use `http://DOCKER_LXC_IP:8888/admin/` to check direct Pi-hole web
reachability, replacing the placeholder with the private inventory address.
If necessary, use curl's `--resolve` with the hostname and known server address
to test HTTPS independently of DNS while preserving hostname verification.
These LAN instructions do not configure remote DNS or routing over Tailscale.

## Verification record — November 10, 2026

The exported server root matched the live root over strictly verified SSH.
After the approved SSL trust installation, all 13 deployed server service routes
passed macOS curl TLS verification without `--cacert`. Twelve resolved normally;
Uptime needed an explicit address because the Mac resolver failed its lookup.
Pi-hole answered that name correctly while the router returned NXDOMAIN; no DNS
configuration was changed during the certificate work.

The operator confirmed Mac access, then confirmed iPhone access after resolving
DNS and enabling CA trust. iPhone verification was operator-reported Safari
behavior, not an automated test. Other devices and remote HTTPS remain unverified.
