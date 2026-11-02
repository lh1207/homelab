# Tailscale runbook

> **Status: proposed, unverified.** The repository contains no Tailscale deployment, host assignment, or service configuration. Confirm where Tailscale runs and how it is managed before using platform-specific commands. This runbook does not establish whether this network uses an exit node, subnet routes, or any particular authentication method.

Use the shared [runbook conventions](common.md) for incident notes and change records.

## Diagnose access

First determine whether the issue affects one device or multiple devices, and whether local LAN access still works. Record the affected device and the time the issue began. Do not share device keys, auth keys, or other credentials in notes.

If the affected machine is confirmed to be a Linux host running Tailscale under systemd, these checks are read-only:

```sh
sudo systemctl status tailscaled --no-pager
tailscale status
tailscale ip -4
sudo journalctl -u tailscaled --since '30 minutes ago' --no-pager
```

Run `tailscale` commands as the local account with access to the daemon. If Tailscale runs in a container, appliance, or another host, use that platform's documented status and log view instead; its placement is not recorded here.

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
