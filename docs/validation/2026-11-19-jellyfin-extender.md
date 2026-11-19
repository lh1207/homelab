# Jellyfin access from extender TVs — November 19, 2026

Status: Chrome sign-in reachability verified; operator confirmed the TV setup
works. No router, extender, Jellyfin, or deployment settings were changed.

## Observed topology and existing rule

Inspection in Chrome showed a TP-Link RE205 range extender on the upstream
Archer A7 LAN. The ASUS RT-AC67P runs in wireless-router mode with its WAN on
that upstream LAN and Jellyfin on its separate downstream LAN.

The ASUS Virtual Server / Port Forwarding page showed forwarding already enabled
and an existing Jellyfin TCP rule mapping external port 8096 to port 8096 on the
Docker LXC. No new forwarding rule or firewall exception was necessary.

## Client address and acceptance evidence

For TVs connected through the extender, enter the following server URL in the
Jellyfin app, substituting the ASUS router's upstream-facing WAN address:

```text
http://<asus-wan-ip>:8096
```

The extender's management address is not the Jellyfin server address. This path
uses the existing ASUS forwarding rule rather than the downstream private DNS
name and Caddy certificate setup. HTTP on this path is unencrypted; this record
establishes access within the household network, not Internet exposure or HTTPS
trust on the TVs.

- Chrome on the Mac loaded Jellyfin's **Please sign in** page through that WAN
  address and port. This verifies application reachability from the test client;
  it does not by itself prove a TV workflow.
- After being given this URL for the extender-connected TVs, the operator
  reported **“works thanks”** and supplied a photo in the chat. This is operator
  confirmation of the TV setup; the photo was not independently inspected or
  copied into the public repository.
- Authenticated TV playback, playback duration, codec, direct play, transcoding,
  and every individual TV were not independently measured during this check.

Real LAN addresses are retained in the ignored local operational handoff, in
keeping with the repository's address policy. The supplied photo's signed CDN
URL is omitted from this public record.

## Scope and recovery

No configuration rollback is needed because no live settings changed. Existing
HTTPS and Tailscale access remain separate documented paths. If this TV address
stops working, recheck the ASUS WAN address, the existing TCP 8096 forwarding
rule, and Jellyfin's direct service before changing network settings.
