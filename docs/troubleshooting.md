# Troubleshooting Log

Newest entries go at the top. Format: problem, cause, fix.
## TS-006: `Rename-NetAdapter` says no adapter found
- **Cause:** The name didn't match exactly. The command used `Ethernet2`, but Windows names the second adapter `Ethernet 2` (with a space), or the name was different on that VM.
- **Fix:** List the real names with `Get-NetAdapter`, identify the NAT adapter by its `10.0.2.x` address, and rename by interface index:
```powershell
  Get-NetIPAddress -AddressFamily IPv4 | select InterfaceAlias, InterfaceIndex, IPAddress
  Get-NetAdapter -InterfaceIndex <n> | Rename-NetAdapter -NewName "NAT"
```
- **Note:** If only one adapter appears, enable Adapter 2 (Internal Network `lab.local`) in VirtualBox with the VM powered off.

## TS-005: Events searchable but no detection
- **Cause:** Events don't create detections by themselves; a correlation rule must exist, be enabled, and run on its schedule.
- **Fix:** Tested the query in Advanced Event Search first, then created a rule (Summary trigger, 5 min frequency, 5 min window, MITRE Brute Force). Fresh failed logins triggered it.
- **Note:** The search window must be at least as long as the frequency. Detections lag events by one schedule interval.

## TS-004: Collector `permission denied` on `auth.log`
- **Cause:** `/var/log/auth.log` is `syslog:adm` with mode `640`, and the `logscale-collector` service account wasn't in `adm`.
- **Fix:**
  ```bash
  sudo usermod -aG adm logscale-collector
  sudo systemctl restart logscale-collector
  ```
- **Check:** `sudo -u logscale-collector head -n 1 /var/log/auth.log`
- **Note:** If it still fails, add `SupplementaryGroups=adm` via `sudo systemctl edit logscale-collector`.

## TS-003: Config edits ignored (Fleet Management full mode)
- **Cause:** Local `config.yaml` had `mode: full`, so the collector takes its config from the Falcon console.
- **Fix:** Created the config in the Fleet Management tab and assigned it to the host. Left the local file unchanged.

## TS-002: Wrong service name
- **Cause:** Examples used `humio-log-collector`, but the install is `logscale-collector` (data dir `/var/lib/logscale-collector`).
- **Fix:** Used the real name. Verify with `systemctl list-units | grep -i -E "humio|logscale|collector"`.

## TS-001: Unsure of sink type (`hec` vs `humio`)
- **Cause:** The sink type must match the ingest endpoint.
- **Fix:** Created an HEC data connection and used `type: hec` with its URL and key (the key is shown only once).

## Quick checks when events don't arrive
1. `journalctl -u logscale-collector -f` for errors
2. Config assigned to the host in Fleet Management?
3. Can the service account read the file?
4. Sink type, URL, and key correct?
5. Outbound HTTPS (443) working, and parser assigned?

> Never store tokens or keys in this file.