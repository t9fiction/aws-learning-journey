# 06 — EC2 Essentials

## Key Commands

### List instances
```bash
# All instances
aws ec2 describe-instances

# Only running instances
aws ec2 describe-instances --filters Name=instance-state-name,Values=running

# With specific fields (query)
aws ec2 describe-instances \
  --query 'Reservations[*].Instances[*].{ID:InstanceId,Type:InstanceType,State:State.Name,IP:PublicIpAddress}' \
  --output table

# Filter by tag
aws ec2 describe-instances \
  --filters Name=tag:Name,Values=web-server
```

### Launch an instance
```bash
aws ec2 run-instances \
  --image-id ami-0abcdef1234567890 \
  --instance-type t2.micro \
  --key-name MyKeyPair \
  --security-group-ids sg-12345678 \
  --subnet-id subnet-12345678 \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=MyServer}]'
```

### Stop/Start/Terminate
```bash
aws ec2 stop-instances --instance-ids i-1234567890abcdef0
aws ec2 start-instances --instance-ids i-1234567890abcdef0
aws ec2 terminate-instances --instance-ids i-1234567890abcdef0
```

### Security Groups
```bash
# Create
aws ec2 create-security-group \
  --group-name web-sg \
  --description "Web server security group"

# Add ingress rules (allow HTTP/HTTPS)
aws ec2 authorize-security-group-ingress \
  --group-id sg-12345678 \
  --protocol tcp \
  --port 80 \
  --cidr 0.0.0.0/0

aws ec2 authorize-security-group-ingress \
  --group-id sg-12345678 \
  --protocol tcp \
  --port 443 \
  --cidr 0.0.0.0/0

# Allow SSH from your IP
aws ec2 authorize-security-group-ingress \
  --group-id sg-12345678 \
  --protocol tcp \
  --port 22 \
  --cidr YOUR_IP/32

# Revoke a rule
aws ec2 revoke-security-group-ingress \
  --group-id sg-12345678 \
  --protocol tcp \
  --port 22 \
  --cidr 0.0.0.0/0
```

### Key Pairs
```bash
# Create a key pair (downloads the .pem)
aws ec2 create-key-pair --key-name MyKey --query 'KeyMaterial' --output text > MyKey.pem
chmod 400 MyKey.pem

# List key pairs
aws ec2 describe-key-pairs
```

### Elastic IP
```bash
# Allocate
aws ec2 allocate-address

# Associate to an instance
aws ec2 associate-address --instance-id i-123 --allocation-id eipalloc-123

# Release
aws ec2 release-address --allocation-id eipalloc-123
```

### Volumes & Snapshots
```bash
# List volumes
aws ec2 describe-volumes

# Create snapshot
aws ec2 create-snapshot --volume-id vol-123 --description "Backup before update"

# List snapshots
aws ec2 describe-snapshots --owner-ids self

# Delete snapshot
aws ec2 delete-snapshot --snapshot-id snap-123
```

### AMIs
```bash
# List your AMIs
aws ec2 describe-images --owners self

# Create AMI from instance
aws ec2 create-image --instance-id i-123 --name "MyServer-AMI-2024"

# Deregister AMI
aws ec2 deregister-image --image-id ami-123
```

### Instance Metadata (from within an EC2 instance)
```bash
# Fetch instance ID from the instance itself
TOKEN=$(curl -s -X PUT "http://169.254.169.254/latest/api/token" \
  -H "X-aws-ec2-metadata-token-ttl-seconds: 21600")
curl -s -H "X-aws-ec2-metadata-token: $TOKEN" \
  http://169.254.169.254/latest/meta-data/instance-id
```

## Common One-Liners

```bash
# Get public IP of an instance
aws ec2 describe-instances --instance-ids i-123 \
  --query 'Reservations[0].Instances[0].PublicIpAddress' --output text

# Get all instance IDs
aws ec2 describe-instances --query 'Reservations[*].Instances[*].InstanceId' --output text

# Stop all running instances
aws ec2 stop-instances --instance-ids \
  $(aws ec2 describe-instances --filters Name=instance-state-name,Values=running \
    --query 'Reservations[*].Instances[*].InstanceId' --output text)

# Tag an instance
aws ec2 create-tags --resources i-123 --tags Key=Environment,Value=Production

# Get console output (troubleshooting boot issues)
aws ec2 get-console-output --instance-id i-123
```

---

### GUI Equivalent
**EC2 Console** → Instances → Launch / Stop / Start / Terminate.  
**Security Groups** under Network & Security.  
**Elastic IPs** under Network & Security.  
**Snapshots** under Elastic Block Store.

### Next: [07 — IAM, Lambda, DynamoDB](./07-iam-lambda-dynamodb.md)
