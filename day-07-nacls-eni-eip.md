# Day 7 — Network ACLs, Elastic Network Interfaces & Elastic IPs

## Network ACLs (NACLs)

A **stateless** firewall at the **subnet level**. All instances in the subnet are governed by the same NACL.

### Stateless Explained

With stateful firewalls (Security Groups), if you allow inbound SYN, the return SYN-ACK is automatically allowed. With **NACLs**, you must explicitly allow both directions.

### Default NACL

Every VPC has a **default NACL** that allows **all inbound and outbound** traffic. Custom NACLs start with **all deny**.

### Rule Processing

- Rules are evaluated **by number** (lowest first)
- First match wins
- Each rule has: **Rule# → Type → Protocol → Port Range → Source/Dest → Allow/Deny**

### Example NACL: Web Subnet

**Inbound Rules:**

| Rule# | Type | Protocol | Port | Source | Action |
|-------|------|----------|------|--------|--------|
| 100 | HTTP | TCP | 80 | 0.0.0.0/0 | ALLOW |
| 110 | HTTPS | TCP | 443 | 0.0.0.0/0 | ALLOW |
| 120 | SSH | TCP | 22 | 203.0.113.0/24 | ALLOW |
| 130 | Ephemeral | TCP | 1024-65535 | 0.0.0.0/0 | ALLOW |
| * | All | All | All | 0.0.0.0/0 | DENY |

**Outbound Rules:**

| Rule# | Type | Protocol | Port | Dest | Action |
|-------|------|----------|------|------|--------|
| 100 | HTTP | TCP | 80 | 0.0.0.0/0 | ALLOW |
| 110 | HTTPS | TCP | 443 | 0.0.0.0/0 | ALLOW |
| 120 | Ephemeral | TCP | 1024-65535 | 0.0.0.0/0 | ALLOW |
| * | All | All | All | 0.0.0.0/0 | DENY |

**Note**: Ephemeral ports (1024-65535) are needed for return traffic from connections initiated by your instances.

### CLI Commands

```bash
# Create NACL
aws ec2 create-network-acl --vpc-id vpc-12345678

# Add inbound rule
aws ec2 create-network-acl-entry \
  --network-acl-id acl-12345678 \
  --rule-number 100 \
  --protocol tcp \
  --port-range From=80,To=80 \
  --cidr-block 0.0.0.0/0 \
  --rule-action allow \
  --ingress

# Add outbound ephemeral rule (for return traffic)
aws ec2 create-network-acl-entry \
  --network-acl-id acl-12345678 \
  --rule-number 100 \
  --protocol tcp \
  --port-range From=1024,To=65535 \
  --cidr-block 0.0.0.0/0 \
  --rule-action allow \
  --egress

# Associate NACL with subnet
aws ec2 associate-network-acl \
  --network-acl-id acl-12345678 \
  --subnet-id subnet-12345678

# Describe NACLs
aws ec2 describe-network-acls

# Delete NACL entry
aws ec2 delete-network-acl-entry \
  --network-acl-id acl-12345678 \
  --rule-number 100 --ingress
```

### Sizing Ephemeral Port Ranges

| Client OS | Ephemeral Range |
|-----------|----------------|
| Linux | 32768-60999 |
| Windows (modern) | 49152-65535 |
| NAT Gateway | 1024-65535 |
| ALB/NLB | 1024-65535 |

**Safest NACL ephemeral rule**: `1024-65535`

## Elastic Network Interfaces (ENI)

An ENI is a **virtual network card** attached to an EC2 instance.

### Core Attributes

| Attribute | Description |
|-----------|-------------|
| **Private IP** | Primary (always set) + secondary IPs |
| **Public IP** | Optional, assigned at launch or via EIP |
| **Elastic IP** | Static public IP (see below) |
| **Security Groups** | Attached to ENI, not instance |
| **Source/Dest Check** | Default ON — must disable if ENI acts as NAT |
| **MAC Address** | Unique, persistent |
| **Description** | Free-form text |

