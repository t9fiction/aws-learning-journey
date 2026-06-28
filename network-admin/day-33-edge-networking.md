# Day 33 — Edge Networking (Local Zones, Wavelength, Outposts)

## Concept

AWS edge networking extends AWS infrastructure **closer to end users and devices** — reducing latency beyond what standard regions can achieve.

## Comparison

| Feature | Region | Local Zone | Wavelength | Outposts |
|---------|--------|------------|------------|----------|
| **Distance from users** | 100-1000s km | ~50 km | 5G tower | In your DC |
| **Latency** | 10-100 ms | <5 ms | <1 ms | <1 ms |
| **Services available** | All AWS services | Subset (EC2, EBS, ECS, RDS) | Very limited (EC2, EBS) | Full region subset |
| **Management** | AWS managed | AWS managed | AWS managed | You manage hardware |
| **Connection to region** | Native | High-speed fiber | 5G + fiber | Fiber |
| **Pricing** | Standard | +20% compute premium | Premium | Hardware cost |

## Local Zones

AWS Local Zones place **compute, storage, and database** services closer to large population centers that don't have a full region.

### Available Services
- EC2 (select instance types)
- EBS (gp2, gp3, io1)
- ECS/EKS
- RDS (select engines)
- ElastiCache
- ALB/NLB
- VPC (subnets in Local Zone)

### Architecture

```
Region (us-east-1)
  ├── Availability Zone (us-east-1a)
  ├── Availability Zone (us-east-1b)
  └── Local Zone (us-east-1-nyc-1)
       ├── Subnet (10.0.100.0/24)
       └── EC2, RDS, ECS resources
```

### Setup

```bash
# 1. List available local zones
aws ec2 describe-availability-zones \
  --filters Name=zone-type,Values=local-zone \
  --all-availability-zones

# 2. Opt in to a local zone
# (Do this in the Console — no direct CLI for opting in)
# EC2 Dashboard → Account Attributes → Zones

# 3. Create subnet in the local zone
aws ec2 create-subnet \
  --vpc-id vpc-12345678 \
  --cidr-block 10.0.100.0/24 \
  --availability-zone us-east-1-nyc-1
```

### Use Cases
- **Media & entertainment**: real-time rendering, transcoding
- **Gaming**: game server hosting near players
- **Healthcare**: PACS image processing near hospitals
- **Retail**: point-of-sale processing

### Gotchas
- **Not all instance types** available (no GPU-heavy types in most LZs)
- **EBS only gp2/gp3/io1** — no io2 or magnetic
- **Data transfer**: out to internet = standard rates; LZ ↔ region = slightly higher
- **RDS supports Multi-AZ** but cross-LZ not recommended
- **You must opt in** per Local Zone

---

## Wavelength Zones

AWS Wavelength embeds AWS compute at **5G network edge** — inside telecom provider data centers (Verizon, Vodafone, etc.).

### Architecture

```
5G Device
    │
    ▼
5G Tower (radio)
    │
    ▼
Telecom Aggregation Site
    │
    ▼
AWS Wavelength Zone
  ├── EC2
  ├── EBS
  └── VPC subnet
    │
    ▼
AWS Region (via fiber backhaul)
```

### Available Services
- EC2 (limited instances: t3, m5, r5)
- EBS (gp2)
- VPC (subnets, SGs, route tables)
- Carrier Gateway (for 5G inbound)

### Use Cases
- **Autonomous vehicles**: real-time sensor processing
- **AR/VR**: low-latency rendering
- **IoT**: industrial equipment monitoring
- **Smart cities**: traffic cameras, video analytics

### Setup

