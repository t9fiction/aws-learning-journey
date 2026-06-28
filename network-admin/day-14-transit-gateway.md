# Day 14 — Transit Gateway

## Concept

Transit Gateway (TGW) is a **network transit hub** that connects VPCs, VPNs, and Direct Connect in a **single gateway**. It solves the transitive routing problem that VPC Peering has.

## Why Transit Gateway?

| Problem | VPC Peering | Transit Gateway |
|---------|-------------|-----------------|
| 10 VPCs communicating | Need 45 peer connections | Connect each to one TGW |
| Transitive routing | Not supported | Native |
| VPN to on-premises | Each VPC needs separate VPN | One VPN on TGW, shared |
| Cross-account | Complex | Simple with RAM |
| Route management | Manual per connection | Centralized |

## Architecture

```
              ┌──────────────────┐
              │  Transit Gateway │
              └──┬────┬────┬────┘
           ┌─────┘    │    └──────┐
           │          │           │
      ┌────┴───┐ ┌───┴───┐ ┌───┴────┐
      │ VPC A  │ │ VPC B │ │  VPN   │
      │ Prod   │ │ Dev   │ │ On-prem │
      └────────┘ └───────┘ └────────┘
```

## Key Concepts

### Attachment Types

| Attachment | What It Connects |
|------------|-----------------|
| VPC | Attach a VPC (subnets in each AZ) |
| VPN | Site-to-Site VPN connection |
| Direct Connect Gateway | DX Virtual Interface |
| Peering | Connect to another TGW (cross-region) |
| Connect | Connect SD-WAN appliances |

### Route Tables

TGW has its own route tables — separate from VPC route tables.

- **TGW route table**: controls traffic between attachments
- **VPC route table**: TGW appears as a target (`0.0.0.0/0 → tgw-xxx`)
- **Association**: each attachment is associated with a TGW route table
- **Propagation**: attachments can propagate routes into TGW route tables

### How Routing Works

```
VPC A (10.0.0.0/16)
  → VPC route table has route: 10.1.0.0/16 → tgw-xxx
  → TGW route table has route: 10.1.0.0/16 → VPC B attachment
  → TGW forwards to VPC B

On-prem (192.168.0.0/16)
  → VPN attachment propagates route: 192.168.0.0/16
  → TGW route table learns it
  → VPC A can reach on-prem via TGW
```

## CLI Setup

### Step 1: Create Transit Gateway

```bash
aws ec2 create-transit-gateway \
  --description "Main corporate TGW" \
  --options "AmazonSideAsn=64512,AutoAcceptSharedAttachments=enable,DefaultRouteTableAssociation=enable,DefaultRouteTablePropagation=enable"
```

### Step 2: Attach VPCs

```bash
# Create VPC attachment
aws ec2 create-transit-gateway-vpc-attachment \
  --transit-gateway-id tgw-12345678 \
  --vpc-id vpc-12345678 \
  --subnet-ids subnet-1a subnet-1b subnet-1c
```

### Step 3: Update VPC Route Tables

```bash
# In VPC A, route to VPC B via TGW
aws ec2 create-route \
  --route-table-id rtb-vpc-a \
  --destination-cidr-block 10.1.0.0/16 \
  --transit-gateway-id tgw-12345678
```

### Step 4: Create TGW Route Table and Associations

```bash
# Create TGW route table
TGW_RT_ID=$(aws ec2 create-transit-gateway-route-table \
  --transit-gateway-id tgw-12345678 \
  --query 'TransitGatewayRouteTable.TransitGatewayRouteTableId' --output text)

# Associate attachments
aws ec2 associate-transit-gateway-route-table \
  --transit-gateway-route-table-id $TGW_RT_ID \
  --transit-gateway-attachment-id tgw-attach-xxx

# Propagate routes from attachments
aws ec2 enable-transit-gateway-route-table-propagation \
  --transit-gateway-route-table-id $TGW_RT_ID \
  --transit-gateway-attachment-id tgw-attach-yyy
```

## VPN Attachment

```bash
# Create VPN attachment on TGW
aws ec2 create-vpn-connection \
  --customer-gateway-id cgw-xxx \
  --transit-gateway-id tgw-xxx \
  --type ipsec.1

# TGW handles routing — no need for separate VPN on each VPC
```

## Cross-Region TGW Peering

```bash
# Create peering attachment (requester)
aws ec2 create-transit-gateway-peering-attachment \
  --transit-gateway-id tgw-us-east-1 \
  --peer-transit-gateway-id tgw-eu-west-1 \
  --peer-region eu-west-1 \
  --peer-account-id 999999999999
```

## Resource Access Manager (RAM) — Cross-Account

Share TGW with other accounts:

```bash
# Create a resource share
aws ram create-resource-share \
  --name "TGW-Share" \
  --resource-arns arn:aws:ec2:us-east-1:111111111111:transit-gateway/tgw-xxx \
  --principals 999999999999
```

## Key CLI Commands

```bash
# Transit Gateway
aws ec2 create-transit-gateway
aws ec2 describe-transit-gateways
aws ec2 delete-transit-gateway

# Attachments
aws ec2 create-transit-gateway-vpc-attachment
aws ec2 describe-transit-gateway-attachments
aws ec2 delete-transit-gateway-vpc-attachment

# Route Tables
aws ec2 create-transit-gateway-route-table
aws ec2 associate-transit-gateway-route-table
aws ec2 enable-transit-gateway-route-table-propagation
aws ec2 create-transit-gateway-route

# Peering
aws ec2 create-transit-gateway-peering-attachment
aws ec2 accept-transit-gateway-peering-attachment
```

## Design Patterns

### Isolated Environments (Dev/Test/Prod)

```
TGW Route Table: "Production"
  Associations: Prod VPC, Prod VPN
  Propagations: Prod VPC, Prod VPN
  → Prod traffic stays isolated

TGW Route Table: "Non-Prod"
  Associations: Dev VPC, Test VPC
  Propagations: Dev VPC, Test VPC, Dev VPN
  → Dev and Test can talk; no access to Prod
```

### Shared Services

```
TGW Route Table: "Shared"
  Associations: Shared Services VPC, All workload VPCs
  Propagations: Shared Services VPC
  → Workloads can reach shared services (AD, DNS, monitoring)
  → Shared services knows about all VPCs (for return traffic)
```

## Gotchas

- **TGW is region-scoped** — use peering for cross-region
- **Max 5000 attachments per TGW**
- **50 Gbps per VPC attachment** (burst)
- **Data transfer costs** apply ($0.02/GB per AZ)
- **TGW route propagation**: routes from attachments must be propagated to relevant route tables
- **VPC route tables** still need explicit routes pointing to TGW
- **TGW supports multicast** (additional cost)
- **Blackhole routes** in TGW route table can be used to drop traffic to specific CIDRs
- **Default route table**: if you don't associate/ propagate, no traffic flows
