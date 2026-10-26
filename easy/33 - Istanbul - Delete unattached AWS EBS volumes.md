## Problem

Finance found orphaned EBS volumes racking up cost in this account. Your job is to find every volume that is **not attached** to an EC2 instance and **delete** it.  
  
Do **not** delete volumes that are attached (in-use). Do **not** stop or terminate instances.  
  
The AWS CLI is already configured against the local endpoint. Start with:  
aws ec2 describe-volumes  
  
NOTE: this scenario was done using Floci, a local AWS emulator; there are no real AWS resources.

## Solution

List all unattached EBS volumes in the AWS account using the AWS CLI. The `available` status indicates that a volume is not attached to an EC2 instance.

```bash
aws ec2 describe-volumes \
  --filters Name=status,Values=available \
  --query 'Volumes[*].VolumeId' \
  --output text
```

The command filters volumes by status and outputs only their volume IDs.

Delete every unattached volume by iterating over the returned IDs:

```bash
for vol in $(aws ec2 describe-volumes \
  --filters Name=status,Values=available \
  --query 'Volumes[*].VolumeId' \
  --output text); do
    echo "Deleting $vol"
    aws ec2 delete-volume --volume-id "$vol"
done
```

This loop retrieves all volumes with the `available` status, iterates over each volume ID and deletes each volume using `aws ec2 delete-volume`.