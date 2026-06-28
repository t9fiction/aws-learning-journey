# AWS for Network Administrators — 35-Day Learning Path

A comprehensive, day-by-day curriculum designed for network professionals moving to AWS.
No fluff, no AI/ML — pure networking.

## Weekly Overview

| Week | Days | Focus |
|------|------|-------|
| **Week 1** — Foundations | 1–7 | Core AWS networking building blocks |
| **Week 2** — Compute, Governance & Advanced VPC | 8–15 | EC2, core services, Organizations, traffic inspection, container networking, VPC connectivity |
| **Week 3** — Connectivity, DNS & LB | 16–22 | VPN, Direct Connect, Route 53, load balancers, CloudFront |
| **Week 4** — Security & Monitoring | 23–28 | WAF, Shield, Network Firewall, Global Accelerator, Flow Logs, Reachability Analyzer |
| **Week 5** — Hybrid & Capstone | 29–35 | PrivateLink, multi-region, hybrid, security, edge, cost optimization, capstone |

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

### 🟦 Week 1 — Foundations (Days 1–7)

| Day | Topic |
|-----|-------|
| 1 | AWS Global Infrastructure — Regions, AZs, Edge Locations |
| 2 | IAM for Network Admins — Users, Roles, Policies, CLI |
| 3 | VPC Deep Dive — CIDR, creation, default vs custom VPC |
| 4 | Subnets, Route Tables, Internet Gateway |
| 5 | NAT Gateway, NAT Instance, Egress-Only Internet Gateway |
| 6 | Security Groups — Stateful Firewall |
| 7 | Network ACLs — Stateless Firewall, Elastic Network Interfaces, Elastic IPs |

### 🟩 Week 2 — Compute, Governance & Advanced VPC (Days 8–15)

| Day | Topic |
|-----|-------|
| 8 | EC2 & Compute Fundamentals — Instances, AMIs, key pairs, Session Manager |
| 9 | Core AWS Services — S3, EBS, RDS, CloudWatch, CloudTrail |
| 10 | AWS Organizations, RAM & Config — Governance, resource sharing, compliance |
| 11 | Gateway Load Balancer & Traffic Inspection — GWLB, TGW appliance mode |
| 12 | Container Networking & Infrastructure as Code — ECS, EKS, IaC |
| 13 | VPC Peering — Cross-account, cross-region |
| 14 | Transit Gateway — Hub-and-spoke, multi-VPC |
| 15 | VPC Endpoints — Gateway Endpoints, Interface Endpoints (PrivateLink) |

### 🟨 Week 3 — Connectivity, DNS & LB (Days 16–22)

| Day | Topic |
|-----|-------|
| 16 | Site-to-Site VPN |
| 17 | Client VPN |
| 18 | Direct Connect — Dedicated, hosted, LAG |
| 19 | Route 53 — DNS, hosted zones, routing policies |
| 20 | Elastic Load Balancers — ALB, NLB, CLB |
| 21 | CloudFront CDN — Distributions, origins, behaviors |
| 22 | AWS WAF — Web ACLs, rules, rate limiting |

### 🟧 Week 4 — Security & Monitoring (Days 23–28)

| Day | Topic |
|-----|-------|
| 23 | AWS Shield — DDoS protection |
| 24 | AWS Network Firewall — Stateful inspection |
| 25 | Global Accelerator — Anycast, traffic optimization |
| 26 | VPC Flow Logs — Capture, analyze, Athena |
| 27 | Reachability Analyzer, Network Manager, IPAM |
| 28 | Route 53 Resolver — DNS resolution, inbound/outbound endpoints |

### 🟥 Week 5 — Hybrid & Capstone (Days 29–35)

| Day | Topic |
|-----|-------|
| 29 | PrivateLink & VPC Lattice — Service-to-service networking |
| 30 | Multi-Region Architecture — Disaster recovery, cross-region |
| 31 | Hybrid Networking — VPN + DX + TGW together |
| 32 | Network Security — Encryption, TLS, compliance |
| 33 | Edge Networking — Local Zones, Wavelength, Outposts |
| 34 | Cost Optimization for Networking |
| 35 | Capstone — Design an enterprise multi-VPC, multi-region hybrid network |
