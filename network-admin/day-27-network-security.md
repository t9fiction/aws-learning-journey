# Day 27 — Network Security

## Concept

Network security in AWS follows the **defense-in-depth** model — multiple layers of controls working together.

## Security Layers

```
Layer 1: Edge (Shield + CloudFront + Global Accelerator)
Layer 2: VPC Boundaries (IGW/NAT/TGW/DX — route tables)
Layer 3: Subnet Boundaries (Network ACLs)
Layer 4: Instance Boundaries (Security Groups)
Layer 5: Host (OS firewall, WAF agent, IPS agent)
Layer 6: Application (WAF, request validation)
```

## Encryption in Transit

### TLS/SSL

| Service | TLS Termination |
|---------|----------------|
| **ALB** | Yes — upload cert to ACM or import |
| **CloudFront** | Yes — ACM (must be us-east-1) |
| **NLB** | Yes (TLS listener) |
| **Global Accelerator** | No — passes through to backend |
| **EC2 direct** | N/A — app handles it |

```bash
# ACM request
aws acm request-certificate \
  --domain-name "*.example.com" \
  --validation-method DNS \
  --region us-east-1
```

### IPSec

Used by Site-to-Site VPN. Automatically configured:
- IKEv1 or IKEv2
- AES-128 or AES-256
- SHA-1 or SHA-256
- DH Groups 14, 15, 16, etc.

### MACsec (Direct Connect)

Encrypts physical DX connection at Layer 2:

```bash
aws directconnect update-connection \
  --connection-id dxcon-xxx \
  --encryption-mode should_encrypt
```

### AWS Certificate Manager (ACM)

| Feature | Detail |
|---------|--------|
| **Public certs** | Free, auto-renewed |
| **Private CA** | Create your own CA for internal certs |
| **Import** | Bring your own certs from external CAs |
| **Auto-renewal** | DNS validation needed (email validation deprecated) |

## Network Access Control

### Principle of Least Privilege

1. **Start with deny all** — default SG denies everything
2. **Open only what's needed** — specific ports, specific sources
3. **Review regularly** — find unused SGs, overly permissive rules

### Checking for Open Resources

```bash
# Find SGs that allow 0.0.0.0/0 on sensitive ports
aws ec2 describe-security-groups \
  --filters "Name=ip-permission.cidr,Values=0.0.0.0/0" \
  --query 'SecurityGroups[?length(IpPermissions[?FromPort==`22` || FromPort==`3389` || FromPort==`3306` || FromPort==`5432`]) > `0`]' \
  --output table

# Find SGs with no inbound rules (unused)
aws ec2 describe-security-groups \
  --query 'SecurityGroups[?!length(IpPermissions) && GroupName!=`default`]'

# Find S3 buckets with public access
aws s3api list-buckets --query 'Buckets[*].Name' --output text | \
  xargs -I {} aws s3api get-public-access-block --bucket {} 2>/dev/null
```

### Network Segmentation

```bash
# Separate tiers by subnet + NACL + SG
# NACL: broad subnet-level rules
# SG: fine-grained instance-level rules

# Example: Database tier
# NACL (subnet level) allows only 3306 from app CIDR
# SG (instance level) allows 3306 from app SG only
```

## VPC Security Features

### Source/Destination Check

Every ENI checks by default that traffic is destined for its IP. Disable for NAT instances and network appliances:

```bash
aws ec2 modify-network-interface-attribute \
  --network-interface-id eni-xxx \
  --source-dest-check "Value=false"
```

### Flow Logs for Anomaly Detection

```bash
# Detect unusual outbound traffic
aws logs start-query \
  --log-group-name vpc-flow-logs \
  --start-time $(date -d '1 hour ago' +%s) \
  --end-time $(date +%s) \
  --query-string 'filter action="ACCEPT" and dstaddr not like "10." and dstaddr not like "172." and dstaddr not like "192." | stats sum(bytes) as total by dstaddr | sort total desc | limit 20'
```

