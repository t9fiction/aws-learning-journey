# Day 10 — VPC Endpoints

## Concept

A VPC Endpoint lets you privately connect your VPC to **supported AWS services** without going through the internet, NAT, VPN, or Direct Connect. Traffic stays **entirely within the AWS network**.

## Two Types

| Type | What It Connects To | Access Method |
|------|--------------------|--------------|
| **Gateway Endpoint** | S3, DynamoDB | Route table entry |
| **Interface Endpoint** (PrivateLink) | 150+ AWS services + 3rd party | ENI with private IP |

## Gateway Endpoint

### How It Works

```
VPC
  ┌──────────────┐
  │ EC2 Instance  │──── (route: AWS service prefix → vpce-xxx)
  │ (private IP)  │                  │
  └──────────────┘                  │
       ┌────────────────────────────┘
       ▼
  Gateway Endpoint
       │
       ▼
  S3 or DynamoDB
```

**Simple** — add an entry in your route table pointing to the endpoint.

### Create Gateway Endpoint for S3

```bash
# Find the S3 prefix list ID for your region
aws ec2 describe-prefix-lists \
  --query 'PrefixLists[?PrefixListName==`com.amazonaws.us-east-1.s3`].PrefixListId'

# Create Gateway Endpoint
aws ec2 create-vpc-endpoint \
  --vpc-id vpc-12345678 \
  --service-name com.amazonaws.us-east-1.s3 \
  --route-table-ids rtb-public rtb-private
```

### Critical: Route Table Update

The `--route-table-ids` parameter automatically adds a route:

```
Destination: pl-6da54004 (S3 prefix list)
Target: vpce-xxxxxxxx
```

Instances in these subnets now reach S3 through the endpoint instead of IGW/NAT.

### Gateway Endpoint Policy

Control access with a custom policy:

```bash
aws ec2 create-vpc-endpoint \
  --vpc-id vpc-12345678 \
  --service-name com.amazonaws.us-east-1.s3 \
  --route-table-ids rtb-private \
  --policy-document file://endpoint-policy.json
```

Example policy — allow only specific bucket:

```json
{
  "Version": "2008-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": "*",
      "Action": ["s3:GetObject", "s3:PutObject"],
      "Resource": "arn:aws:s3:::my-company-bucket/*"
    }
  ]
}
```

## Interface Endpoint (PrivateLink)

### How It Works

```
VPC
  ┌──────────────┐
  │ EC2 Instance  │────► ENI (10.0.1.50) ───► AWS Service
  └──────────────┘     (private IP)
                       │
                   PrivateLink
                       │
                       ▼
                  SQS / SNS / KMS / EC2 API / etc.
```

### Create Interface Endpoint

```bash
# Create Endpoint for SQS
aws ec2 create-vpc-endpoint \
  --vpc-id vpc-12345678 \
  --vpc-endpoint-type Interface \
  --service-name com.amazonaws.us-east-1.sqs \
  --subnet-ids subnet-1a subnet-1b \
  --security-group-ids sg-endpoint
```

### Key Facts
- Creates an **ENI** in each specified subnet with a private IP
- Uses **Security Groups** for access control
- You pay per hour ($0.01/hour per AZ) + data processing ($0.01/GB)
- Supports **DNS resolution** (service-specific DNS names resolve to the private IPs)

### DNS Names

When you create an Interface Endpoint, AWS provides DNS names:

- **Regional**: `vpce-xxx.sqs.us-east-1.vpce.amazonaws.com` → private IP in any AZ
- **Zonal**: `vpce-xxx-us-east-1a.sqs.us-east-1.vpce.amazonaws.com` → private IP in specific AZ
- **Private DNS**: If enabled, the default service DNS (e.g., `sqs.us-east-1.amazonaws.com`) resolves to the private IPs instead of public IPs

```bash
# Enable private DNS
aws ec2 modify-vpc-endpoint \
  --vpc-endpoint-id vpce-xxx \
  --private-dns-enabled
```

## Gateway vs Interface: Comparison

| Feature | Gateway Endpoint | Interface Endpoint |
|---------|-----------------|-------------------|
| Services | S3, DynamoDB only | 150+ AWS services |
| Cost | **Free** | $0.01/hr per AZ + data |
| Access control | Policy + route table | Security Group |
| Implementation | Route table prefix | ENI in subnet |
| On-prem access | No (VPC only) | Yes (via VPN/DX) |
| Cross-region | No | No |
| Bandwidth | No limit | 10 Gbps per ENI |

## On-Premises Access via Interface Endpoint

Interface Endpoints can be reached from on-premises via VPN or Direct Connect because they have **private IPs**.

```
On-prem ─── VPN ─── VPC ─── Interface Endpoint ─── AWS Service
```

## CLI Commands

```bash
# Create
aws ec2 create-vpc-endpoint                    # Gateway type (default)
aws ec2 create-vpc-endpoint --vpc-endpoint-type Interface

# Describe
aws ec2 describe-vpc-endpoints
aws ec2 describe-vpc-endpoint-services          # List available services

# Modify
aws ec2 modify-vpc-endpoint                     # Add/remove subnets, SGs, policy
aws ec2 modify-vpc-endpoint --private-dns-enabled

# Delete
aws ec2 delete-vpc-endpoints
```

## Console

**VPC → Endpoints** — create and manage all endpoints
**VPC → Endpoint Services** — if you want to offer your own service via PrivateLink

## Gotchas

- **Gateway Endpoints cost nothing** — always use them for S3/DynamoDB
- **Interface Endpoints cost money** — use only when needed
- **Gateway Endpoints are regional** — won't work cross-region
- **Interface Endpoints are AZ-scoped** — create in each AZ you use
- **Bucket policies**: When using Gateway Endpoint, S3 bucket policies can restrict access to the endpoint (use `aws:SourceVpce` condition)
- **Private DNS** and on-prem: If you enable private DNS, on-prem DNS queries to `s3.amazonaws.com` may break
- **Prefix lists**: the route for Gateway Endpoints uses the prefix list as destination
