# Linux Sensitive File Change Detection

| | |
|---|---|
| **Rule** | `linux_sensitive_file_change_detection` (correlation rule, High severity) |
| **MITRE** | Persistence, Account Manipulation (T1098) and Create Account (T1136.001) [intended mapping, not yet set on the rule] |
| **Source** | Linux auditd log (`/var/log/audit/audit.log`) from `cspoclinuxserver` |

## What it detects
A write to a file that controls accounts and privileges: `/etc/passwd`, `/etc/shadow`, or the sudoers configuration. Attackers edit these files to add accounts or grant themselves root access.

## Audit rules
The server's auditd watches the files and tags each write with a key:
```
-w /etc/passwd -p wa -k passwd_changes
-w /etc/shadow -p wa -k shadow_changes
-w /etc/sudoers -p wa -k sudoers_changes
-w /etc/sudoers.d/ -p wa -k sudoers_changes
```

## Query
```
#repo!=base_sensor #event.dataset=auditd.syscall rule.name=/^(passwd|shadow|sudoers)_changes$/
| groupBy([rule.name, process.executable, user.name], function=count())
```
- `rule.name` holds the audit key, `process.executable` is the program that wrote the file, and `user.name` is the user who originally logged in (it stays the same after `sudo`).
- Filtering on `auditd.syscall` excludes `CONFIG_CHANGE` records, which carry the same key whenever audit rules are loaded and would otherwise cause false alerts.
- 5-minute search window (13:25 to 13:30 in the detection below). Summary trigger and 5-minute schedule [confirm against the saved rule].

## Test
```bash
sudo useradd -m auditlab
sudo userdel -r auditlab
```

## Result
The rule raised a detection with these details:

| Field | Value |
|---|---|
| Rule | `linux_sensitive_file_change_detection` |
| Description | "detects changes in sensitive file (passwd, shadow, sudoers)" |
| Detection type | Correlation rule |
| Status | Completed (detection status: New, unassigned) |
| Severity | High |
| Risk score | 100/100 (very rare: no recent occurrences of this detection type, and users 0 and 1000 are both in the top 1% rarest) |
| Search window | Nov. 26, 2026 13:25:00 to 13:30:00 (5 minutes) |
| Detection start / end time | Nov. 26, 2026 13:27:16 / 13:30:00 |
| Detection completed | Nov. 26, 2026 13:31:12 (about 1 minute after the window closed) |
| Results | 4 rows (run duration 3 s) |
| Users | `labadmin`, `root` |
| Audit keys | [`passwd_changes`, `shadow_changes`, check the search log] |
| Programs | [`/usr/sbin/useradd`, `/usr/sbin/userdel`, check the search log] |
| Host | [`cspoclinuxserver`] |

## Notes
- One change produces several audit records (for example an open and a rename of `/etc/passwd`), so counts are higher than 1.
- Legitimate admin tools such as `useradd`, `usermod`, and `passwd` trigger the rule too. The `process.executable` column shows who made the change, so a text editor or script touching `/etc/shadow` stands out.
- Possible false positives: package installs and routine administration. Exclude known tools or accounts if it gets noisy.
- **MITRE mapping.** The detection shows no tactic or technique, so the mapping was not set on the rule. Add Persistence with Account Manipulation and Create Account in the rule editor.
- **Users.** `labadmin` is the login user and `root` is the account the commands ran as after `sudo`.
- **Risk score.** The score of 100 reflects rarity in this small lab. It is not the same as severity.
- Detections lag events by up to one schedule interval plus ingest delay. Here the detection completed about 1 minute after the search window closed.