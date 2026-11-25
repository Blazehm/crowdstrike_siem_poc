# Windows Account Created and Added to a Group Detection

| | |
|---|---|
| **Rule** | `windows_account_group_add_detection` (correlation rule, Medium severity) |
| **MITRE** | Persistence, Create Account (T1136) and Account Manipulation (T1098) |
| **Source** | Windows Security log from the domain controller (`LAB-DC`) |

## What it detects
An account that is created (event 4720) and added to a security group (4728, 4732, or 4756) within the same search window. Each event alone is routine, but together they match an attacker creating a backdoor account and giving it privileges.

## Query
```
#repo!=base_sensor (windows.EventID=4720 or windows.EventID=4728 or windows.EventID=4732 or windows.EventID=4756)
| case {
    windows.EventID=4720 | kind := "created" ;
    * | kind := "group_add" ;
  }
| groupBy([user.target.name], function=[count(kind, distinct=true, as=kinds), collect([user.target.group.name, user.name])])
| kinds >= 2
```
- `case` labels each event as a creation or a group add.
- The parser stores the affected account in `user.target.name` for both event types, so the two events line up by name.
- `groupBy` builds one row per account and counts how many different kinds were seen. An account with both gets `kinds = 2`.
- `collect` adds the group name and the acting user to the result.
- 10-minute search window (12:50 to 13:00 in the detection below), longer than the run frequency so pairs that straddle a run boundary are still caught. Summary trigger and 5-minute schedule [confirm against the saved rule].

## Test
On `LAB-DC`, create a disabled account, add it to the lab group, then clean up:
```powershell
New-ADUser -Name "tmpuser2" -SamAccountName tmpuser2 -Enabled $false
Add-ADGroupMember -Identity "SOC-Lab-Admins" -Members tmpuser2
Remove-ADGroupMember -Identity "SOC-Lab-Admins" -Members tmpuser2 -Confirm:$false
Remove-ADUser tmpuser2 -Confirm:$false
```

## Result
The rule raised a detection with these details:

| Field | Value |
|---|---|
| Rule | `windows_account_group_add_detection` |
| Detection type | Correlation rule |
| Status | Completed |
| Severity | Medium |
| Risk score | 100/100 (very rare: no recent occurrences of this detection type, hosts, or user) |
| Tactics and techniques | Persistence via Create Account; Persistence via Account Manipulation |
| Search window | Nov. 25, 2026 12:50:00 to 13:00:00 (10 minutes) |
| Detection completed | Nov. 25, 2026 13:03:32 (about 3.5 minutes after the window closed) |
| Account | `tmpuser2` |
| Acting user | `administrator` |
| Group | [`SOC-Lab-Admins`, confirm in the search log] |
| Host | `lab-dc.poclab.local` |

## Notes
- **Risk score.** The score of 100 reflects rarity in this small lab, where nothing like it had happened recently. It is not the same as severity.
- **No sensitive-group filter.** Any security group counts. A production rule would limit the group events to groups such as Domain Admins.
- **Name matching.** The rule joins the two events on `user.target.name`. Matching on the account's security ID would be sturdier if both events expose it.
- **Possible false positives:** onboarding scripts that create an account and add it to a group. Exclude known provisioning accounts using `user.name`.
- **Overlapping windows** can raise the same pair in two consecutive runs.
- Detections lag events by up to one schedule interval plus ingest delay.