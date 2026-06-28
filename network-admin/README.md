# AWS for Network Administrators — 32-Day Learning Path

A comprehensive, day-by-day curriculum designed for network professionals moving to AWS.
No fluff, no AI/ML — pure networking.

## Structure

**Week 1 — Foundations** (Days 1–7): Core AWS networking building blocks + compute
**Week 2 — Connectivity & Routing** (Days 8–16): Compute, core services, VPC connectivity, DNS
**Week 3 — LB, CDN & Security** (Days 17–22): Load balancing, content delivery, protection
**Week 4 — Monitoring, Hybrid & Advanced** (Days 23–32): Operations, multi-region, capstone

## Daily Format

Each day file covers:
- **Concept** — What it is and why it matters to a network admin
- **How it works** — Architecture, components, constraints
- **Hands-on** — CLI / Console walkthrough
- **Key commands** — Quick-reference `aws` commands
- **Routing & design notes** — How this affects traffic flow
- **Gotchas** — Common pitfalls
- **Cost notes** — Pricing implications

## Prerequisites

- Basic networking knowledge (CIDR, TCP/IP, DNS, routing)
- An AWS account (free tier is sufficient)
- AWS CLI installed and configured

---

## Curriculum

| Day | Topic |
|-----|-------|
| **Week 1: Foundations** |
| 1 | AWS Global Infrastructure — Regions, AZs, Edge Locations |
| 2 | IAM for Network Admins — Users, Roles, Policies, CLI |
| 3 | VPC Deep Dive — CIDR, creation, default vs custom VPC |
| 4 | Subnets, Route Tables, Internet Gateway |
| 5 | NAT Gateway, NAT Instance, Egress-Only Internet Gateway |
| 6 | Security Groups — Stateful Firewall |
| 7 | Network ACLs — Stateless Firewall, Elastic Network Interfaces, Elastic IPs |
| 8 | EC2 & Compute Fundamentals — Instances, AMIs, key pairs, Session Manager |
| 9 | Core AWS Services — S3, EBS, RDS, CloudWatch, CloudTrail |
| **Week 2: Connectivity & Routing** |
| 10 | VPC Peering — Cross-account, cross-region |
| 11 | Transit Gateway — Hub-and-spoke, multi-VPC |
| 12 | VPC Endpoints — Gateway Endpoints, Interface Endpoints (PrivateLink) |
| 13 | Site-to-Site VPN |
| 14 | Client VPN |
| 15 | Direct Connect — Dedicated, hosted, LAG |
| 16 | Route 53 — DNS, hosted zones, routing policies |
| **Week 3: LB, CDN & Security** |
| 17 | Elastic Load Balancers — ALB, NLB, CLB |
| 18 | CloudFront CDN — Distributions, origins, behaviors |
| 19 | AWS WAF — Web ACLs, rules, rate limiting |
| 20 | AWS Shield — DDoS protection |
| 21 | AWS Network Firewall — Stateful inspection |
| 22 | Global Accelerator — Anycast, traffic optimization |
| **Week 4: Monitoring, Hybrid & Advanced** |
| 23 | VPC Flow Logs — Capture, analyze, Athena |
| 24 | Reachability Analyzer, Network Manager, IPAM |
| 25 | Route 53 Resolver — DNS resolution, inbound/outbound endpoints |
| 26 | PrivateLink & VPC Lattice — Service-to-service networking |
| 27 | Multi-Region Architecture — Disaster recovery, cross-region |
| 28 | Hybrid Networking — VPN + DX + TGW together |
| 29 | Network Security — Encryption, TLS, compliance |
| 30 | Edge Networking — Local Zones, Wavelength, Outposts |
| 31 | Cost Optimization for Networking |
| 32 | Capstone — Design an enterprise multi-VPC, multi-region hybrid network |
