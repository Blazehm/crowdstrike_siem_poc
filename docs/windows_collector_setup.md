
# Windows Collector Setup
 
Ships the Windows Security, System, and Directory Service event logs from `LAB-DC` to CrowdStrike Falcon Next-Gen SIEM using the LogScale Collector. Console menu names vary by version; check CrowdStrike's current docs.
 
## Why a collector as well as the sensor
The Falcon sensor reports endpoint behavior (processes, network, files) on its own. The collector ships the Windows event logs themselves, such as logons, account changes, and group changes. Sensor data lands in `#repo=base_sensor`, and collector data in a separate repository, so the two are easy to tell apart.
 
## 1. Create the Windows HEC data connection
In the console, add a new **HEC / HTTP Event Connector** (separate from the Linux one), choose a **Windows event parser**, and save the **API key** (shown once) and the **URL**. Use a connection name that includes `windows`, so it's easy to filter later.
 
## 2. Install the collector on LAB-DC
The Core DC has no paste support, so use HOST01 as a staging machine:
1. Save the Windows install command from Fleet Management in a file on HOST01, such as `C:\Temp\install-collector.ps1`. It contains an enrollment token, so treat it as a secret.
2. Copy it across and run it:
```powershell
Copy-Item C:\Temp\install-collector.ps1 \\LAB-DC\C$\Temp\
```
   Then on `LAB-DC`:
```powershell
powershell -ExecutionPolicy Bypass -File C:\Temp\install-collector.ps1
Get-Service *logscale*
```
3. Delete the script from both machines afterward.
The host should appear as connected in Fleet Management.
 
## 3. Create the config
In Fleet Management, create a config:
```yaml
sources:
  windows_events:
    type: wineventlog
    sink: ngsiem_win
    channels:
      - name: Security
      - name: System
      - name: Directory Service
 
sinks:
  ngsiem_win:
    type: hec
    token: <API_KEY>
    url: <API_URL>
```
- The `sink:` name must match the sink block exactly.
- The token and URL must come from the **Windows** connection, not the Linux one.
- The URL has the form `https://<id>.ingest.<region>.crowdstrike.com`, with no extra path.
- Compare it with CrowdStrike's Windows (NG-SIEM) example for your collector version.
## 4. Assign the config to the host
Creating a config does not apply it. Select the config for the `LAB-DC` collector instance in Fleet Management, and wait a few minutes. Skipping this leaves the connection idle at 0 B.
 
## 5. Verify
Generate events on the DC:
```powershell
Add-ADGroupMember -Identity "SOC-Lab-Admins" -Members labuser1
Remove-ADGroupMember -Identity "SOC-Lab-Admins" -Members labuser1 -Confirm:$false
New-ADUser -Name "Temp User" -SamAccountName tmpuser1 -Enabled $false
Remove-ADUser tmpuser1 -Confirm:$false
```
Search for the collector's copies, excluding sensor data:
```
#repo!=base_sensor "4728"
```
Then check that the connection's ingest counter rises, and that the timestamp and parsed fields look right on an event. If they don't, change the parser on the connection.
 
## Events to expect
 
| Channel | Events | Use |
|---|---|---|
| Security | 4624, 4625 (logons), 4720 (account created), 4728/4732/4756 (group changes), 4740 (lockout), 4688 (process creation) | Identity and account detections |
| System | 7045 (service installed), 6008 (unexpected shutdown) | Persistence and system events |
| Directory Service | AD database events (domain controllers only) | AD health |
 
Channel names must match Windows exactly. List them with `Get-WinEvent -ListLog *`.
 
## HOST01
HOST01 runs the sensor only. To collect its event logs too, install the collector there and assign the same config without the `Directory Service` channel.
 
## Notes
- A test event from `eventcreate` can appear under `base_sensor`, delivered by the sensor, so it doesn't prove the collector works. To test the collector alone, add a `file` source for a text file and search with `#repo!=base_sensor`.
- A domain controller is chatty. Note the daily ingest per host for your licensing estimate.
- Never commit the install command, token, or key. See [troubleshooting.md](troubleshooting.md) for common errors.
 