### ENI Use Cases

- **Management network** — separate ENI for admin SSH traffic
- **Dual-homed instances** — ENI in public subnet + ENI in private subnet
- **NAT/Appliance** — disable Source/Dest Check
- **Failover** — move ENI to another instance (faster than DNS)
- **Logging/Monitoring** — dedicated ENI for monitoring traffic

### CLI Commands

```bash
# Create ENI
aws ec2 create-network-interface \
  --subnet-id subnet-12345678 \
  --description "Management ENI"

# Attach ENI to instance
aws ec2 attach-network-interface \
  --network-interface-id eni-12345678 \
  --instance-id i-12345678 \
  --device-index 1

# Assign a secondary private IP
aws ec2 assign-private-ip-addresses \
  --network-interface-id eni-12345678 \
  --secondary-private-ip-addresses 10.0.1.50

# Describe ENIs
aws ec2 describe-network-interfaces

# Modify attribute (disable source/dest check)
aws ec2 modify-network-interface-attribute \
  --network-interface-id eni-12345678 \
  --source-dest-check "Value=false"

# Detach ENI
aws ec2 detach-network-interface --attachment-id eni-attach-123

# Delete ENI
aws ec2 delete-network-interface --network-interface-id eni-12345678
```

### Device Index

- **eth0 (device-index 0)**: Primary ENI — always attached, cannot be detached
- **eth1 (device-index 1)**: Secondary ENI (optional)
- Additional ENIs use device-index 2, 3, 4...

## Elastic IP (EIP)

A **static public IPv4 address** you can allocate and assign to instances or NAT gateways.

### Key Facts
- **5 EIPs per region** (soft limit — can be increased)
- **Cost**: ~$0.005/hour when **not associated** with a running instance (to prevent IP squatting)
- **Free** when associated with a running instance
- **Region-scoped** — cannot move between regions
- You can move EIPs between instances and ENIs

### CLI Commands

```bash
# Allocate
aws ec2 allocate-address --domain vpc

# Associate with instance
aws ec2 associate-address \
  --instance-id i-12345678 \
  --allocation-id eipalloc-12345678

# Associate with ENI
aws ec2 associate-address \
  --network-interface-id eni-12345678 \
  --allocation-id eipalloc-12345678

# Disassociate
aws ec2 disassociate-address --association-id eipassoc-12345678

# Release
aws ec2 release-address --allocation-id eipalloc-12345678

# List all EIPs
aws ec2 describe-addresses

# Check if EIP is associated
aws ec2 describe-addresses --allocation-ids eipalloc-12345678 \
  --query 'Addresses[0].AssociationId'
```

## Putting It All Together

### Typical Three-Tier Architecture

```
Internet
  │
  ▼
IGW
  │
  ▼
ALB (public subnet, sg-web)
  │
  ▼
Web/App servers (private subnet, sg-app)
  ├── Auto-scaling group
  ├── Multiple AZs
  └── Outbound via NAT Gateway
  │
  ▼
RDS Database (private subnet, sg-db)
  ├── Multi-AZ
  ├── No public access
  └── Backup subnet group
```

## Console

**VPC → Network ACLs** — manage NACLs
**EC2 → Network Interfaces** — manage ENIs
**VPC → Elastic IPs** — manage EIPs

## Gotchas

- **NACL rules are stateless** — always create both inbound AND outbound rules
- **NACL rule numbers**: use increments of 10 or 100 so you can insert rules later
- **ENI Source/Dest Check** must be disabled for NAT instances and network appliances
- **EIPs cost money when idle** — don't leave them unassociated
- **EIP associated with an ENI persists** even after instance termination (if ENI not deleted)
- **ENIs are AZ-scoped** — can only attach to instances in the same AZ
- **Primary ENI (eth0) persists** after instance termination (unless instance is set to auto-delete)
