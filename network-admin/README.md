# AWS for Network Administrators — 30-Day Learning Path

A comprehensive, day-by-day curriculum designed for network professionals moving to AWS.
No fluff, no AI/ML — pure networking.

## Structure

**Week 1 — Foundations** (Days 1–7): Core AWS networking building blocks
**Week 2 — Connectivity & Routing** (Days 8–14): Connecting VPCs, offices, and the internet
**Week 3 — LB, CDN & Security** (Days 15–20): Load balancing, content delivery, protection
**Week 4 — Monitoring, Hybrid & Advanced** (Days 21–30): Operations, multi-region, capstone

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
| **Week 2: Connectivity & Routing** |
| 8 | VPC Peering — Cross-account, cross-region |
| 9 | Transit Gateway — Hub-and-spoke, multi-VPC |
| 10 | VPC Endpoints — Gateway Endpoints, Interface Endpoints (PrivateLink) |
| 11 | Site-to-Site VPN |
| 12 | Client VPN |
| 13 | Direct Connect — Dedicated, hosted, LAG |
| 14 | Route 53 — DNS, hosted zones, routing policies |
| **Week 3: LB, CDN & Security** |
| 15 | Elastic Load Balancers — ALB, NLB, CLB |
| 16 | CloudFront CDN — Distributions, origins, behaviors |
| 17 | AWS WAF — Web ACLs, rules, rate limiting |
| 18 | AWS Shield — DDoS protection |
| 19 | AWS Network Firewall — Stateful inspection |
| 20 | Global Accelerator — Anycast, traffic optimization |
| **Week 4: Monitoring, Hybrid & Advanced** |
| 21 | VPC Flow Logs — Capture, analyze, Athena |
| 22 | Reachability Analyzer, Network Manager, IPAM |
| 23 | Route 53 Resolver — DNS resolution, inbound/outbound endpoints |
| 24 | PrivateLink & VPC Lattice — Service-to-service networking |
| 25 | Multi-Region Architecture — Disaster recovery, cross-region |
| 26 | Hybrid Networking — VPN + DX + TGW together |
| 27 | Network Security — Encryption, TLS, compliance |
| 28 | Edge Networking — Local Zones, Wavelength, Outposts |
| 29 | Cost Optimization for Networking |
| 30 | Capstone — Design an enterprise multi-VPC, multi-region hybrid network |
