## Problem

This host publishes a machine-readable status marker using Terraform with a local backend. The project lives in _/home/admin/infra/_ and should write _/var/local/platform-status.txt_.  
  
After a refactor, terraform plan and terraform apply no longer succeed, and the status file is missing or stale.  
  
Fix the Terraform project and apply it so the marker is published again.  
  
(Note: Internet access is not needed).

## Solution

Run Terraform plan command to identify why the project is failing:

```bash
terraform plan
```

Terraform reports an unsupported version error:

```bash
Error: Unsupported Terraform Core version

on versions.tf line 2, in terraform:
2: required_version = "~> 1.12.0"

This configuration does not support Terraform version 1.9.8.
```

Check the installed Terraform version and inspect `versions.tf`:

```bash
terraform version
cat versions.tf
```

The installed version is `1.9.8`, but the project requires Terraform `1.12.0`. Since internet access is not needed for this challenge, fix the version constraint to accept the installed version rather than installing a different Terraform release.

Run the plan again:

```bash
terraform plan
```

Terraform reports a new error:

```bash
Error: Cycle: local_file.platform_status, local_file.status_body
```

This reveals a problem in `main.tf`. The original configuration defines two `local_file` resources that reference each other's content:

```
resource "local_file" "status_body" {
  filename = "/var/local/.platform-status-body"
  content  = local_file.platform_status.content
}

resource "local_file" "platform_status" {
  filename        = "/var/local/platform-status.txt"
  content         = local_file.status_body.content
  file_permission = "0644"
}
```

This creates a circular dependency: `status_body` depends on `platform_status`, while `platform_status` depends on `status_body`. Terraform cannot determine the order in which to create the resources and reports a cycle error.

Fix the dependency by assigning the required status text directly to `status_body`, allowing `platform_status` to consume its content:

```
resource "local_file" "status_body" {
  filename = "/var/local/.platform-status-body"
  content  = "SadServers - Parma OK\n"
}

resource "local_file" "platform_status" {
  filename        = "/var/local/platform-status.txt"
  content         = local_file.status_body.content
  file_permission = "0644"
}
```

The dependency now flows in one direction. Terraform creates the body file first and then uses its content to generate the required status marker.

Format and validate the configuration:

```bash
terraform fmt
terraform validate
```

Generate and review the execution plan:

```bash
terraform plan
```

The plan should show two resources to create: `local_file.status_body` and `local_file.platform_status`.

Apply the configuration:

```bash
terraform apply
```

Terraform should successfully create both files.

Verify the required status marker:

```bash
head -n 1 /var/local/platform-status.txt
# Expected output:
SadServers - Parma OK
```
