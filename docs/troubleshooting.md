# Troubleshooting Log

Newest entries go at the top. Format: problem, cause, fix.

## TS-014: Audit log events flagged "filtered out by the parser"
- **Cause:** The audit connection used a syslog-style parser (`#type: linux-syslog`), which cannot read auditd records (`type=SYSCALL msg=audit(...)`). Events arrived with `#error=true`, no parsed fields, and an ingest-time `@timestamp`. The original line stayed intact in `@rawstring`.
- **Fix:** Assigned the `linux-audit` parser to the audit connection. Events now arrive as `#event.dataset=auditd.syscall` or `auditd.user_cmd`, with the audit key in `rule.name`, the program in `process.executable`, the login user in `user.name`, and the event time in `@timestamp`.
- **Note:** Audit rule loading also writes `type=CONFIG_CHANGE` records that carry the same key. Filter detection queries to `#event.dataset=auditd.syscall` so loading rules doesn't trigger them.

## TS-013: Collector sink returns 401 Unauthorized
- **Cause:** CrowdStrike rejected the API key for the `falcon_audit` sink. The `falcon` sink kept working, so the problem was limited to the audit sink's `url` or `token`: a key that was copied wrongly or truncated, regenerated since, copied from the wrong connection, or paired with another connection's URL.
- **Fix:** Generate a new API key on the audit connection (it is shown once), paste it and that connection's own URL into the `falcon_audit` sink, and quote both values. Test the key outside the collector: `curl` a test event to the connection's URL with `Authorization: Bearer <key>` and expect HTTP 200. Make sure the config is still assigned to the host.
- **Result:** The collector kept retrying. Once the key was valid, the queued audit events arrived, about 18 hours of backlog.

## TS-012: Windows brute-force query matched nothing (event code and field names)
- **Cause:** The query used guessed field names. The parser uses `windows.EventID` for the event ID, `@collect.host` for the host, and `source.address` for the source, not `event.code` and `source.ip`. The event a failed `net use` produces also depends on the protocol: Kerberos (connecting by name from another machine) logs 4771, while NTLM (connecting by IP, or running the command on the DC itself) logs 4625 and 4776.
- **Fix:** Use the real fields and match both families: `(windows.EventID=4771 or windows.EventID=4625)` grouped by account and `source.address`.
- **Note:** The source value may appear as `::ffff:10.10.10.20`. To see which codes a host is sending: `#repo!=base_sensor @collect.host="LAB-DC" | groupBy([windows.EventID], function=count())`.

## TS-011: Windows HEC connection stays idle (0 B)
- **Cause:** The collector config was created in Fleet Management but never selected for the collector host, so the collector never received the sources and sink.
- **Fix:** Assign the config to the `LAB-DC` collector instance in Fleet Management, then wait a few minutes. The connection began showing ingest.
- **Note:** Events from the host can still appear in search while the connection is idle, because Falcon sensor telemetry arrives separately (`#repo=base_sensor`). A test event written with `eventcreate` showed up that way and did not prove the collector worked. To test the collector alone, use a `file` source and `#repo!=base_sensor`.
- **Check:** The host shows as connected with the config assigned, and the connection's ingest counter rises.

## TS-010: Shared clipboard not working on the Server Core DC
- **Cause:** Clipboard sharing relies on `VBoxTray.exe`, a desktop component. Server Core has no desktop shell, so it doesn't start by itself.
- **Tried:** Found it at `C:\Program Files\Oracle\VirtualBox Guest Additions\VBoxGuest\VBoxTray.exe` and started it by hand. Paste still did not work.
- **Workaround:** Manage the DC from HOST01, where paste works: `Enter-PSSession -ComputerName LAB-DC -Credential POCLAB\Administrator` (if it fails, run `Enable-PSRemoting -Force` on the DC). Scripts can also be passed through a shared folder.

## TS-009: HOST01 shows only one adapter with a `169.254.x.x` address
- **Cause:** Only the internal-network adapter was enabled in VirtualBox. It has no DHCP server, so Windows gave it an automatic `169.254.x.x` address, and there was no NAT adapter for internet access.
- **Fix:** Shut the VM down cleanly, then in VirtualBox enable **Adapter 2** and attach it to **NAT**. After boot, `Get-NetIPAddress -AddressFamily IPv4` should list a `169.254.x.x` adapter (lab) and a `10.0.x.15` adapter (NAT). Rename them by interface index.

## TS-008: Stale default gateway `10.0.2.1` on the LAB adapter
- **Cause:** A gateway from the earlier `10.0.2.x` address plan stayed on `LAB`. The lab network needs no gateway, since internet access goes through the NAT adapter, and a second default route can break connectivity.
- **Fix:**
```powershell
  Get-NetRoute -DestinationPrefix "0.0.0.0/0" | select InterfaceAlias, NextHop, RouteMetric
  Remove-NetRoute -InterfaceAlias "LAB" -DestinationPrefix "0.0.0.0/0" -NextHop 10.0.2.1 -Confirm:$false
```

- **Check:** Only the NAT adapter should keep a default route, and `Test-NetConnection crowdstrike.com -Port 443` should pass.

## TS-007: DC's NAT adapter lists outside DNS servers
- **Cause:** The `NAT` adapter had its own DNS servers (`192.168.0.52`, `192.168.0.53`, `8.8.8.8`, `8.8.4.4`). A DC should resolve only through itself, or domain lookups can leak to outside servers.
- **Fix:** Point both adapters at the DC's own DNS and let the DNS server forward external names:
```powershell
  Set-DnsClientServerAddress -InterfaceAlias "NAT" -ServerAddresses 127.0.0.1
  Set-DnsClient -InterfaceAlias "NAT" -RegisterThisConnectionsAddress $false
  Add-DnsServerForwarder -IPAddress 8.8.8.8
```

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