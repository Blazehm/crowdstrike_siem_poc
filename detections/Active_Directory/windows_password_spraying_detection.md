# Windows Password Spraying Detection

| | |
|---|---|
| **Rule** | `windows_password_spraying_detection` (correlation rule, Medium severity) |
| **MITRE** | Credential Access, Brute Force: Password Spraying (T1110.003) |
| **Source** | Windows Security log from the domain controller (`LAB-DC`) |

## What it detects
One source making failed logons against many different accounts, with only a few attempts per account. Attackers do this to stay under account lockout thresholds. The [password brute force rule](password_bruteforce.md) needs 4 failures on a single account, so it cannot see this pattern.

## Query
```
#repo!=base_sensor (windows.EventID=4771 or windows.EventID=4625)
| groupBy([source.address], function=[count(as=failures), count(user.target.name, distinct=true, as=accounts)])
| accounts >= 3
```
- **4771:** Kerberos pre-authentication failed. **4625:** failed logon (NTLM).
- Groups failures by source address and counts how many **distinct accounts** each source tried.
- The threshold uses `accounts`, so an attempt that logs more than one event doesn't skew it.
- 3 accounts is a low threshold for the lab. A production rule would use a higher number over a longer window.
- 5-minute search window (16:30 to 16:35 in the detection below). Summary trigger and 5-minute schedule [confirm against the saved rule].

## Test
From HOST01, one wrong password against each of three lab accounts. One failure per account stays well below the lockout threshold of 5:
```powershell
"labuser2","labuser3","labuser4" | ForEach-Object { net use \\LAB-DC\IPC$ "/user:POCLAB\$_" WrongPass1! }
net use * /delete /y
```
The accounts are used only for this test and had no recent failures, so nothing locks.

## Result
The rule raised a detection with these details:

| Field | Value |
|---|---|
| Rule | `windows_password_spraying_detection` |
| Detection type | Correlation rule |
| Severity | Medium |
| Tactic and technique | Credential Access via Password Spraying |
| Search window | Nov. 23, 2026 16:30:00 to 16:35:00 (5 minutes) |
| Failed logon events | Within the search window |
| Detection completed | Nov. 23, 2026 16:38:11 (about 3 minutes after the window closed) |
| Accounts targeted | `labuser2`, `labuser3`, `labuser4` |
| Distinct accounts / failures | 3 / [failures, check the search log] |
| Source address | `::ffff:10.10.10.20` (HOST01, IPv6-mapped form) |
| Host | `lab-dc.poclab.local` |

## Notes
- **Different shape from brute force.** Brute force counts failures per account, while spraying counts accounts per source. One attempt per account never trips the brute-force rule, so the two rules complement each other.
- Failures against accounts that don't exist are not counted by this rule. Kerberos logs those differently (event 4768), so a spraying tool that guesses usernames needs another rule.
- The source address can appear in IPv6-mapped form (`::ffff:10.10.10.20`), which can split one source into two rows.
- Possible false positives: a shared workstation where several people mistype passwords, or a misconfigured service trying many accounts. Raise the threshold or exclude known sources if it gets noisy.
- Detections lag events by up to one schedule interval plus ingest delay. Here the detection completed about 3 minutes after the search window closed.