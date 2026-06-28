# Day 1 — AWS Global Infrastructure

## Concept

AWS has the largest global footprint of any cloud provider. As a network admin, you need to understand the physical topology because it governs latency, resilience, and data sovereignty.

## Hierarchy

```
Region (e.g. us-east-1)
  └─ Availability Zones (AZs) — 3-6 per region
       └─ Data Centers — each AZ has 1+ data centers
            └─ Racks, Servers, Network Switches
```

### Region
- A geographic area with 2+ AZs (e.g. us-east-1 = N. Virginia, eu-west-1 = Ireland)
- **Completely independent** — isolated from other regions
- **Data doesn't leave a region unless you explicitly move it**
- Not all services are in all regions

### Availability Zone (AZ)
- **One or more data centers** within a region, ~10 km apart
- Connected via **redundant, high-bandwidth fiber** (< 2 ms latency between AZs)
- Independent power, cooling, physical security
- An AZ failure won't affect other AZs
- **Name pattern**: `us-east-1a`, `us-east-1b`, `us-east-1c` (mapped differently per account for load balancing)

### Data Center
- 50k–80k servers per facility
- Redundant power (UPS + generators), cooling, network
- Multiple fiber providers for internet connectivity

### Edge Locations / PoPs
- **400+ Points of Presence** worldwide
- Used by CloudFront (CDN), Global Accelerator, Route 53
- Cache content closer to users — reduces latency
- **Not compute** — CDN cache and DNS only

### Local Zones
- Extensions of regions — put compute closer to end-users
- Example: us-east-1-nyc-1a
- Good for latency-sensitive apps (gaming, media, healthcare)

### Wavelength Zones
- Embedded in 5G carrier networks (Verizon, etc.)
- Single-digit millisecond latency to mobile users

## Why This Matters to Network Admins

| Decision | Impact |
|----------|--------|
| Region choice | Latency to users, compliance (GDPR, etc.) |
| Multi-AZ | High availability (survive AZ failure) |
| Cross-region traffic | $0.02/GB egress cost, higher latency |
| Edge locations | Cache static content, reduce origin load |

## Key Facts

- 30+ regions, 95+ AZs globally
- Each region has **min 2, max 6** AZs (most have 3)
- Inter-AZ latency: < 2 ms
- Inter-region latency: 10–100+ ms depending on distance
- Data transfer **into** AWS is free
- Data transfer **out** is charged (e.g. $0.09/GB for first 10 TB from us-east)

## CLI Commands

### Configuration & Credentials

```bash
# Interactive setup (profile default)
aws configure

# Interactive setup (named profile)
aws configure --profile myprofile

# View current config
aws configure list
aws configure list --profile myprofile

# Get/set specific config values
aws configure get region
aws configure set region us-east-1
aws configure get region --profile myprofile
aws configure set output json

# List all named profiles
aws configure list-profiles

# Verify identity & permissions
aws sts get-caller-identity

# Manually set credentials (less common; prefer `aws configure`)
aws configure set aws_access_key_id AKIAxxxxxxxxxxxx
aws configure set aws_secret_access_key xxxxxxxxxxxxxxxxxxxx
```

### Region & AZ Discovery

```bash
# Describe regions
aws ec2 describe-regions

# Describe AZs in a region
aws ec2 describe-availability-zones --region us-east-1

# Describe AZs with detailed info
aws ec2 describe-availability-zones --all-availability-zones

# Pricing for data transfer
# (check via AWS Pricing Calculator)
```

## Console

**EC2 Dashboard → Global view** shows regions.
**VPC Dashboard** shows AZs per region.
**CloudFront Console** shows edge locations.

## Design Notes

- **Always deploy across 2+ AZs** for production
- Choose regions closest to your users
- Use multiple regions for disaster recovery
- Be aware of data transfer costs between regions
- Not all instance types are available in all AZs

## Gotchas

- AZ names (`us-east-1a`) are **different for each AWS account** — don't hardcode them
- Some older regions (us-east-1, eu-west-1) have more services than newer ones
- China regions (cn-north-1, cn-northwest-1) require separate account registration
- **GovCloud** regions (us-gov-west-1, us-gov-east-1) for US government workloads
