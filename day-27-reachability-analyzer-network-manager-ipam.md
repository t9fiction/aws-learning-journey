# Day 27 — Reachability Analyzer, Network Manager & IPAM

## Reachability Analyzer

### Concept

A **network diagnostic tool** that tests whether two endpoints in your VPC can communicate — no need to launch instances or send traffic. It simulates the VPC routing and firewall rules.

### Use Cases

- Verify connectivity before deploying resources
- Troubleshoot "why can't A reach B?"
- Validate changes to route tables, SGs, NACLs
- Document network paths for compliance

### What It Checks

- VPC route tables
- Security Group rules
- Network ACL rules
- VPC Peering configurations
- Transit Gateway routes
- VPC Gateway/Interface Endpoints

### How It Works

```bash
# Test connectivity from EC2 to RDS
aws ec2 create-network-insights-path \
  --source arn:aws:ec2:us-east-1:xxx:instance/i-12345678 \
  --destination arn:aws:ec2:us-east-1:xxx:network-interface/eni-rds-xxx \
  --destination-port 3306 \
  --protocol TCP

# Analyze the path
aws ec2 start-network-insights-analysis \
  --network-insights-path-id nip-xxx

# Get results
aws ec2 describe-network-insights-analyses \
  --network-insights-analysis-ids nia-xxx
```

### Output

```json
{
  "NetworkInsightsAnalyses": [{
    "NetworkPathFound": true,
    "ForwardPathComponents": [
      {
        "SequenceNumber": 1,
        "Component": {
          "Id": "arn:aws:ec2:us-east-1:xxx:instance/i-12345678",
          "Arn": "arn:aws:ec2:us-east-1:xxx:instance/i-12345678",
          "Name": "web-server"
        }
      },
      {
        "SequenceNumber": 2,
        "Component": {
          "Id": "arn:aws:ec2:us-east-1:xxx:subnet/subnet-12345678",
          "Arn": "arn:aws:ec2:us-east-1:xxx:subnet/subnet-12345678"
        },
        "Actions": [
          {"Action": "ROUTE", "AnalysisDetail": "Route 10.0.2.0/24 → local"},
          {"Action": "SECURITY_GROUP", "AnalysisDetail": "Rule: sg-app, port 3306 → ALLOW"}
        ]
      }
    ],
    "NetworkPathFound": true
  }]
}
```

### CLI Commands

```bash
aws ec2 create-network-insights-path
aws ec2 start-network-insights-analysis
aws ec2 describe-network-insights-analyses
aws ec2 delete-network-insights-path
```

### Console

**VPC → Reachability Analyzer** — create paths and run analyses

### Gotchas

- **Paths are regional** — cannot test cross-region connectivity
- **Destinations**: ENI, instance ID, or IP address
- **Maximum 100 paths per account per region**
- **Cost**: $0.10 per path analysis
- **Results may differ from live traffic** if routes change between analysis and actual traffic
- **Does not test internet or VPN connectivity** — only within AWS network

---

## Network Manager

### Concept

A **central dashboard** for managing your global network — including VPCs, TGWs, VPNs, and Direct Connect — all in one place.

### Key Features

- **Global view**: see all your AWS network resources across regions and accounts
- **Topology visualization**: auto-discovered network map
- **Monitoring**: CloudWatch metrics integrated for all network resources
- **Transit Gateway management** from a single pane
- **Network telemetry**: performance metrics, errors, bandwidth

### How to Set Up

```bash
# 1. Create a global network
aws networkmanager create-global-network \
  --description "Corporate Network"

# 2. Register TGW
aws networkmanager register-transit-gateway \
  --global-network-id gn-xxx \
  --transit-gateway-arn arn:aws:ec2:us-east-1:xxx:transit-gateway/tgw-xxx
```

### Console

**VPC → Network Manager** — dashboard with topology map

### Gotchas

- **VPN connections must be announced via BGP** to appear in topology
- **Cross-account**: works with RAM-shared resources
- **Cost**: $0.001/hour per monitored resource

---

## IPAM (IP Address Manager)

### Concept

A managed service for **planning, tracking, and monitoring** your IP address space across AWS regions and accounts.

### Why IPAM?

| Problem IPAM Solves | How |
|-------------------|-----|
| Overlapping CIDRs | Centralized IP space planning |
| Subnet exhaustion | Automatic IP utilization reporting |
| Manual tracking | Automated IP inventory |
| Multi-account sprawl | Cross-account IP management |

### How It Works

```
IPAM Hierarchy:

IPAM
  └── Private Scope (10.0.0.0/8)
       ├── Pool: us-east-1 (10.0.0.0/16)
       │    ├── Pool: Prod (10.0.0.0/17)
       │    │    ├── vpc-111 (10.0.0.0/24)
       │    │    └── vpc-222 (10.0.1.0/24)
       │    └── Pool: Dev (10.0.128.0/17)
       │         └── vpc-333 (10.0.128.0/24)
       └── Pool: eu-west-1 (10.1.0.0/16)
            └── ...
  └── Public Scope — for public IPs (optional)
```

### Setup

```bash
# 1. Create IPAM (operator account)
aws ec2 create-ipam \
  --description "Corporate IPAM" \
  --operating-regions RegionName=us-east-1 \
  --operating-regions RegionName=eu-west-1

# 2. Create a top-level pool
aws ec2 create-ipam-pool \
  --ipam-scope-id ipam-scope-private-xxx \
  --address-family ipv4 \
  --locale us-east-1 \
  --allocation-default-netmask-length 16

# 3. Create sub-pool
aws ec2 create-ipam-pool \
  --ipam-scope-id ipam-scope-private-xxx \
  --source-ipam-pool-id ipam-pool-top-xxx \
  --address-family ipv4 \
  --locale us-east-1 \
  --allocation-default-netmask-length 24 \
  --auto-import

# 4. Provision CIDR
aws ec2 provision-ipam-pool-cidr \
  --ipam-pool-id ipam-pool-prod-xxx \
  --cidr 10.0.0.0/16
```

### CLI Commands

```bash
aws ec2 create-ipam
aws ec2 describe-ipams
aws ec2 delete-ipam
aws ec2 create-ipam-pool
aws ec2 describe-ipam-pools
aws ec2 allocate-ipam-pool-cidr  # allocate CIDR to VPC
aws ec2 get-ipam-pool-allocations
```

### Console

**VPC → IPAM** — dashboard with IP usage and pool hierarchy

### Gotchas

- **IPAM operates at the organization level** (use AWS Organizations)
- **Cross-account**: one account hosts IPAM, other accounts provision from it
- **Cost**: $0.001/hour per IPAM + $0.00003/hour per monitored IP
- **Default quota**: 10 IPAMs per region per account
- **Automatic import**: VPCs with CIDRs matching a pool are auto-discovered
- **Does not manage on-premises IPs** — only AWS resources
