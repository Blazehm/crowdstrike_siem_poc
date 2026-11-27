# Linux Cron Change Detection

| | |
|---|---|
| **Rule** | `linux_cron_change_detection` (correlation rule, Medium severity) |
| **MITRE** | Persistence, Scheduled Task/Job (T1053); the cron sub-technique is T1053.003 |
| **Source** | Linux auditd log (`/var/log/audit/audit.log`) from `cspoclinuxserver` |

## What it detects
A new or modified cron entry. Attackers add cron jobs so their code runs again after a reboot or at regular intervals, which is a common way to keep access on Linux.

## Audit rules
```
-w /etc/crontab -p wa -k cron_changes
-w /etc/cron.d/ -p wa -k cron_changes
-w /var/spool/cron/crontabs/ -p wa -k cron_changes
```
The last line covers per-user crontabs edited with `crontab -e`.

## Query
```
#repo!=base_sensor #event.dataset=auditd.syscall rule.name=cron_changes
| groupBy([process.executable, user.name], function=count())
```
- Matches audit records tagged `cron_changes`, then shows the program that wrote the file and the user who originally logged in.
- Filtering on `auditd.syscall` excludes `CONFIG_CHANGE` records written when audit rules are loaded.
- 5-minute search window (14:15 to 14:20 in the detection below). Summary trigger and 5-minute schedule [confirm against the saved rule].

## Test
Create a harmless job that runs `/bin/true`, then remove it:
```bash
echo "* * * * * root /bin/true" | sudo tee /etc/cron.d/labtest
sudo rm /etc/cron.d/labtest
```

## Result
The rule raised a detection with these details:

| Field | Value |
|---|---|
| Rule | `linux_cron_change_detection` |
| Description | "detects key cron_changes" |
| Detection type | Correlation rule |
| Status | Completed (detection status: New, unassigned) |
| Severity | Medium |
| Tactic and technique | Persistence via Scheduled Task/Job |
| Search window | Nov. 27, 2026 14:15:00 to 14:20:00 (5 minutes) |
| Detection start / end time | Nov. 27, 2026 14:18:14 / 14:20:00 |
| Detection completed | Nov. 27, 2026 14:21:59 (about 2 minutes after the window closed) |
| Results | 2 rows (run duration 1 s) |
| Users | `labadmin`, `root` |
| Programs | [`/usr/bin/tee`, `/usr/bin/rm`, check the search log] |
| Host | [`cspoclinuxserver`] |

## Notes
- Creating a file with `tee` and deleting it with `rm` are separate events, so the detection shows both programs.
- Possible false positives: package installs and system maintenance routinely add files under `/etc/cron.d`. Exclude known package-manager activity if it gets noisy.
- **Risk score.** The score is 1/100 because the same users had just appeared in earlier detections. The sensitive-file detection scored 100 when it was new, so the score reflects rarity and falls as tests repeat. It is not the same as severity.
- **Users.** `labadmin` is the login user and `root` is the account the commands ran as after `sudo`.
- Detections lag events by up to one schedule interval plus ingest delay. Here the detection completed about 2 minutes after the search window closed.