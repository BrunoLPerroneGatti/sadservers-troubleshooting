## Problem

 Can't ping google.com. It returns ping: google.com: Name or service not known. Expected is being able to resolve the hostname.

## Solution

Inspect the DNS configuration. Check `/etc/resolv.conf`:

```bash
cat /etc/resolv.conf
ls -l /etc/resolv.conf
```

The file contained:

```conf
nameserver 127.0.0.53
options edns0 trust-ad
search .
```

`/etc/resolv.conf` was a symbolic link to:

```
/run/systemd/resolve/stub-resolv.conf
```

This is a normal configuration for `systemd-resolved`, which provides a local DNS stub resolver.

Inspect the hostname lookup configuration:

```bash
grep '^hosts:' /etc/nsswitch.conf
```

The output was:

```conf
hosts:          files
```

The `hosts` entry instructed the name service switch (NSS) to look up hostnames only in local files, such as `/etc/hosts`, without consulting DNS.

Check the status of `systemd-resolved`:

```bash
systemctl status systemd-resolved --no-pager
```

The service was `active (running)`.

Inspect the configured DNS servers:

```bash
resolvectl status
```

The `ens5` interface had the following DNS server configured:

```
DNS Servers: 10.1.0.2
```

The required resolver libraries were also present:

```
ls -l /lib/x86_64-linux-gnu/libnss_dns.so.2 \
      /lib/x86_64-linux-gnu/libnss_resolve.so.2
```

These checks indicated that the resolver service and DNS configuration existed, the immediate issue was the NSS hosts configuration.

Fix the NSS configuration. Edit `/etc/nsswitch.conf`.
Change the hosts line from:

```
hosts:          files
```

to:

```
hosts:          files dns
```

This preserves local hostname lookups through files and enables DNS lookups through the NSS DNS module.

Test the connection:

```bash
ping -c 2 google.com
```

The hostname resolved successfully to an IPv4 address:

```
PING google.com (142.251.214.78) 56(84) bytes of data.
```