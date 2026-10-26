## Problem

A developer created a testing program that is continuously writing to a log file _/var/log/bad.log_ and filling up disk.
This program is no longer needed. Find it and terminate it. Do not delete the log file.

## Solution

```bash
lsof /var/log/bad.log # find the PID of the process writing to the log file
kill <PID>
```