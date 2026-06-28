# Day 8 — EC2 & Compute Fundamentals

## Concept

Amazon EC2 provides **virtual servers** in the cloud. As a network admin, you need to understand how instances connect to the network, how instance types affect networking performance, and how to manage access.

## Instance Lifecycle

```
Stop ──► Start ──► Running ──► Stop ──► Terminate
  │         │          │
  └─────────┴──────────┘
        Reboot
```

| State | Network State | Charged |
|-------|--------------|---------|
| Running | ENIs attached, traffic flows | Yes |
| Stopped | ENIs detached | EBS only |
| Terminated | ENIs deleted | No |
| Hibernated | ENIs detached | EBS only |

## Instance Types (Networking-Relevant)

| Family | Use Case | Network Performance |
|--------|----------|-------------------|
| **T3/T3a** | Burstable (general) | Up to 5 Gbps |
| **M5/M5a/M6i** | General purpose | Up to 25 Gbps |
| **C5/C6i** | Compute optimized | Up to 100 Gbps |
| **R5/R6i** | Memory optimized | Up to 50 Gbps |
| **I3/I4i** | Storage optimized | Up to 50 Gbps |
| **P3/P4** | GPU/ML (excluded per your request) | Up to 400 Gbps |
| **G4/G5** | Graphics | Up to 25 Gbps |

### Network Performance Indicators

- **Baseline bandwidth**: guaranteed minimum
- **Burst bandwidth**: available when needed (credit-based for T3)
- **PPS**: packets per second capacity
- **Connections**: concurrent connection limit
- **ENA vs ENA Express**: newer drivers = better performance

### ENA (Elastic Network Adapter)

All modern instances use **ENA** (instead of the older Intel 82599 VF). ENA supports:

- **Up to 100 Gbps** per ENI
- **SR-IOV** (single root I/O virtualization) — direct hardware access
- **Multiple TX/RX queues** — better parallelism

### ENA Express

For workloads needing **ultra-low latency** (< 10µs):

```bash
# Enable on an ENI
aws ec2 modify-network-interface-attribute \
  --network-interface-id eni-xxx \
  --ena-srd-specification '{
    "EnaSrdEnabled": true
  }'
```

Requires both instances to support ENA Express and same placement group.

## Placement Groups

Control how instances are placed on underlying hardware.

| Type | What It Does | Use Case | Network Impact |
|------|-------------|----------|---------------|
| **Cluster** | Instances in same AZ, same rack | Low-latency (HPC) | Highest bandwidth, lowest latency |
| **Spread** | Instances on different hardware | High availability | Standard |
| **Partition** | Instances in groups, isolated from other groups | Large distributed systems | Standard |

```bash
# Create cluster placement group
aws ec2 create-placement-group \
  --group-name hpc-cluster \
  --strategy cluster

# Launch instance in placement group
aws ec2 run-instances \
  --placement GroupName=hpc-cluster \
  --instance-type c5n.18xlarge \
  --subnet-id subnet-1a
```

## AMIs (Amazon Machine Images)

A template for launching instances — contains OS, software, configuration.

### AMI Sources
- **Amazon provided**: Amazon Linux 2, Amazon Linux 2023, Ubuntu, Windows
- **AWS Marketplace**: third-party (Cisco, Palo Alto, etc.)
- **Community**: shared by others
- **Custom**: your own (created from an instance)

```bash
# Find an AMI
aws ec2 describe-images \
  --owners amazon \
  --filters "Name=name,Values=al2023-ami-*-kernel-6.1-x86_64" \
  --query 'Images[*].[ImageId,Name,CreationDate]' --output text

# Create an AMI from an instance
aws ec2 create-image \
  --instance-id i-12345678 \
  --name "web-server-v1" \
  --no-reboot
```

## Key Pairs

Used for SSH access (Linux) or password decryption (Windows).

```bash
# Create key pair
aws ec2 create-key-pair \
  --key-name my-key \
  --query 'KeyMaterial' --output text > my-key.pem
chmod 400 my-key.pem

# Import existing public key
aws ec2 import-key-pair \
  --key-name my-key \
  --public-key-material fileb://~/.ssh/id_rsa.pub

# SSH into instance
ssh -i my-key.pem ec2-user@<public-ip>
```

## User Data & Instance Metadata

### User Data (bootstrapping scripts)

```bash
# Pass a script that runs at first boot
aws ec2 run-instances \
  --image-id ami-xxx \
  --instance-type t3.micro \
  --user-data file://bootstrap.sh
```

Example `bootstrap.sh`:
```bash
#!/bin/bash
yum update -y
yum install -y httpd
systemctl start httpd
echo "<h1>Hello from $(hostname -f)</h1>" > /var/www/html/index.html
```

### Instance Metadata Service (IMDS)

Metadata is available from within the instance at `http://169.254.169.254/latest/meta-data/`. Two versions:

- **IMDSv1**: request-response (no token)
- **IMDSv2**: session-oriented (token required) — **recommended, more secure**

```bash
# IMDSv2 (secure)
TOKEN=$(curl -s -X PUT "http://169.254.169.254/latest/api/token" \
  -H "X-aws-ec2-metadata-token-ttl-seconds: 21600")

curl -s -H "X-aws-ec2-metadata-token: $TOKEN" \
  http://169.254.169.254/latest/meta-data/instance-id

curl -s -H "X-aws-ec2-metadata-token: $TOKEN" \
  http://169.254.169.254/latest/meta-data/public-ipv4

curl -s -H "X-aws-ec2-metadata-token: $TOKEN" \
  http://169.254.169.254/latest/meta-data/network/interfaces/macs/
```

