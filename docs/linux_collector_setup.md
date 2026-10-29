# Linux Collector Setup and Detection

Ships `/var/log/auth.log` from an Ubuntu VM to CrowdStrike Falcon Next-Gen SIEM using the LogScale Collector, then detects SSH brute force. Console menu names vary by version; check CrowdStrike's current docs.

## 1. Create the HEC data connection
In the console, add a **HEC / HTTP Event Connector**, pick a Linux or auth parser, and save the **API key** and **URL**.

## 2. Install the collector
Run the install command from Fleet Management **on the VM**.
```bash
<INSTALL_COMMAND_FROM_CONSOLE>
systemctl status logscale-collector
```
The host should show as connected. Leave the local `config.yaml` unchanged (`mode: full`).

## 3. Grant log access
```bash
ls -l /var/log/auth.log     # if missing: sudo apt install rsyslog
sudo usermod -aG adm logscale-collector
sudo systemctl restart logscale-collector
```

## 4. Assign the config
In Fleet Management, create this config and assign it to the host:
```yaml
sources:
  auth_log:
    type: file
    include:
      - /var/log/auth.log
    sink: ngsiem

sinks:
  ngsiem:
    type: hec
    token: <API_KEY>
    url: <API_URL>
```

## 5. Verify ingestion
```bash
logger -p auth.info "pipeline test 001"
```
Search `"pipeline test"` in Advanced Event Search and check the timestamp and parsed fields.

## 6. Create the detection rule
Test this query in Advanced Event Search first:
```
"Failed password"
| regex("from (?<src_ip>\\d+\\.\\d+\\.\\d+\\.\\d+)")
| groupBy([src_ip], function=count())
| _count >= 5
```
Then create a correlation rule with it:
- **Trigger:** Summary
- **Schedule:** every 5 minutes, 10-minute window
- **MITRE:** Brute Force (T1110)

## 7. Trigger and confirm
```bash
ssh -p 2222 fakeuser@127.0.0.1    # wrong password, repeat 6+ times
```
Wait one rule run, then check the detections area of the console. The source IP shows as `10.0.2.2` (VirtualBox NAT).

See [troubleshooting.md](troubleshooting.md) for common errors.

> Never commit tokens, keys, or install commands.