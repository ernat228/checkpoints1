## Investigation Transcript

| # | Evidence file (command) | What I expected | What I found | What I concluded |
|---|---|---|---|---|
| 1 | `00-README.txt` | List of commands collected from `office-srv`. | Files `01` to `14` covering systemd units, timers, logs, `/usr/local/bin/`, and `/var/log/`. | Start by checking failed systemd services. |
| 2 | `05 (systemctl --failed)` | `nightly-report` showing as failed. | Two failed units: `legacy-sync.service` and `nightly-report.service`. | Two leads; check both to see which one belongs to `/var/log/nightly-report.log`. |
| 3 | `06 (systemctl status legacy-sync.service)` | Maybe related to the report job. | Old FTP archive sync job that failed over 40 days ago (`exit-code 1`). | Dead end - unrelated to last week's script update. Ruled out. |
| 4 | `07 (systemctl list-timers --all)` | `nightly-report.timer` active and running every 2 minutes. | `nightly-report.timer` is `active (waiting)`, last triggered <2m ago, next in <2m. | Timer works fine; the failure happens when the service runs the script. |
| 5 | `08 (systemctl status nightly-report.service)` | Error code showing why the script isn't writing logs. | `Active: failed (Result: exit-code)`, `ExecStart=/usr/local/bin/nightly-report.sh (code=exited, status=203/EXEC)`. | `203/EXEC` means systemd cannot execute the script (wrong permissions, missing file, or bad shebang). |
| 6 | `10 (ls -la /usr/local/bin/)` | Missing `+x` permission after someone replaced the script last week. | `-rwxr-xr-x 1 root root 412 ... nightly-report.sh`. | Dead end - permissions are `755` (`+x` is already there). Problem is inside the file itself. |
| 7 | `11 (cat -A /usr/local/bin/nightly-report.sh)` | Normal `#!/bin/bash` line at the top. | First line is `#!/bin/bash^M$` and all lines end with `^M$` (Windows CRLF). | Found the bug: the updated script was saved on Windows with `\r\n` endings, so Linux tries to run `/bin/bash\r` and fails with `203/EXEC`. |
| 8 | `12 (ls -la /var/log/nightly-report.log)` | Old log file untouched since last week. | `-rw-r--r-- 1 root root 18420 ...` modified 8 days ago. | Confirms the log path and `root` permissions are fine once the script actually runs. |

## Root cause

When `/usr/local/bin/nightly-report.sh` was updated last week, it was saved with Windows CRLF (`\r\n`) line endings (`11`), which broke the `#!/bin/bash` shebang and caused systemd to fail with `status=203/EXEC` (`08`). Since the script never ran, nothing was written to `/var/log/nightly-report.log` (`12`).

## Fix

```bash
# Convert CRLF line endings to Unix LF
sudo sed -i 's/\r$//' /usr/local/bin/nightly-report.sh

# Make sure execute permission is still set
sudo chmod 755 /usr/local/bin/nightly-report.sh

# Reset failed state and run the service now
sudo systemctl reset-failed nightly-report.service
sudo systemctl start nightly-report.service
