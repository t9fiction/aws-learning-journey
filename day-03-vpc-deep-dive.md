# Day 3 — VPC Deep Dive

## Concept

A Virtual Private Cloud (VPC) is your **isolated network** inside AWS. It's the foundation for everything — EC2, RDS, Lambda, and all other services run inside a VPC.

## What a VPC Gives You

- **Logical isolation** from other AWS customers
- **Full control** over IP addressing, subnets, routing
- **No latency penalty** — same physical hardware, just logically isolated
- **Analogous to a traditional VLAN + router** in a data center

## Key Parameters When Creating a VPC

### CIDR Block (IP Range)
- **IPv4 only** (or dual-stack with IPv6)
- Size: `/16` (65536 IPs) to `/28` (16 IPs)
- **Cannot be changed** after creation
- Must be **private** (RFC 1918):
  - `10.0.0.0/8` — largest, use this for enterprise
  - `172.16.0.0/12` — medium
  - `192.168.0.0/16` — small
- Can add secondary CIDRs later

### Tenancy
- **Default** — instances can be shared or dedicated hardware
- **Dedicated** — all instances run on single-tenant hardware (expensive)

## Reserved IPs (Every subnet)

AWS **reserves 5 IPs per subnet** — the first 4 and the last 1:

| IP | Purpose |
|----|---------|
| `.0` | Network address |
| `.1` | VPC router |
| `.2` | DNS server (Amazon DNS) |
| `.3` | Reserved for future use |
| `.255` | Broadcast (not supported) |

Example: In `10.0.1.0/24`, usable IPs are `10.0.1.4` — `10.0.1.254` (251 IPs)

**Factor this into subnet sizing:**

| CIDR | Total IPs | Usable IPs |
|------|-----------|------------|
| /24 | 256 | 251 |
| /23 | 512 | 507 |
| /22 | 1024 | 1019 |
| /21 | 2048 | 2043 |
| /20 | 4096 | 4091 |

## Default VPC vs Custom VPC

### Default VPC
- **Created automatically** in each region for every account
- CIDR: `172.31.0.0/16` with `/20` subnets per AZ
- Internet Gateway attached
- Route tables pre-configured
- Public IP assigned automatically to instances
- **Convenient but not production-ready**

### Custom VPC
- **You create and control everything**
- No internet access by default (you add it)
- You choose CIDR, subnets, routing
- **Always use custom VPCs for production**

## VPC Components (Built-in)

| Component | Purpose |
|-----------|---------|
| **VPC Router** | Routes traffic between subnets and gateways |
| **DHCP Options Set** | DNS servers, domain name |
| **DNS Resolution** | Amazon-provided DNS |
| **DNS Hostnames** | Assign DNS names to instances |
| **Main Route Table** | Default route table for all subnets (can be overridden) |
| **Network ACL** | Stateless firewall at subnet level |

## How the VPC Router Works

The VPC router is an **implicit, always-on** device that:
- Routes traffic between subnets in the same VPC
- Routes traffic to/from attached gateways (IGW, NAT, VPN, DX)
- Uses route tables to make forwarding decisions
- Is fully managed — you can't SSH into it

## DHCP Options Sets

Controls **DNS servers, domain name, and NTP** for instances in the VPC.

### Default DHCP Options
- **DNS**: AmazonProvidedDNS (Route 53 Resolver at VPC CIDR +2)
- **Domain name**: `<region>.compute.internal` (e.g., `us-east-1.compute.internal`)
- **NTP**: Amazon Time Sync Service (169.254.169.123)

### Custom DHCP Options

Useful when you want instances to use your own DNS servers:

```bash
# Create DHCP options
aws ec2 create-dhcp-options \
  --dhcp-configurations \
    "Key=domain-name-servers,Values=10.0.0.10,10.0.0.11" \
    "Key=domain-name,Values=corp.example.com" \
    "Key=ntp-servers,Values=169.254.169.123"

# Associate with VPC
aws ec2 associate-dhcp-options \
  --dhcp-options-id dopt-12345678 \
  --vpc-id vpc-12345678
```

### Key Facts
- **One DHCP options set per VPC** (associated, not inherited)
- **Cannot modify** a DHCP options set — create a new one and associate
- Changing DHCP options may cause a **brief DNS interruption** (DHCP renewal)
- Amazon Route 53 Resolver inbound/outbound endpoints (Day 28) override this for hybrid DNS
- Custom DNS servers must be reachable from the VPC

## CLI Commands

```bash
# Create a VPC
aws ec2 create-vpc --cidr-block 10.0.0.0/16

# Create VPC with DNS support
aws ec2 create-vpc \
  --cidr-block 10.0.0.0/16 \
  --amazon-provided-ipv6-cidr-block

# Enable DNS hostnames
aws ec2 modify-vpc-attribute \
  --vpc-id vpc-12345678 \
  --enable-dns-hostnames

# Describe VPCs
aws ec2 describe-vpcs

# Describe VPC with details
aws ec2 describe-vpcs --vpc-ids vpc-12345678

# Add secondary CIDR
aws ec2 associate-vpc-cidr-block \
  --vpc-id vpc-12345678 \
  --cidr-block 10.1.0.0/16

# Describe VPC CIDR blocks
aws ec2 describe-vpcs \
  --vpc-ids vpc-12345678 \
  --query 'Vpcs[0].CidrBlockAssociationSet'

# Delete VPC (must be empty)
aws ec2 delete-vpc --vpc-id vpc-12345678
```

## Console

**VPC → Your VPCs** — create, view, manage
**VPC → Subnets** — view subnet IP utilization
**VPC Dashboard** — overview of all VPC resources

## Design Notes

| Use Case | Recommended CIDR | Subnets |
|----------|-----------------|---------|
| Small project | `10.0.0.0/20` | 2 |
| Dev/Test | `10.0.0.0/16` | 4 |
| Production | `10.0.0.0/16` | 6+ |
| Enterprise (multi-VPC) | `10.0.0.0/8` (carve into /16s) | Many |

## Gotchas

- **CIDR cannot be changed** after creation — plan ahead
- VPCs are **region-scoped** — cannot span regions
- You can have **5 VPCs per region** (soft limit — can be increased)
- **Peered VPCs cannot have overlapping CIDRs**
- Secondary CIDRs **can** overlap with other VPCs, but primary cannot in a peering
- VPC deletion fails if dependencies exist (subnets, IGW, etc.)
- DNS resolution and DNS hostnames are **separate settings** — both often needed
