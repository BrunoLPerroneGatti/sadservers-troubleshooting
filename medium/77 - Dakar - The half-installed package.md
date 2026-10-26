## Problem

The checkout platform's local health agent stopped working after a package upgrade was interrupted. The checkout-agent binary may already be on disk, but its systemd service is not healthy.  
  
Restore the package and service so the agent runs again and still comes back after a reboot. Do not wipe the package database under _/var/lib/dpkg_, and do not replace the service with a hand-written unit of your own.

## Solution

Check the package status:

```bash
dpkg -l checkout-agent
```

Output:

```bash
iU  checkout-agent 1.0 all
```

The status `iU` indicates that the package has been unpacked but not yet configured.

Check for incomplete package installations:

```bash
sudo dpkg --audit
```

Output:

```bash
The following packages have been unpacked but not yet configured.
They must be configured using dpkg --configure or the
configure menu option in dselect for them to work:
 checkout-agent       Checkout API health agent
```

Inspect the service:

```bash
systemctl status checkout-agent.service --no-pager -l
systemctl is-enabled checkout-agent.service
```

The service was inactive (`inactive (dead)`) and disabled.

Check that the package's files and original service unit exist:

```bash
dpkg -L checkout-agent
ls -l /lib/systemd/system/checkout-agent.service
```

The service unit and executable were already present on disk.

Complete the interrupted package configuration:

```bash
sudo dpkg --configure checkout-agent
```


Verify that the package is now correctly installed:

```bash
dpkg -l checkout-agent
sudo dpkg --audit
```

The package status changed from `iU` to `ii`, and `dpkg --audit` returned no output.

Reload systemd and enable the service:

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now checkout-agent.service
```

The `enable` operation configures the service to start automatically at boot, while `--now` starts it immediately.

Check the service status:

```bash
systemctl status checkout-agent.service --no-pager -l
```

Expected status:

```bash
Loaded: loaded (/usr/lib/systemd/system/checkout-agent.service; enabled; preset: enabled)
Active: active (running)
```

Verify that the service is enabled:

```bash
systemctl is-enabled checkout-agent.service
# Expected output:
enabled
```
