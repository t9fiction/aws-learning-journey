# Day 6 — Security Groups

## Concept

A Security Group (SG) is a **stateful virtual firewall** at the **ENI level** (instance level). It controls inbound and outbound traffic for one or more instances.

This is the **primary firewall** for network admins working with EC2, RDS, ELB, Lambda, and other services.

## Key Characteristics

| Feature | Behavior |
|---------|----------|
| **Stateful** | If you allow inbound, the return traffic is automatically allowed (no need to specify outbound) |
| **Default deny** | All inbound is denied unless explicitly allowed |
| **Allow only** | You can only add ALLOW rules — no DENY rules |
| **ENI-level** | Applied to elastic network interfaces, not the instance itself |
| **AZ scope** | Can reference SGs in the same VPC only |
| **Max SGs per ENI** | 5 (soft limit) |
| **Max rules per SG** | 60 inbound + 60 outbound |
| **Evaluation** | All rules are evaluated (OR logic) — any matching rule allows traffic |

## Security Group vs Network ACL

| Feature | Security Group | Network ACL |
|---------|---------------|-------------|
| Level | Instance (ENI) | Subnet |
| Stateful | Yes | No (return traffic must be explicitly allowed) |
| Rules | Allow only | Allow + Deny |
| Evaluation order | All rules evaluated | Rule numbers (lowest first) |
| Return traffic | Auto-allowed | Must be explicit |
| Applies to | Instances with SG attached | All instances in subnet |

## Default Security Group

Every VPC comes with a **default SG**. By default:
- Allows **all outbound** traffic
- Allows **all inbound** from other instances in the same SG
- Denies **all other inbound**

**Delete the default SG** in production — create your own.

## Creating Rules

### Inbound Rules (what comes in)

```bash
# SSH access from your office
aws ec2 authorize-security-group-ingress \
  --group-id sg-12345678 \
  --protocol tcp \
  --port 22 \
  --cidr 203.0.113.0/24

# HTTP/HTTPS from anywhere
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

# MySQL from an app SG (referencing another SG)
aws ec2 authorize-security-group-ingress \
  --group-id sg-database \
  --protocol tcp \
  --port 3306 \
  --source-group sg-app-server
```

### Outbound Rules (what goes out)

```bash
# Allow outbound HTTPS to internet
aws ec2 authorize-security-group-egress \
  --group-id sg-12345678 \
  --protocol tcp \
  --port 443 \
  --cidr 0.0.0.0/0

# Restrict outbound to specific S3 prefix list (S3 Gateway Endpoint)
aws ec2 authorize-security-group-egress \
  --group-id sg-12345678 \
  --protocol tcp \
  --port 443 \
  --prefix-list pl-6da54004  # S3 prefix for us-east-1
```

### All Protocols

```bash
# ICMP (ping)
aws ec2 authorize-security-group-ingress \
  --group-id sg-12345678 \
  --protocol icmp \
  --port -1 \
  --cidr 10.0.0.0/16

# All traffic
aws ec2 authorize-security-group-ingress \
  --group-id sg-12345678 \
  --protocol -1 \
  --cidr 10.0.0.0/16

# UDP
aws ec2 authorize-security-group-ingress \
  --group-id sg-12345678 \
  --protocol udp \
  --port 53 \
  --cidr 10.0.0.0/16  # DNS
```

## Common SG Patterns for Network Admins

### Web Application

```
Web Tier SG (sg-web)
  Inbound:  80, 443 from 0.0.0.0/0
  Outbound: All to sg-app

App Tier SG (sg-app)
  Inbound:  Custom TCP from sg-web
  Outbound: 3306 to sg-db, 443 to 0.0.0.0/0

DB Tier SG (sg-db)
  Inbound:  3306 from sg-app
  Outbound: None needed
```

### Bastion / Jump Box

```
Bastion SG
  Inbound:  22 from your-office-IP/32
  Outbound: 22 to internal instances

Internal SG
  Inbound:  22 from sg-bastion
```

### VPC-Only Service

```
Internal SG
  Inbound:  Ports from internal CIDR only
  Outbound: None or restricted
```

## Describe & List Rules

```bash
# Describe a security group
aws ec2 describe-security-groups --group-ids sg-12345678

# List all SGs in VPC
aws ec2 describe-security-groups \
  --filters Name=vpc-id,Values=vpc-12345678

# List all inbound rules (JSON)
aws ec2 describe-security-groups \
  --group-ids sg-12345678 \
  --query 'SecurityGroups[0].IpPermissions'

# List SGs with no rules (unused)
aws ec2 describe-security-groups \
  --query 'SecurityGroups[?!length(IpPermissions)]'
```

## Delete Security Group

```bash
aws ec2 delete-security-group --group-id sg-12345678
```

**Cannot delete if**: any ENI is associated with it.

## CLI Commands Summary

```bash
aws ec2 create-security-group
aws ec2 delete-security-group
aws ec2 describe-security-groups
aws ec2 authorize-security-group-ingress
aws ec2 authorize-security-group-egress
aws ec2 revoke-security-group-ingress
aws ec2 revoke-security-group-egress
```

## Console

**EC2 → Security Groups** — full management
VPC Console also shows SGs under Security

## Gotchas

- **SG references work only in the same VPC** (or peered VPC if configured)
- **SGs are stateful** — don't need to add return traffic rules
- You can attach **multiple SGs** to one ENI — all rules combined
- Changing SGs takes effect **immediately**
- SGs are **evaluated before NACLs** for inbound (SG at ENI, NACL at subnet)
- **Cannot block traffic explicitly** — if you need DENY, use NACL
- **Default outbound allows all** — lock it down in production
- Rule limits: 60 inbound + 60 outbound per SG
- SGs are **VPC-specific** — can't use the same SG across VPCs
- **CIDR-based rules** use the ENI's private IP, not public IP
