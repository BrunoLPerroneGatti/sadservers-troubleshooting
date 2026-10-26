## Problem

There's a web server access log file at /home/admin/access.log. The file consists of one line per HTTP request, with the requester's IP address at the beginning of each line (first column).  
  
Find what's the IP address that has the most requests in this file (there's no tie; the IP is unique). Write the solution into a file /home/admin/highestip.txt.

## Solution

```bash
# Count the occurrences of each IP, sort by frequency, and select the most frequent one
awk '{print $1}' /home/admin/access.log | sort | uniq -c | sort -nr | head -n 1

# Write the IP address to the required file
awk '{print $1}' /home/admin/access.log | sort | uniq -c | sort -nr | head -n 1 | awk '{print $2}' > /home/admin/highestip.txt
```