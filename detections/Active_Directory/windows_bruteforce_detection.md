# Windows Password Brute Force Detection

| | |
|---|---|
| **Rule** | `windows_bruteforce_detection` (correlation rule, Medium severity) |
| **MITRE** | Credential Access, Brute Force: Password Guessing (T1110.001) |
| **Source** | Windows Security log from the domain controller (`LAB-DC`) |

## What it detects
Repeated failed logons against one account from one source in a short window, the usual sign of password guessing against a domain account.

## Query
```
#repo!=base_sensor (windows.EventID=4771 or windows.EventID=4625)
| groupBy([user.target.name, source.address, windows.EventID], function=count())
| _count >= 4
```
- **4771:** Kerberos pre-authentication failed (connecting by name from another machine).
- **4625:** failed logon (NTLM, for example connecting by IP or running on the DC itself).
- Grouping by event ID counts each kind of failure separately, so one attempt that logs several events doesn't inflate the total.
- 5-minute search window (13:50 to 13:55 in the detection below). Summary trigger and 5-minute schedule [confirm against the saved rule].

## Test
From HOST01, four wrong passwords against a lab account. The domain locks accounts at 5 failures, so four stays below it:
```powershell
1..4 | ForEach-Object { net use \\LAB-DC\IPC$ /user:POCLAB\labuser1 WrongPass1! ; Start-Sleep 2 }
net use * /delete /y
```

## Result
The rule raised a detection with these details:

| Field | Value |
|---|---|
| Rule | `windows_bruteforce_detection` |
| Detection type | Correlation rule |
| Severity | Medium |
| Tactic and technique | Credential Access via Brute Force |
| Search window | Nov. 23, 2026 13:50:00 to 13:55:00 (5 minutes) |
| Failed logon event | Nov. 23, 2026 13:54:51 |
| Detection completed | Nov. 23, 2026 13:58:13 (about 3 minutes after the event) |
| Account | `labuser1` |
| Source address | `::ffff:10.10.10.20` (HOST01, IPv6-mapped form) |
| Event dataset / outcome | `windows.security` / failure |

## Notes
- The event depends on the protocol: Kerberos failures log 4771, NTLM failures log 4625 (with 4776). The rule covers both.
- The source address appears in IPv6-mapped form (`::ffff:10.10.10.20`, as in the detection above), which can split one source into two rows.
- Possible false positives: mistyped passwords and stale credentials on services. Raise the threshold or exclude known service accounts if it gets noisy.
- A threshold of 4 sits just under the lockout threshold of 5, so the detection fires before the account locks.
- Detections lag events by up to one schedule interval plus ingest delay. Here the detection completed about 3 minutes after the last failed logon.