## DDoS Protection

| Layer | Protection |
|-------|-----------|
| **Network (L3/L4)** | Shield Standard (free) or Shield Advanced |
| **Application (L7)** | WAF rate-based rules + CloudFront |
| **DNS** | Route 53 (scales automatically, anycast DNS) |
| **Edge** | CloudFront absorbs attacks at edge |

```bash
# WAF rate limiting — block > 2000 requests in 5 minutes from one IP
aws wafv2 update-web-acl \
  --rules '[
    {
      "Name": "DDoS-RateLimit",
      "Priority": 0,
      "Action": {"Block": {}},
      "Statement": {
        "RateBasedStatement": {
          "Limit": 2000,
          "AggregateKeyType": "IP"
        }
      },
      "VisibilityConfig": {"MetricName": "DDoS-RateLimit", ...}
    }
  ]'
```

## Vulnerability Scanning

### Amazon Inspector (Network Reachability)

```bash
# Enable Inspector
aws inspector2 enable --resource-types EC2

# Network Reachability findings — automatically discovers:
# - Ports open to the internet
# - Ports accessible from specific CIDRs
# - Paths to sensitive resources
```

### Third-Party Scanners

- Use **Gateway Load Balancer** (GWLB) to chain security appliances
- Supports Palo Alto, Fortinet, Check Point, etc.

## Compliance Frameworks

| Framework | Key Network Requirements |
|-----------|------------------------|
| **PCI DSS** | Encrypt transmission, restrict network access to cardholder data |
| **HIPAA** | Encrypt PHI in transit, network segmentation |
| **SOC 2** | Access controls, monitoring, encryption |
| **FedRAMP** | Boundary protection, least privilege |
| **GDPR** | Data residency, access controls |

**AWS Artifact**: download compliance reports:

```bash
aws artifact get-report --document-id xxx
```

## Logging & Audit

### CloudTrail (API activity)

```bash
# Create CloudTrail for network event logging
aws cloudtrail create-trail \
  --name network-trail \
  --s3-bucket-name my-cloudtrail-logs \
  --is-multi-region-trail \
  --include-global-service-events
```

### Network Events Worth Auditing

| Event | Why |
|-------|-----|
| `CreateSecurityGroup` | New firewall rules |
| `AuthorizeSecurityGroupIngress` | Opening ports |
| `CreateVpc` | New VPC |
| `CreateVpcPeeringConnection` | Cross-VPC access |
| `CreateNetworkAclEntry` | New subnet firewall rules |
| `AttachInternetGateway` | Making subnet public |
| `ModifyVpcAttribute` | DNS settings changes |

## Security Checklist for Network Admins

- [ ] All VPCs use custom SGs (no default SG open)
- [ ] No public SGs on RDS, ElastiCache, or internal services
- [ ] S3 buckets not publicly accessible
- [ ] NACLs restrict sensitive ports
- [ ] VPC Flow Logs enabled on all VPCs
- [ ] CloudTrail enabled in all regions
- [ ] Shield Advanced for production workloads
- [ ] WAF rate limiting on internet-facing ALBs/CloudFront
- [ ] Access keys rotated every 90 days
- [ ] IAM roles used instead of long-term keys on EC2
- [ ] TLS 1.2+ enforced on all endpoints
- [ ] Route 53 Resolver DNS Firewall enabled
- [ ] Reachability Analyzer used to validate security posture
- [ ] Inspector network assessments enabled

## Gotchas

- **Default SG allows all outbound** — restrict in production
- **Don't rely on NACLs alone** — SGs provide stateful protection
- **Security Groups have no explicit DENY** — use NACL if you need to block specific IPs
- **CloudTrail** doesn't log S3 data events by default — enable data events for S3
- **Inspector Network Reachability** only works for EC2 — not for on-prem resources
- **ACM certificates** can't be exported (private key stays in AWS)
- **TLS termination at ALB/CloudFront** means traffic behind them is unencrypted by default — enable backend encryption
