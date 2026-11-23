# SSH Brute Force Detection

| | |
|---|---|
| **Rule** | `poc_ssh_failed_logins` (correlation rule, Low severity) |
| **MITRE** | Credential Access, Brute Force (T1110) |
| **Source** | Linux syslog (`/var/log/auth.log`) |

## What it detects
Repeated failed SSH logins from one source IP in a short window.

## Query
```
"Failed password"
| regex("from (?<src_ip>\\d+\\.\\d+\\.\\d+\\.\\d+)")
| groupBy([src_ip], function=count())
| _count >= 5
```
Summary trigger, 5-minute schedule, 5-minute window.

## Test
```bash
ssh -p 2222 fakeuser@127.0.0.1    # wrong password, repeat 6+ times
```
Several usernames were tried (`fakeuser`, `fakeuser2`, `labadmin`).

## Result
The rule raised a detection with these details:

| Field | Value |
|---|---|
| Detection type | Correlation rule |
| Severity | Low |
| Tactic and technique | Credential Access via Brute Force |
| Vendor / source product | CrowdStrike, Linux / Linux Syslog |
| Category | Identity |
| Activity start | Oct. 15, 2026 16:28:12 |
| Activity end | Oct. 15, 2026 18:15:00 |
| Duration | 01:46:47 |
| Users involved | `fakeuser`, `fakeuser2`, `labadmin` |
| Source IP | `10.0.2.2` |