### Useful Metadata Keys

| Path | Value |
|------|-------|
| `instance-id` | i-1234567890abcdef0 |
| `public-ipv4` | Public IP assigned to instance |
| `local-ipv4` | Private IP |
| `mac` | MAC address of the ENI |
| `network/interfaces/macs/` | List of MACs |
| `iam/security-credentials/role-name` | Temporary AWS credentials |
| `placement/availability-zone` | AZ where instance runs |
| `tags/instance/Name` | Instance name tag (IMDSv2 only) |

## Instance Purchasing Options

| Option | Commitment | Discount | Network Effect |
|--------|-----------|----------|---------------|
| **On-Demand** | None | None | Standard |
| **Reserved (1/3 year)** | Term commitment | Up to 72% | Standard |
| **Savings Plans** | $/hr commitment | Up to 72% | Standard |
| **Spot** | None (can be interrupted) | Up to 90% | Standard |
| **Dedicated Host** | Physical server | Varies | Standard |
| **Capacity Reservations** | Region/AZ capacity | None | Standard |

## SSH Key Management

```bash
# List key pairs
aws ec2 describe-key-pairs

# Get fingerprint
aws ec2 describe-key-pairs --key-names my-key \
  --query 'KeyPairs[0].KeyFingerprint'

# Delete key pair
aws ec2 delete-key-pair --key-name my-key
```

## Systems Manager Session Manager

Connect to instances **without SSH** — no open ports, no bastion hosts, no SSH keys:

```bash
# Prerequisites: SSM Agent + IAM role on instance
# SSM Agent is pre-installed on Amazon Linux 2/2023

# Start session
aws ssm start-session --target i-12345678

# Port forwarding (tunnel)
aws ssm start-session \
  --target i-12345678 \
  --document-name AWS-StartPortForwardingSession \
  --parameters '{"portNumber":["80"], "localPortNumber":["8080"]}'

# Now access localhost:8080 to reach the instance's port 80
```

**Benefits for network admins:**
- No SSH port (22) open in security groups
- No bastion host needed
- All connections logged in CloudTrail
- IAM-based access control
- Works across private subnets (no NAT needed)

### Run Command (Remote Execution)

Execute scripts/commands on multiple instances without SSH:

```bash
# Run a shell command on all web servers tagged with Environment=Production
aws ssm send-command \
  --document-name "AWS-RunShellScript" \
  --targets "Key=tag:Environment,Values=Production" \
  --parameters 'commands=["yum update -y"]' \
  --comment "Security patch all production instances"
```

### Parameter Store (Centralized Config)

Store network configuration securely (VPN PSKs, API keys, config data):

```bash
# Store a parameter
aws ssm put-parameter \
  --name "/network/vpn/shared-secret" \
  --value "supersecret123" \
  --type SecureString \
  --key-id alias/aws/ssm

# Retrieve (from instance)
aws ssm get-parameter \
  --name "/network/vpn/shared-secret" \
  --with-decryption

# Hierarchy example
/network/dns/primary     → "10.0.0.10"
/network/dns/secondary   → "10.0.0.11"
/network/ntp/server      → "169.254.169.123"
/network/vpn/psk         → (encrypted)
```

### SSM Agent

- Pre-installed on Amazon Linux 2/2023, Ubuntu AMIs
- Requires IAM role: `AmazonSSMManagedInstanceCore`
- Outbound HTTPS (port 443) to SSM endpoints — no inbound ports needed
- Can be installed on on-premises servers (hybrid activation)

### SSM Architecture Impact

Session Manager changes the network design dramatically:

```
Before SSM:
  Internet ──► Bastion (port 22) ──► Private instances (port 22)
  └── Open SSH port, manage bastion, keys to distribute

After SSM:
  Internet ──► SSM API (443) ──► SSM Agent (no open ports)
  └── No bastion, no SSH keys, IAM-based, CloudTrail-logged
```

## CLI Commands Summary

```bash
aws ec2 run-instances
aws ec2 describe-instances
aws ec2 stop-instances
aws ec2 start-instances
aws ec2 terminate-instances
aws ec2 reboot-instances
aws ec2 create-key-pair
aws ec2 describe-key-pairs
aws ec2 create-image
aws ec2 describe-images
aws ec2 create-placement-group
aws ec2 modify-instance-attribute

# Session Manager
aws ssm start-session
aws ssm send-command
```

## Console

**EC2 → Instances** — launch, view, manage
**EC2 → AMIs** — create and manage AMIs
**EC2 → Key Pairs** — create and manage
**EC2 → Placement Groups** — create and manage
**Systems Manager → Session Manager** — start sessions

## Gotchas

- **Public IPs change on stop/start** (unless using Elastic IP)
- **Default vCPU limit**: 5 per region (can be increased)
- **Instance metadata**: IMDSv2 is more secure — enable `MetadataOptions HttpTokens=required`
- **Source/Dest Check**: enabled by default — disable if instance acts as NAT/appliance
- **ENA driver**: must be installed for > 10 Gbps performance
- **Termination protection**: enable to prevent accidental deletion: `aws ec2 modify-instance-attribute --instance-id i-xxx --disable-api-termination`
- **Spot instances**: can be terminated with 2-minute notice — design accordingly
- **Hibernation**: only supported for specific instance types; RAM is saved to EBS
- **SYN/ACK behavior**: ensure security groups allow ephemeral ports for return traffic
