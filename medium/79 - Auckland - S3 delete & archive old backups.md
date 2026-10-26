## Problem

Ops left years of database dumps in s3://auckland-backups. Keys use an absolute date prefix:  
backups/YYYY-MM-DD/...  
  
Apply this retention policy (use the date in the **key**, ignore LastModified):  

- **Before 2023-01-01** — **delete** those objects
- **From 2023-01-01 through 2024-12-31** — keep them but set storage class to GLACIER
- **On or after 2025-01-01** — leave as Standard
- Keys **without** that date prefix — leave untouched

The AWS CLI is ready (no configuration or authentication needed). You can start with:  
aws s3 ls s3://auckland-backups --recursive

## Solution

Inspect the bucket:

```bash
aws s3 ls s3://auckland-backups --recursive
```

Initial output:

```bash
backups/2020-06-15/db.dump
backups/2021-03-01/db.dump
backups/2022-12-31/db.dump
backups/2023-01-01/db.dump
backups/2023-07-15/db.dump
backups/2024-12-31/db.dump
backups/2025-01-01/db.dump
backups/2026-03-15/db.dump
backups/README.txt
current/latest.dat
```

The objects are grouped by the date embedded in their keys. `backups/README.txt` and `current/latest.dat` do not follow the dated backup naming convention and must remain untouched.

Delete the three objects dated before `2023-01-01`:

```bash
aws s3 rm s3://auckland-backups/backups/2020-06-15/db.dump
aws s3 rm s3://auckland-backups/backups/2021-03-01/db.dump
aws s3 rm s3://auckland-backups/backups/2022-12-31/db.dump
```

Archive backups from 2023 through 2024.
To change an existing S3 object's storage class, copy the object onto itself and specify the desired storage class using `--storage-class`:

```bash
aws s3 cp \
  s3://auckland-backups/backups/2023-01-01/db.dump \
  s3://auckland-backups/backups/2023-01-01/db.dump \
  --storage-class GLACIER

aws s3 cp \
  s3://auckland-backups/backups/2023-07-15/db.dump \
  s3://auckland-backups/backups/2023-07-15/db.dump \
  --storage-class GLACIER

aws s3 cp \
  s3://auckland-backups/backups/2024-12-31/db.dump \
  s3://auckland-backups/backups/2024-12-31/db.dump \
  --storage-class GLACIER
```

List the bucket again:

```bash
aws s3 ls s3://auckland-backups --recursive
```

The resulting listing contained:

```
backups/2023-01-01/db.dump
backups/2023-07-15/db.dump
backups/2024-12-31/db.dump
backups/2025-01-01/db.dump
backups/2026-03-15/db.dump
backups/README.txt
current/latest.dat
```

The three older backups were gone, while the archived backups and the objects that should remain untouched were still present.

Verify archived storage classes:

```bash
aws s3api head-object --bucket auckland-backups --key backups/2023-01-01/db.dump
aws s3api head-object --bucket auckland-backups --key backups/2023-07-15/db.dump
aws s3api head-object --bucket auckland-backups --key backups/2024-12-31/db.dump
```

All three returned:

```bash
"StorageClass": "GLACIER"
```

Verify Standard storage classes:

```bash
aws s3api head-object --bucket auckland-backups --key backups/2025-01-01/db.dump
aws s3api head-object --bucket auckland-backups --key backups/2026-03-15/db.dump
```

Both returned:

```bash
"StorageClass": "STANDARD"
```
