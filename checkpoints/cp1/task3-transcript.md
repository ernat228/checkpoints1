## Investigation Transcript

| # | Evidence file (command) | What I expected | What I found | What I concluded |
|---|---|---|---|---|
| 1 | `00-README.txt` | Index of diagnostic files collected from `office-srv`. | Lists files `01` to `22` collected on `2026-11-16 09:42 UTC` as `aigerim`. | Start by checking failed systemd units and active timers. |
| 2 | `05-systemctl-failed.txt` (`systemctl --failed`) | `nightly-report.service` listed as failed. | Two failed units: `legacy-sync.service` and `nightly-report.service`. | Two leads; check both to rule out unrelated failures. |
| 3 | `10-systemctl-status-legacy-sync.txt` & `11-journalctl-legacy-sync.txt` | Possible link to the report job. | Description says `Legacy file sync (old office, no longer used)`, runs `ExecStart=/bin/false`, failed at boot on `Nov 09 09:51:22` (`status=1/FAILURE`) just like previous boots. | Dead end - unrelated legacy unit that always exits `1` on boot. Ruled out. |
| 4 | `06-systemctl-list-timers.txt` (`systemctl list-timers`) | `nightly-report.timer` active and firing every 2 minutes. | `nightly-report.timer` is active; last triggered `09:42:01 UTC` (12s ago), next in `1min 47s`. | Timer scheduling works properly; the failure happens when the service runs the script. |
| 5 | `08-systemctl-cat-nightly-report.txt` (`systemctl cat ...`) | Service unit pointing to the report script and timer set to 2 minutes. | `ExecStart=/usr/local/bin/nightly-report.sh` (`Type=oneshot`); timer has `OnBootSec=1min`, `OnUnitActiveSec=2min`, `WantedBy=timers.target`. | Unit files are valid and timer is configured to persist across reboots. |
| 6 | `07-systemctl-status-nightly-report.txt` (`systemctl status ...`) | Exit status showing why the service fails. | `Active: failed (Result: exit-code)`, `status=203/EXEC`, `Failed to execute /usr/local/bin/nightly-report.sh: No such file or directory`. | `203/EXEC` with `No such file or directory` means either the script path is missing or its `#!` interpreter path is invalid. |
| 7 | `09-journalctl-nightly-report.txt` (`journalctl -u ...`) | Exact timestamp when the service stopped working. | Succeeded every 2 minutes through `Nov 09 14:18:01`, then started failing with `status=203/EXEC` at `Nov 09 14:20:01`. | Something changed right around `Nov 09 14:20`. |
| 8 | `21-apt-history.txt` (`tail -n 12 /var/log/apt/history.log`) | Check if a package upgrade broke the service. | `apt full-upgrade -y` ran on `2026-11-09` from `09:38:12` to `09:41:37` (updating kernel, `systemd`, `bash`). | Dead end - the upgrade happened 4.5 hours earlier, and the report ran fine until `14:18`. |
| 9 | `20-ls-l-bin-bash.txt` (`which bash; ls -l /bin/bash`) | Confirm `/bin/bash` exists after the `bash` package upgrade. | `/usr/bin/bash` and `-rwxr-xr-x 1 root root 1446024 Mar 31 2024 /bin/bash`. | Dead end - `/bin/bash` exists and is executable. |
| 10 | `13-ls-l-usr-local-bin.txt` (`ls -l /usr/local/bin`) | Script missing or lacking `+x` permission. | `-rwxr-xr-x 1 root root 91 Nov  9 14:20 nightly-report.sh`. | Dead end - file exists with `755` permissions, and its mtime (`Nov 9 14:20`) matches the exact minute failures began. |
| 11 | `14-ls-l-var-log.txt` & `15-tail-nightly-report-log.txt` | Log file frozen right before `Nov 09 14:20`. | `-rw-r--r-- 1 root root 86412 Nov  9 14:18 nightly-report.log`; last entry is `Mon Nov  9 14:18:01 UTC 2026`. | Confirms log path and `root` write permissions work when the script runs. |
| 12 | `16-cat-nightly-report-sh.txt`, `17-cat-A-nightly-report-sh.txt` & `18-head-1-od-c.txt` | Inspect script contents and raw bytes on line 1. | `cat` looks normal (`#!/bin/bash`), but `cat -A` shows `#!/bin/bash^M$` on every line and `od -c` shows `# ! / b i n / b a s h \r \n`. | Root cause found: the script was saved with Windows CRLF (`\r\n`) line endings, so the kernel tries to execute `/bin/bash\r`, which does not exist. |
| 13 | `19-run-by-hand.txt` & `22-bash-history-aigerim.txt` | Confirm manual run error and see how the broken file got there. | Running the script directly or with `sudo` fails with `required file not found` / `No such file or directory` (`19`). `.bash_history` shows `aigerim` ran `sudo cp ~/nightly-report.sh /usr/local/bin/nightly-report.sh` and tried `chmod +x` / `chmod 755` (`22`). | Confirms `aigerim` copied a Windows-edited script from `~` into `/usr/local/bin/` at `14:20`, and the kernel rejects the `/bin/bash\r` shebang directly. |

## Root cause

On `Nov 9 14:20`, `aigerim` copied `~/nightly-report.sh` (which had been edited/saved with Windows CRLF `\r\n` line endings) over `/usr/local/bin/nightly-report.sh` (`13`, `22`). Because of the trailing `\r` on line 1 (`#!/bin/bash\r` shown in `17` and `18`), the Linux kernel fails to find the interpreter and returns `No such file or directory` (`19`), which systemd logs as `status=203/EXEC` (`07`, `09`). Since the script never starts, nothing has been appended to `/var/log/nightly-report.log` since `Nov 9 14:18` (`14`, `15`).

## Fix

```bash
# 1. Strip carriage return (\r) characters to convert CRLF to Unix LF
sudo sed -i 's/\r$//' /usr/local/bin/nightly-report.sh

# 2. Ensure executable permissions remain set
sudo chmod 755 /usr/local/bin/nightly-report.sh

# 3. Clear the failed unit state and run the service now
sudo systemctl reset-failed nightly-report.service
sudo systemctl start nightly-report.service
```

## Verification

1. **Confirm Unix LF endings and clean shebang:**
   - Command: `head -n 1 /usr/local/bin/nightly-report.sh | od -c`
   - Expected: `0000000   #   !   /   b   i   n   /   b   a   s   h  \n` (no `\r`).
2. **Confirm the service runs without `203/EXEC` and updates the log:**
   - Command: `systemctl status nightly-report.service && ls -l /var/log/nightly-report.log && tail -n 10 /var/log/nightly-report.log`
   - Expected: `status=0/SUCCESS`, updated timestamp on `/var/log/nightly-report.log`, and a new `date` + `df -h /` + `free -h` block at the bottom.
3. **Confirm automatic runs and persistence across reboot:**
   - Command: `systemctl is-enabled nightly-report.timer` (expected: `enabled`), then `sudo reboot`.
   - After reboot, wait ~2 minutes (`OnBootSec=1min` + `OnUnitActiveSec=2min`) and run `systemctl list-timers nightly-report.timer` and `tail -n 10 /var/log/nightly-report.log` to verify the timer triggers automatically and writes a fresh entry.
