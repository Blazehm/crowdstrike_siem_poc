
# Windows DC and Host Setup
 
Builds an isolated Active Directory domain, `poclab.local`, on VirtualBox: a Server Core domain controller (`LAB-DC`) and a Desktop Experience member server (`HOST01`). Both run the Falcon sensor.
 
## Layout
 
| | LAB-DC | HOST01 |
|---|---|---|
| **OS** | Windows Server 2022 Core | Windows Server 2022 Desktop Experience |
| **Resources** | 2 GB RAM | 2 GB RAM |
| **VirtualBox Adapter 1** | Internal Network `soclab` | Internal Network `soclab` |
| **VirtualBox Adapter 2** | NAT | NAT |
| **LAB adapter** | `10.10.10.10/24`, no gateway | `10.10.10.20/24`, no gateway |
| **LAB DNS** | `10.10.10.10`, `127.0.0.1` | `10.10.10.10` |
| **NAT adapter** | DHCP, DNS `127.0.0.1` | DHCP, DNS `10.10.10.10` |
| **Role** | Domain controller, collector, sensor | Member server, sensor |
 
The internal network name must match on both VMs. Only the NAT adapter has internet access, which the sensor and collector use.
 
## 1. LAB-DC: network
Find the adapters by address (the NAT adapter has a `10.0.x.15` address), then rename and configure them:
```powershell
Get-NetIPAddress -AddressFamily IPv4 | select InterfaceAlias, InterfaceIndex, IPAddress
Get-NetAdapter -InterfaceIndex <lab-index> | Rename-NetAdapter -NewName "LAB"
Get-NetAdapter -InterfaceIndex <nat-index> | Rename-NetAdapter -NewName "NAT"
Rename-Computer -NewName "LAB-DC" -Restart
```
After the restart:
```powershell
New-NetIPAddress -InterfaceAlias "LAB" -IPAddress 10.10.10.10 -PrefixLength 24
Set-DnsClientServerAddress -InterfaceAlias "LAB" -ServerAddresses 10.10.10.10,127.0.0.1
Set-DnsClientServerAddress -InterfaceAlias "NAT" -ServerAddresses 127.0.0.1
Set-DnsClient -InterfaceAlias "NAT" -RegisterThisConnectionsAddress $false
Set-TimeZone -Id "India Standard Time"
```
 
## 2. LAB-DC: promote to a domain controller
```powershell
Install-WindowsFeature AD-Domain-Services -IncludeManagementTools
Install-ADDSForest -DomainName "poclab.local" -DomainNetbiosName "POCLAB" -InstallDns
```
Set a recovery (DSRM) password when asked. The server restarts. Then add a DNS forwarder so the DC can resolve internet names:
```powershell
Add-DnsServerForwarder -IPAddress 8.8.8.8
Resolve-DnsName crowdstrike.com
Test-NetConnection crowdstrike.com -Port 443
```
 
## 3. LAB-DC: lab objects and lockout policy
```powershell
New-ADOrganizationalUnit -Name "SOC-Lab" -Path "DC=poclab,DC=local"
New-ADGroup -Name "SOC-Lab-Admins" -GroupScope Global -Path "OU=SOC-Lab,DC=poclab,DC=local"
New-ADUser -Name "Lab User 1" -SamAccountName labuser1 -Path "OU=SOC-Lab,DC=poclab,DC=local" -AccountPassword (Read-Host -AsSecureString "Password") -Enabled $true
Set-ADDefaultDomainPasswordPolicy -Identity poclab.local -LockoutThreshold 5 -LockoutObservationWindow 00:10:00 -LockoutDuration 00:10:00
```
Run failed-logon tests only against `labuser1`, never `Administrator`.
 
## 4. Auditing (both machines)
```powershell
$subs = "Logon","Special Logon","Credential Validation","Kerberos Service Ticket Operations","Security Group Management","User Account Management","Audit Policy Change"
foreach ($s in $subs) { auditpol /set /subcategory:"$s" /success:enable /failure:enable }
auditpol /set /subcategory:"Process Creation" /success:enable
reg add "HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System\Audit" /v ProcessCreationIncludeCmdline_Enabled /t REG_DWORD /d 1 /f
```
On the DC, also run `auditpol /set /subcategory:"Kerberos Authentication Service" /success:enable /failure:enable`. Check the result with `auditpol /get /category:*`.
 
## 5. HOST01: network and domain join
Identify and rename the adapters as in step 1, then:
```powershell
New-NetIPAddress -InterfaceAlias "LAB" -IPAddress 10.10.10.20 -PrefixLength 24
Set-DnsClientServerAddress -InterfaceAlias "LAB" -ServerAddresses 10.10.10.10
Set-DnsClientServerAddress -InterfaceAlias "NAT" -ServerAddresses 10.10.10.10
Set-DnsClient -InterfaceAlias "NAT" -RegisterThisConnectionsAddress $false
Set-TimeZone -Id "India Standard Time"
Rename-Computer -NewName "HOST01" -Restart
```
Test, then join:
```powershell
Test-NetConnection 10.10.10.10 -Port 445
Resolve-DnsName poclab.local
Add-Computer -DomainName poclab.local -Credential POCLAB\Administrator -Restart
```
Apply the auditing commands from step 4 on HOST01 after the restart.
 
## 6. Falcon sensor (both machines)
Download the Windows sensor on HOST01, copy it to the DC, and install on each with the PoC tenant's CID:
```powershell
New-Item -ItemType Directory -Path \\LAB-DC\C$\Temp -Force
Copy-Item .\WindowsSensor.exe \\LAB-DC\C$\Temp\
.\WindowsSensor.exe /install /quiet /norestart CID=<YOUR_CID>
```
On the DC, run `C:\Temp\WindowsSensor.exe` with the same options. Confirm with `Get-Service CSFalconService` and check that both hosts appear in the console.
 
## 7. Verify
On `LAB-DC`:
```powershell
Get-ADComputer HOST01 | select Name, Enabled
Get-DnsServerResourceRecord -ZoneName poclab.local -Name HOST01 -RRType A
dcdiag /q
```
HOST01 should be listed and enabled, its DNS record should show `10.10.10.20` only, and `dcdiag /q` should print nothing. After an unclean shutdown, `dcdiag` can flag `DFSREvent` and `SystemLog` for 24 hours, which is harmless.
 
## Notes
- Shut the VMs down cleanly (`Stop-Computer`), not with Power Off.
- Paste doesn't work on the Core DC. Manage it from HOST01 with `Enter-PSSession -ComputerName LAB-DC -Credential POCLAB\Administrator`.
- Never commit your CID or tokens. See [troubleshooting.md](troubleshooting.md) for common errors.
 
