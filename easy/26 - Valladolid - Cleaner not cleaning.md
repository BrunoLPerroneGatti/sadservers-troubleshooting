## Problem

The systemd service _log-cleaner.service_ is supposed to be run manually (not a timer or cron job) and delete log files older than 7 days in the _/var/log/app_ directory.  
  
The service runs successfully (exit code 0), but no logs are ever deleted.  
  
Fix the service and/or the script so that _old_data.log_ (older than 7 days) is deleted, but _recent_data.log_ is preserved.  
  
If you accidentally delete the wrong files while debugging, run ~/reset_logs.sh to restore them.
## Solution

Verify the systemd service:

```bash
systemctl cat log-cleaner
```

```ini
# /etc/systemd/system/log-cleaner.service
[Unit]
Description=Daily Log Cleaner

[Service]
Type=oneshot
ExecStart=/bin/bash /opt/scripts/log-cleaner.sh
WorkingDirectory=/root
```

Verify the script:

```bash
cat /opt/scripts/log-cleaner.sh
```

```bash
#!/bin/bash

# Configuration
LOG_DIR="/var/log/app"
DAYS=7

echo "Starting Cleanup..."

find . -maxdepth 1 -name "*.log" -type f -mtime -7 -print -delete

echo "Cleanup finished."
```

The script has two issues:
- The `find` command searched `.` (the current working directory). Because the systemd service specifies `WorkingDirectory=/root`, it searched `/root` instead of `/var/log/app`. 
- ``-mtime -7`` removed newer files modified less than seven 7 days ago.

We need to update the script:
```bash
find $LOG_DIR -maxdepth 1 -name "*.log" -type f -mtime +$DAYS -print -delete
```

Then restart the service:
```bash
sudo systemctl restart log-cleaner
```

