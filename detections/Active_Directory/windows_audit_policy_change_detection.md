# Windows Audit Policy Change Detection

| | |
|---|---|
| **Rule** | `windows_audit_policy_change_detection` (correlation rule, Medium severity) |
| **MITRE** | Defense Evasion, Impair Defenses (T1562) |
| **Source** | Windows Security log from the domain controller (`LAB-DC`) |

## What it detects
A change to the system's audit policy (event 4719). Attackers weaken or turn off auditing to hide their activity, so any change is worth a look.

## Query
```
#repo!="base_sensor" windows.EventID="4719"
```
- Matches a single event, with no counting or grouping.
- **Execution mode:** real-time, so the detection appears as the event is ingested, not on a schedule.
- **Suppression:** suppress all actions for 1 minute, so several changes in quick succession raise one detection.

## Test
On `LAB-DC`, enable and then disable an audit subcategory, which returns it to its original state:
```powershell
auditpol /set /subcategory:"Other Logon/Logoff Events" /success:enable
auditpol /set /subcategory:"Other Logon/Logoff Events" /success:disable
```

## Result
The rule raised a detection with these details:

| Field | Value |
|---|---|
| Rule | `windows_audit_policy_change_detection` |
| Detection type | Correlation rule |
| Severity | Medium |
| Risk score | 50/100 |
| Tactic and technique | Defense Evasion via Impair Defenses |
| Execution mode | Real-time |
| Suppression | Suppress all actions, 1-minute period |
| Start and end time | Nov. 24, 2026 10:42:10 (same moment) |
| User | `administrator` |
| Host | `lab-dc.poclab.local` |

The risk score of 50 comes from the detection type being too new to rate and the observed activity (Defense Evasion).

## Notes
- **Single-event rule.** Start and end times match because the rule alerts on one event. The two `auditpol` commands each log a 4719, and the 1-minute suppression turned them into one detection.
- **Execution modes.** This rule runs in real-time mode. The brute-force and spraying rules count events over a window, so they run on a schedule. As far as I know, real-time rules suit simple matches like this one, while rules that count or group events need a scheduled window.
- **Possible false positives:** legitimate administrator changes, and Group Policy updates that re-apply audit settings. Exclude known admin accounts or hosts, or alert only when auditing is turned off, if it gets noisy.
- **Stronger variant:** check which subcategory changed and whether success or failure logging was removed, so the rule fires only for changes that reduce visibility.
- The rule's description in the console reads "4179". The event ID is 4719, so correct the text.