```bash
# 1. Create VPC with Carrier Gateway
aws ec2 create-carrier-gateway --vpc-id vpc-12345678

# 2. Create subnet in the wavelength zone
aws ec2 create-subnet \
  --vpc-id vpc-12345678 \
  --cidr-block 10.0.200.0/24 \
  --availability-zone us-east-1-wl1-nyc-wlz-1

# 3. Add route to Carrier Gateway
aws ec2 create-route \
  --route-table-id rtb-wavelength \
  --destination-cidr-block 0.0.0.0/0 \
  --carrier-gateway-id cgw-xxx
```

### EBS Pricing
- Storage: $0.125/GB-month (~25% premium over region)
- Snapshots are stored in the parent region

### Gotchas
- **Very limited instance selection**
- **No RDS, no ECS, no Lambda** (at time of writing)
- **Carrier Gateway required** for inbound 5G traffic
- **No inter-Wavelength connectivity** — each Wavelength Zone is isolated
- **Data transfer to region** has higher cost

---

## Outposts

AWS Outposts is a **fully managed rack of AWS hardware** installed in your data center. You run AWS services locally while connected to an AWS region.

### Architecture

```
Your Data Center
  ┌─────────────────────┐
  │  AWS Outposts Rack  │◄──── Fiber ────► AWS Region
  │  ┌───────────────┐  │
  │  │ EC2, EBS      │  │
  │  │ ECS/EKS, RDS  │  │
  │  │ S3 (local)    │  │
  │  └───────────────┘  │
  │  Local Gateway (LGW)│──► Your on-prem network
  └─────────────────────┘
```

### Key Concepts

- **Outpost**: one or more racks in your facility
- **Outpost subnet**: a subnet in an Outpost (similar to AZ)
- **Local Gateway (LGW)**: connects Outpost to your on-prem network
- **Region**: the parent AWS region (us-east-1, etc.)

### Available Services
- EC2 (most instance types, including GPU)
- EBS (gp2, gp3, io1, st1, sc1)
- ECS/EKS
- RDS (Oracle, SQL Server, PostgreSQL, MySQL)
- S3 on Outposts
- ELB (ALB, NLB)

### Setup

```bash
# 1. Order Outpost (Console — not CLI)
# AWS ships the hardware to your data center

# 2. Create Outpost subnet
aws ec2 create-subnet \
  --vpc-id vpc-12345678 \
  --cidr-block 10.0.0.0/24 \
  --outpost-arn arn:aws:outposts:us-east-1:xxx:outpost/op-xxx
```

### Networking Details

| Component | Purpose |
|-----------|---------|
| **Service Link** | Outpost ↔ Region (encrypted, 10/25/100 Gbps) |
| **Local Gateway** | Outpost ↔ Your on-prem network |
| **Service Link Route Table** | Controls traffic between Outpost and region |
| **Local Gateway Route Table** | Controls traffic between Outpost and on-prem |

```bash
# Create local gateway route table
aws ec2 create-local-gateway-route-table-vpc-association \
  --local-gateway-route-table-id lgw-rtb-xxx \
  --vpc-id vpc-12345678
```

### Use Cases
- **Latency-sensitive**: sub-millisecond required
- **Data residency**: data must stay on-premises
- **Local data processing**: large datasets that can't move to cloud
- **Hybrid workloads**: lift-and-shift with minimal rearchitecture

### Gotchas
- **Hardware lead time**: 8-12 weeks to receive Outpost
- **Power and space**: you provide rack space, power, cooling
- **Service Link bandwidth**: minimum 1 Gbps dedicated fiber
- **RDS on Outposts**: licensing handled via AWS
- **S3 on Outposts**: not the same as S3 — uses bucket policy differently
- **Failure domain**: your data center's power/cooling — no AWS SLA
- **Software updates**: AWS handles patching via Service Link

---

## Comparison Summary

| Need | Solution |
|------|----------|
| Sub-10ms latency for many users | Local Zone |
| Sub-1ms for 5G users | Wavelength |
| Sub-1ms for your DC users | Outposts |
| Cache content globally | CloudFront + Edge Locations |
| TCP optimization + anycast IPs | Global Accelerator |
