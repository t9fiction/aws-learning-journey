# Day 30 — Capstone: Design an Enterprise Multi-VPC, Multi-Region Hybrid Network

## Scenario

You work for **Globex Corp**, a global enterprise with:

- **Headquarters** in New York (primary data center)
- **Branch offices** in London, Tokyo, Sydney
- **AWS presence**: us-east-1 (primary), eu-west-1 (DR)
- **Compliance**: PCI DSS for payment data
- **Requirements**:
  - Latency < 50ms for all users
  - RTO < 15 minutes for critical apps
  - All inter-VPC traffic must be inspected
  - Data at rest encrypted, in transit encrypted
  - DNS resolution for both on-prem and AWS resources
  - Separate environments: Prod, Staging, Dev
  - Cost-effective for non-production environments

## Your Task

Design the complete AWS network architecture. Use everything from Days 1-29.

## Solution

### 1. IP Address Planning

```
10.0.0.0/8 — Globex Corp (reserved for AWS + On-Prem)

On-Prem: 10.0.0.0/8 (summary)
  HQ: 10.1.0.0/16
  London: 10.2.0.0/16
  Tokyo: 10.3.0.0/16
  Sydney: 10.4.0.0/16

AWS us-east-1 (Prod): 172.16.0.0/12
  Shared Services: 172.16.0.0/20
  Production: 172.16.16.0/20
  Staging: 172.16.32.0/20
  Dev: 172.16.48.0/20
  Mgmt/Audit: 172.16.64.0/20

AWS eu-west-1 (DR): 172.17.0.0/12
  Similar sub-division

Transit Gateway CIDRs: 172.20.0.0/16 (for TGW peering)
Client VPN CIDR: 10.200.0.0/16
```

### 2. VPC Design (us-east-1)

**Each VPC will have:**

```
Availability Zone A                    Availability Zone B
  ┌─────────────────────┐             ┌─────────────────────┐
  │ Public Subnet       │             │ Public Subnet       │
  │ 172.16.16.0/24      │             │ 172.16.17.0/24      │
  │   - ALB              │             │   - ALB              │
  │   - NAT Gateway      │             │   - NAT Gateway      │
  ├─────────────────────┤             ├─────────────────────┤
  │ Private App Subnet  │             │ Private App Subnet  │
  │ 172.16.16.64/26     │             │ 172.16.17.64/26     │
  │   - EC2 (ASG)       │             │   - EC2 (ASG)       │
  ├─────────────────────┤             ├─────────────────────┤
  │ Private DB Subnet   │             │ Private DB Subnet   │
  │ 172.16.16.128/26    │             │ 172.16.17.128/26    │
  │   - RDS             │             │   - RDS             │
  ├─────────────────────┤             ├─────────────────────┤
  │ TGW Attachment      │             │ TGW Attachment      │
  │ 172.16.16.192/28    │             │ 172.16.17.192/28    │
  └─────────────────────┘             └─────────────────────┘
```

### 3. Connectivity Architecture

```
                         ┌─────────────────────────────────────────────┐
                         │               us-east-1                      │
                         │                    ┌─────────────────────┐  │
                         │                    │  TGW-Prod           │  │
                         │                    │  Route Table:       │  │
                         │                    │  Prod-VPC → TGW     │  │
                         │                    │  DR-VPC → Peering   │  │
                         │                    │  OnPrem → DX        │  │
                         │                    └────────┬────────────┘  │
                         │     ┌──────────────────────┼──────────────┐ │
                         │     │         ┌────────────┼──────────┐   │ │
                         │  ┌──┴──┐  ┌───┴───┐   ┌───┴───┐     │   │ │
                         │  │Prod │  │Shared │   │Dev/Stg│     │   │ │
                         │  │VPC  │  │Svc VPC│   │VPCs   │     │   │ │
                         │  └─────┘  └───────┘   └───────┘     │   │ │
                         └──────────────────────────────────────┼───┘─┘
                                                                │
                          ┌─────────────────────────────────────┼──┐
                          │         TGW Peering (us-east-1 ←→ eu-west-1)
                          │                                     │  │
                          │  eu-west-1                          │  │
                          │  ┌──────────────────────────────────┴┐ │
                          │  │           TGW-DR                  │ │
                          │  │  Route Table:                     │ │
                          │  │  Prod → TGW, OnPrem → DX         │ │
                          │  │  Primary → Peering                │ │
                          │  └──────────────┬────────────────────┘ │
                          │            ┌────┴────┐                 │
                          │            │ DR VPC  │                 │
                          │            └─────────┘                 │
                          └────────────────────────────────────────┘

On-Prem
  ┌──────────┐     ┌─────────────┐     ┌──────────────┐
  │ HQ (NY)  │────►│ Direct      │────►│ DX Gateway   │──► TGW
  │ 10.1.0.0 │     │ Connect     │     │              │
  └──────────┘     └─────────────┘     └──────┬───────┘
                                              │
  ┌──────────┐     ┌─────────────┐            │
  │ London   │────►│ Site-to-Site│────────────┘
  │ 10.2.0.0 │     │ VPN (backup)│
  └──────────┘     └─────────────┘

  ┌──────────┐
  │ Remote   │────► Client VPN ───► TGW ─── Shared Services VPC
  │ Employees│
  └──────────┘
```

