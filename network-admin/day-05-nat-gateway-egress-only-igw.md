# Day 5 — NAT Gateway, NAT Instance & Egress-Only Internet Gateway

## Problem

Instances in **private subnets** need internet access for updates, patches, and external API calls — but should not be directly reachable from the internet.

## Solutions

| Solution | IPv4 | IPv6 | Managed | HA | Cost |
|----------|------|------|---------|----|------|
| NAT Gateway | Yes | No | Yes | Yes (per AZ) | ~$32/month + data |
| NAT Instance | Yes | No | No | No (single EC2) | EC2 cost |
| Egress-Only IGW | No | Yes | Yes | Yes | Free (data only) |
| VPC Gateway Endpoint | S3/DDB only | — | Yes | Yes | Free |
| VPC Interface Endpoint | Yes | No | Yes | Yes | Per hour + data |

## NAT Gateway

A **managed** service that allows private subnet instances to connect to the internet (or other AWS services) **but prevents inbound connections**.

### Architecture

```
Private Instance (10.0.2.10)
  └─ Route: 0.0.0.0/0 → nat-xxx (in public subnet)
       └─ NAT Gateway translates source IP to its Elastic IP
            └─ IGW sends to internet
                 └─ Response comes back to NAT → translated back to private IP
```

### NAT Gateway in a public subnet:
```
                                  ┌─────────────────┐
                                  │  Public Subnet   │
                                  │  NAT Gateway     │◄── Elastic IP
                                  │  10.0.1.100      │
                                  └────────┬────────┘
                                           │
      ┌──────────────────┐        ┌───────┴────────┐
      │  Private Subnet   │        │  IGW            │
      │  EC2 (10.0.2.50)  │───────►│  Route: 0.0.0.0  │
      │  Route: 0.0.0.0   │        │  → nat-xxx      │
      │  → nat-xxx        │        └────────────────┘
      └──────────────────┘
```

### Create NAT Gateway

```bash
# Allocate Elastic IP first
ALLOC_ID=$(aws ec2 allocate-address \
  --domain vpc \
  --query 'AllocationId' --output text)

# Create NAT Gateway in a public subnet
NAT_ID=$(aws ec2 create-nat-gateway \
  --subnet-id subnet-public-1a \
  --allocation-id $ALLOC_ID \
  --query 'NatGateway.NatGatewayId' --output text)

# Add route in private subnet route table
aws ec2 create-route \
  --route-table-id rtb-private \
  --destination-cidr-block 0.0.0.0/0 \
  --nat-gateway-id $NAT_ID
```

### Key Facts
- **Runs in a public subnet** with an Elastic IP
- **AZ-scoped** — if the AZ fails, the NAT goes down
- For high availability: create one NAT Gateway **per AZ**
- Handles up to **45 Gbps** can burst up to 100 Gbps
- Automatically handles **5 Gbps** baseline, can burst
- No security group — controlled by NACLs
- Supports TCP, UDP, ICMP
- Does **not** support IPSec, port forwarding, or being a bastion

### Cost
- **Hourly charge**: ~$0.045/hour ($32/month)
- **Data processing**: $0.045/GB
- **Data transfer**: standard egress rates

## NAT Instance (Legacy)

An **EC2 instance** you configure as a NAT. Cheaper but you manage it.

```bash
# Launch Amazon Linux 2 NAT AMI
aws ec2 run-instances \
  --image-id ami-0abcdef1234567890 \
  --instance-type t3.nano \
  --subnet-id subnet-public \
  --associate-public-ip-address

# Then on the instance, disable source/destination check
aws ec2 modify-instance-attribute \
  --instance-id i-123 \
  --source-dest-check "Value=false"
```

### Drawbacks (vs NAT Gateway)
- Single point of failure
- Manual scaling (bigger instance type)
- Manual patching and maintenance
- You manage the iptables/NAT config
- Only use for cost-saving in dev/test

## Egress-Only Internet Gateway

For **IPv6 traffic** — allows outbound-only from private subnets.

### How it works
- IPv6 addresses are globally unique (no NAT needed)
- Egress-only IGW uses **routing** — traffic from private subnet to IPv6 internet goes through it
- Inbound traffic is **impossible** because there's no return route

```bash
# Create egress-only IGW
aws ec2 create-egress-only-internet-gateway --vpc-id vpc-12345678

# Add route
aws ec2 create-route \
  --route-table-id rtb-private \
  --destination-ipv6-cidr-block ::/0 \
  --egress-only-internet-gateway-id eigw-12345678
```

## Comparison Table

| Feature | NAT Gateway | NAT Instance | Egress-Only IGW |
|---------|------------|-------------|-----------------|
| Managed by AWS | Yes | No | Yes |
| Protocol | IPv4 | IPv4 | IPv6 |
| HA built-in | Per AZ | No | Yes |
| Bandwidth | 45 Gbps | Instance type | 45 Gbps |
| Security Group | No | Yes | No |
| Maintenance | None | You patch | None |
| Cost | Higher ($32/mo + data) | EC2 cost only | Free (data only) |
| Port forwarding | No | Yes (iptables) | No |
| Bastion host | No | Possible | No |

## CLI Commands

```bash
# NAT Gateway
aws ec2 create-nat-gateway
aws ec2 describe-nat-gateways
aws ec2 delete-nat-gateway

# Elastic IPs
aws ec2 allocate-address
aws ec2 release-address
aws ec2 describe-addresses

# Egress-only IGW
aws ec2 create-egress-only-internet-gateway
aws ec2 delete-egress-only-internet-gateway
```

## Console

**VPC → NAT Gateways** — create, view, delete
**VPC → Elastic IPs** — allocate/release
**VPC → Route Tables** — modify routes to point to NAT

## Gotchas

- **NAT Gateway has no inbound ports** — you cannot SSH through it
- **Both NAT and IGW in same VPC** — public subnet routes 0.0.0.0/0 to IGW, private routes to NAT
- **One NAT per AZ** for HA — costs more but survives AZ failure
- NAT Gateway data processing charge can **exceed** the hourly cost for high-traffic apps
- **NAT instances require disabling Source/Dest Check** — otherwise traffic is silently dropped
- Egress-only IGW only works with **IPv6** — no dual-stack for egress-only
