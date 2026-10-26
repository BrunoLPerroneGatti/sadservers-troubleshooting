## Problem

Monitoring reports that the root filesystem is under pressure, but a quick du of _/var/log_ shows almost nothing in the logs of the running application at _/var/log/cordoba-app_.  
  
Find what is holding the space and reclaim it so df and du agree again for practical purposes; currently there's a ~300 MB discrepancy on the root partition /  
  
The service unit is _cordoba-hoarder.service_.

## Solution

Check the root filesystem usage and compare the results from `df` and `du`:

```bash
df -h /
sudo du -sh /
```

`df` reports 5.1 GB used, while `du` reports only 4.7 GB. This discrepancy suggests that disk space is being consumed by something that `du` cannot see, commonly a deleted file that is still held open by a running process.

Inspect the service responsible for the challenge:

```bash
sudo systemctl status cordoba-hoarder.service --no-pager
sudo systemctl cat cordoba-hoarder.service
```

The service runs `/usr/local/sbin/cordoba-hoarder.sh` and has the environment variable `CORDOBA_LEAK_MB=400`, indicating that it is configured to create a disk-space leak.

Find deleted files that are still open:

```bash
sudo lsof +L1
```

The output reveals that the `cordoba-hoarder` process is holding the deleted log file `/var/log/cordoba-app/access.log` open, with a size of approximately 400 MB:

```bash
COMMAND    PID USER FD   TYPE DEVICE  SIZE/OFF NLINK   NODE NAME
cordoba-h 1168 root  9w   REG  259,1 419430456     0 281405 /var/log/cordoba-app/access.log (deleted)
```

Although `access.log` has been deleted, the `cordoba-hoarder` process still has an open file descriptor referencing it. On Linux, deleting a file removes its directory entry, not necessarily its contents from disk.

`du` calculates disk usage by traversing directory trees and summing the space used by files it can find. Since the deleted log file no longer has a directory entry, `du` cannot include its allocated blocks in its total.

`df`, on the other hand, queries the filesystem for its overall space usage. It includes blocks belonging to deleted files that are still open by a process, because those blocks cannot be reclaimed until the final open file descriptor is closed.


Stop the service to close the file descriptor and release the allocated disk space:

```bash
sudo systemctl stop cordoba-hoarder.service
```

Verify that the deleted log file is no longer held open:

```bash
sudo lsof +L1
```

The `cordoba-hoarder` entry should be gone. 

Compare the filesystem usage again:

```bash
df -h /
sudo du -sh /
```

Both commands now report approximately 4.7 GB used, confirming that the hidden disk space has been reclaimed and the challenge is solved.