### 4. Security Architecture

```
Internet Traffic
  │
  ▼
CloudFront (edge DDoS mitigation)
  │
  ▼
WAF (rate limiting, SQLi, XSS)
  │
  ▼
ALB (TLS termination, SG: 443 from CloudFront)
  │
  ▼
Network Firewall (stateful inspection, domain filtering)
  │
  ▼
EC2 App Tier (SG: app port from ALB only)
  │
  ▼
RDS DB Tier (SG: 3306/5432 from App SG only)

VPC Flow Logs → S3 → Athena (audit)
CloudTrail (API audit, all regions)
Inspector (network reachability)
Shield Advanced (DDoS protection)
```

### 5. DNS Architecture

```
Route 53 Public Zone: globex.com
  ├── app.globex.com → CloudFront (global)
  ├── api.globex.com → Global Accelerator
  └── *.globex.com → ALB (via ALIAS)

Route 53 Private Zone: internal.globex.com
  ├── db.internal.globex.com → RDS
  ├── api.internal.globex.com → ALB internal
  └── *.internal.globex.com → VPC private IPs

Route 53 Resolver:
  Inbound Endpoint: 172.16.64.10 (HQ DNS → AWS)
  Outbound Endpoint: 172.16.64.11 (AWS → HQ DNS)
  Forward Rule: globex.local → HQ DNS (10.1.0.10)
```

### 6. Disaster Recovery

```
Strategy: Warm Standby (RTO: <15 min, RPO: <1s)

Primary (us-east-1)                 DR (eu-west-1)
  ┌──────────────────┐              ┌──────────────────┐
  │ Active stack     │              │ Standby (minimal)│
  │ ALB (app)        │              │ ALB (single t3)  │
  │ EC2 (10 x m5.lrg)│              │ EC2 (2 x m5.lrg) │
  │ RDS (db.r5.xl)   │─────────────►│ RDS read replica │
  │ ElastiCache (5n) │─────────────►│ Replication      │
  │ S3 active        │─────────────►│ S3 CRR           │
  └──────────────────┘              └──────────────────┘

Route 53: Failover routing (health check = ALB /health)
  Primary: us-east-1 (weight 100)
  Secondary: eu-west-1 (weight 0, promoted on fail)
```

### 7. Cost Optimization

| Strategy | Savings |
|----------|---------|
| Dev/Stg NAT: single NAT, not per-AZ | ~$64/month |
| Dev/Stg: t3 instances, no DR | Variable |
| Gateway Endpoints for S3/DynamoDB | $0.045/GB on data |
| CloudFront for static assets | 15-20% on egress |
| Reserved capacity: DX, TGW | 30-40% |
| Dev/Stg: turn off outside business hours | 50% compute |

### 8. Implementation Order

```
Phase 1 (Week 1):
  ├── VPCs + Subnets + Route Tables
  ├── TGW + VPC attachments
  └── Shared Services VPC

Phase 2 (Week 2):
  ├── Direct Connect + backup VPN
  ├── Route 53 Resolver
  └── TGW peering to DR

Phase 3 (Week 3):
  ├── Security Groups + NACLs
  ├── WAF + Shield Advanced
  ├── Network Firewall
  └── VPC Flow Logs

Phase 4 (Week 4):
  ├── Route 53 DNS + health checks
  ├── RDS replication (DR)
  ├── S3 CRR
  └── Testing + failover drill
```

## Verification Checklist

- [ ] All VPCs have non-overlapping CIDRs
- [ ] VPC Flow Logs enabled on all VPCs
- [ ] TGW route tables only propagate what's needed
- [ ] Security Groups follow least privilege
- [ ] NACLs have ephemeral port rules (1024-65535)
- [ ] All internet-facing ALBs have WAF attached
- [ ] CloudFront in front of S3 static content
- [ ] Direct Connect + VPN (primary/backup)
- [ ] Route 53 health checks configured for failover
- [ ] Cross-region replication for RDS + S3
- [ ] Route 53 Resolver configured for hybrid DNS
- [ ] Budget alerts set for data transfer costs
- [ ] IAM policies follow least privilege (no wildcards)
- [ ] Shield Advanced enabled on production resources
- [ ] Network Firewall inspecting inter-VPC traffic

---

**Congratulations — you've completed the 30-Day AWS Networking curriculum!**

You now have the knowledge to design, implement, and operate enterprise-grade AWS networks — from basic VPCs to multi-region hybrid architectures.
