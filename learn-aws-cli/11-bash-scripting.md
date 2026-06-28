# 11 — Bash Scripting with AWS CLI

## Best Practices

1. **Always use `--profile` or `AWS_PROFILE`** — never hardcode credentials
2. **Use `set -e`** to exit on first error
3. **Use `jq`** for JSON parsing (more reliable than grep/awk)
4. **Handle pagination** for large result sets
5. **Check return codes** — `$?` is `0` on success
6. **Quote variables** — always use `"$var"` to handle spaces
7. **Use `aws sts get-caller-identity`** to verify auth before proceeding

## Script Pattern 1: Snapshot EBS Volumes

```bash
#!/bin/bash
set -euo pipefail

PROFILE="${1:-default}"
REGION="${2:-us-west-2}"
INSTANCE_ID="${3:-}"

echo "Creating snapshots for instance: $INSTANCE_ID"

# Get all volumes attached to the instance
VOLUMES=$(aws ec2 describe-instances \
  --instance-ids "$INSTANCE_ID" \
  --profile "$PROFILE" \
  --region "$REGION" \
  --query 'Reservations[0].Instances[0].BlockDeviceMappings[*].Ebs.VolumeId' \
  --output text)

for vol in $VOLUMES; do
  echo "Backing up volume: $vol"
  aws ec2 create-snapshot \
    --volume-id "$vol" \
    --description "Auto-backup $(date +%Y-%m-%d)" \
    --profile "$PROFILE" \
    --region "$REGION"
done

echo "Done!"
```

## Script Pattern 2: S3 Website Deploy

```bash
#!/bin/bash
set -e

BUCKET="${1:?Usage: $0 <bucket-name>}"
BUILD_DIR="${2:-./dist}"

# Build the app (example: React/Vue)
if [ -f package.json ]; then
  npm run build
fi

# Sync to S3 with cache control
aws s3 sync "$BUILD_DIR" "s3://$BUCKET/" \
  --delete \
  --cache-control "max-age=3600" \
  --exclude "*.html" \
  --exclude "*.json"

# HTML files: no cache
aws s3 sync "$BUILD_DIR" "s3://$BUCKET/" \
  --delete \
  --include "*.html" \
  --include "*.json" \
  --cache-control "no-cache"

# CloudFront invalidation (if using CloudFront)
DIST_ID=$(aws cloudfront list-distributions \
  --query "DistributionList.Items[?Aliases.Items[?contains(@,'$BUCKET')]].Id" \
  --output text)

if [ -n "$DIST_ID" ]; then
  aws cloudfront create-invalidation \
    --distribution-id "$DIST_ID" \
    --paths "/*"
fi

echo "Deploy complete!"
```

## Script Pattern 3: Clean Old EBS Snapshots

```bash
#!/bin/bash
set -euo pipefail

DRY_RUN="${DRY_RUN:-true}"
RETENTION_DAYS="${RETENTION_DAYS:-30}"
CUTOFF=$(date -d "$RETENTION_DAYS days ago" +%Y-%m-%d)

echo "Finding snapshots older than $CUTOFF..."

OLD_SNAPS=$(aws ec2 describe-snapshots \
  --owner-ids self \
  --query "Snapshots[?StartTime<'${CUTOFF}T00:00:00Z'].SnapshotId" \
  --output text)

if [ -z "$OLD_SNAPS" ]; then
  echo "No old snapshots to clean."
  exit 0
fi

for snap in $OLD_SNAPS; do
  if [ "$DRY_RUN" = true ]; then
    echo "[DRY RUN] Would delete: $snap"
  else
    echo "Deleting snapshot: $snap"
    aws ec2 delete-snapshot --snapshot-id "$snap"
  fi
done

echo "Done! (dry_run=$DRY_RUN)"
```

## Script Pattern 4: Find Unused Security Groups

```bash
#!/bin/bash
set -euo pipefail

echo "Finding unused security groups..."

for sg in $(aws ec2 describe-security-groups \
  --query 'SecurityGroups[*].GroupId' --output text); do
  
  COUNT=$(aws ec2 describe-network-interfaces \
    --filters "Name=group-id,Values=$sg" \
    --query 'length(NetworkInterfaces)' \
    --output text)
  
  if [ "$COUNT" -eq 0 ]; then
    echo "UNUSED: $sg"
  fi
done
```

## Script Pattern 5: Check EC2 Instance Health

```bash
#!/bin/bash
set -euo pipefail

REGION="${1:-us-west-2}"

echo "Instance Health Report ($REGION):"
echo "================================="

aws ec2 describe-instances \
  --region "$REGION" \
  --query 'Reservations[*].Instances[*].{
    ID: InstanceId,
    Name: Tags[?Key==`Name`].Value|[0],
    State: State.Name,
    Type: InstanceType,
    IP: PublicIpAddress,
    Launched: LaunchTime
  }' \
  --output table

# Count by state
echo ""
echo "Summary:"
aws ec2 describe-instances \
  --region "$REGION" \
  --query 'Reservations[*].Instances[*].State.Name' \
  --output text | tr '\t' '\n' | sort | uniq -c | sort -rn
```

## Script Pattern 6: S3 Backup with Rotation

```bash
#!/bin/bash
set -euo pipefail

SRC_DIR="${1:?Source directory}"
BUCKET="${2:?Bucket name}"
PREFIX="backups/$(hostname)/$(date +%Y/%m/%d)"

echo "Backing up $SRC_DIR to s3://$BUCKET/$PREFIX/"

aws s3 sync "$SRC_DIR" "s3://$BUCKET/$PREFIX/" --sse AES256

# Send notification (optional)
aws sns publish \
  --topic-arn arn:aws:sns:us-west-2:123456789012:backup-notifications \
  --message "Backup completed: $SRC_DIR → s3://$BUCKET/$PREFIX/"
```

## Key Commands for Scripting

```bash
# Check auth
aws sts get-caller-identity

# Check region
aws configure get region

# Parse JSON safely
INSTANCE_ID=$(aws ec2 run-instances ... | jq -r '.Instances[0].InstanceId')

# Wait for state (built-in)
aws ec2 wait instance-running --instance-ids "$INSTANCE_ID"

# Tag resources
aws ec2 create-tags --resources "$INSTANCE_ID" --tags Key=Name,Value=MyServer

# Clean up with filters
aws ec2 terminate-instances --instance-ids \
  $(aws ec2 describe-instances --filters "Name=tag:Environment,Values=test" \
    --query 'Reservations[*].Instances[*].InstanceId' --output text)
```

---

### GUI Equivalent
No direct equivalent — scripting is unique to the CLI. The Console can do each action manually but cannot automate sequences.

---

**Congratulations! You've completed the AWS CLI learning path.** You now know everything from installation to advanced scripting. The original PDF was 377K+ lines — this is the condensed, actionable version.
