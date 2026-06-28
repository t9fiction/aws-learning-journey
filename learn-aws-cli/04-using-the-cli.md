# 04 — Using the CLI: Commands, Parameters & Output

## Command Structure

Every AWS CLI command follows this pattern:

```
aws <service> <operation> [parameters] [--options]
```

Examples:
```bash
aws ec2 describe-instances
aws s3 ls
aws iam list-users --output table
aws dynamodb put-item --table-name MyTable --item '{"id": {"S": "123"}}'
```

## Help

```bash
# General help
aws help

# Service help
aws ec2 help

# Command help
aws ec2 run-instances help
```

## Parameter Types

### 1. String parameters
```bash
aws ec2 describe-instances --instance-ids i-1234567890abcdef0
```

### 2. Quoted strings (with spaces/special chars)
```bash
aws ec2 create-security-group \
  --group-name "my security group" \
  --description "My security group description"
```

### 3. Number parameters
```bash
aws ec2 modify-volume --volume-id vol-123 --size 100
```

### 4. Boolean parameters
```bash
aws s3api put-bucket-versioning \
  --bucket my-bucket \
  --versioning-configuration Status=Enabled
```

### 5. List parameters
```bash
# JSON array
aws ec2 terminate-instances --instance-ids '["i-123","i-456"]'

# Space-separated
aws ec2 terminate-instances --instance-ids i-123 i-456
```

### 6. Map / JSON parameters
```bash
aws dynamodb put-item \
  --table-name MyTable \
  --item '{"pk": {"S": "user1"}, "name": {"S": "Alice"}}'
```

## Skeleton Template

Generate a JSON template for any command, then fill it in:

```bash
# Get the skeleton
aws ec2 run-instances --generate-cli-skeleton > skeleton.json

# Edit skeleton.json with your values, then run:
aws ec2 run-instances --cli-input-json file://skeleton.json
```

## Output Formats

### `json` (default)
```bash
aws ec2 describe-instances --output json
```

### `yaml`
```bash
aws ec2 describe-instances --output yaml
```

### `text` (tab-separated, good for grep/awk)
```bash
aws ec2 describe-instances --output text
```

### `table` (pretty-printed, human-readable)
```bash
aws ec2 describe-instances --output table
```

## Filtering Output with `--query`

Uses JMESPath expressions to pick specific fields:

```bash
# Get only instance IDs
aws ec2 describe-instances --query 'Reservations[*].Instances[*].InstanceId'

# Filter by tag
aws ec2 describe-instances --query \
  'Reservations[*].Instances[?Tags[?Key==`Name`].Value == `web-server`].InstanceId'

# Get specific fields as a table
aws ec2 describe-instances --query \
  'Reservations[*].Instances[*].{ID:InstanceId,Type:InstanceType,State:State.Name}' \
  --output table
```

## Pagination

CLI automatically paginates results. Control it with:

```bash
# Set max items per page
aws s3api list-objects --bucket my-bucket --page-size 100

# Limit total results
aws ec2 describe-instances --max-items 50

# Manual pagination with NextToken
aws ec2 describe-instances --max-items 10 > page1.json
# ... extract NextToken from page1.json ...
aws ec2 describe-instances --max-items 10 --starting-token <token>
```

Or disable pagination entirely:
```bash
aws s3api list-objects --bucket my-bucket --no-paginate
```

## Return Codes

| Code | Meaning |
|------|---------|
| `0` | Success |
| `255` | Error (command failed) |
| `250+` | Service-specific errors |

```bash
# Example: check if command succeeded
aws ec2 describe-instances && echo "Success" || echo "Failed"
```

## Shorthand Syntax

Instead of JSON, use shorthand for simple structures:

```bash
# JSON
aws ec2 create-tags --resources i-123 --tags '[{"Key":"Name","Value":"MyServer"}]'

# Same with shorthand (much cleaner)
aws ec2 create-tags --resources i-123 --tags 'Key=Name,Value=MyServer'
```

## File Parameters

Pass parameters from files:

```bash
# From a JSON file
aws ec2 run-instances --cli-input-json file://mydata.json

# From stdin
aws ec2 run-instances --cli-input-json file:///dev/stdin
```

## Wait Commands

Poll AWS until a resource reaches a desired state:

```bash
# Wait for an EC2 instance to be running
aws ec2 wait instance-running --instance-ids i-123

# Wait for an instance to terminate
aws ec2 wait instance-terminated --instance-ids i-123

# Useful in scripts — blocks until state is reached
```

---

### GUI Equivalent
**AWS Management Console** — every CLI command has a Console equivalent. Use the Console to explore, CLI to automate.

### Next: [05 — S3 Essentials](./05-s3-essentials.md)
