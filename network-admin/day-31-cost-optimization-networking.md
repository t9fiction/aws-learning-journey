# Day 29 — Cost Optimization for Networking

## Concept

Networking costs in AWS can be **significant** — often 10-30% of total cloud spend. Understanding where costs come from and how to reduce them is a key skill for network admins.

## Where Networking Costs Come From

| Component | Typical Monthly Cost | Notes |
|-----------|---------------------|-------|
| **Data Transfer Out** | Variable | Biggest cost driver |
| **NAT Gateway** | $32 + $0.045/GB | Per AZ |
| **Direct Connect Port** | $135-$1,350+ | Per connection |
| **Transit Gateway** | $0.05/hr + attachments + data | Per region |
| **VPC Endpoints (Interface)** | $7.20/month per AZ + $0.01/GB | Per endpoint |
| **Client VPN** | $0.01/hr per connection | Per user |
| **Network Firewall** | $0.395/hr + $0.065/GB | Per AZ |
| **Global Accelerator** | $18/month + premium data | Fixed fee |
| **Load Balancers** | $16-18/month + LCU/NLCU | Per LB |
| **WAF** | $5/month per web ACL + $0.60/1M requests | Added costs for managed rules |

## Data Transfer Pricing

### Egress Tiers (per month, us-east-1)

| Tier | Price/GB |
|------|----------|
| First 1 GB | Free |
| Up to 10 TB | $0.09 |
| 10-50 TB | $0.085 |
| 50-150 TB | $0.07 |
| 150-500 TB | $0.05 |
| 500+ TB | Negotiated |

### Inter-Region

| Direction | Price/GB |
|-----------|----------|
| Region A → Region B | $0.01-0.02 |
| Region → Local Zone | $0.01-0.05 |
| Region → Internet | $0.05-0.09 |

### Intra-Region (between AZs)

| Path | Price/GB |
|------|----------|
| AZ A → AZ B (via public IP) | $0.01 |
| AZ A → AZ B (via private IP) | $0.01 |
| AZ A → AZ A | Free |
| TGW AZ A → TGW AZ B | $0.02 |

## Cost Reduction Strategies

### 1. Use VPC Endpoints for Everything

Replace NAT Gateway data processing with free/cheaper VPC Endpoints:

```bash
# Bad: traffic to S3 goes through NAT Gateway ($0.045/GB + egress)
# Good: traffic to S3 goes through Gateway Endpoint (free)
aws ec2 create-vpc-endpoint \
  --vpc-id vpc-12345678 \
  --service-name com.amazonaws.us-east-1.s3 \
  --route-table-ids rtb-private
```

**Savings**: $0.045/GB saved for every S3 API call from private subnets.

### 2. Consolidate NAT Gateways

```
# Expensive: NAT in every AZ (3 AZs = $96/month)
# Cheaper: Single NAT with cross-AZ routing (risks availability)
# Balanced: One NAT per AZ but share transit traffic
```

**If you can tolerate AZ failover**: use one NAT Gateway, accept the cross-AZ data charge ($0.01/GB).

### 3. Optimize Direct Connect

- Use **hosted connections** (from partners) instead of dedicated — cheaper for <1 Gbps
- Right-size port: 500 Mbps hosted vs 1 Gbps dedicated
- Use **Direct Connect Gateway** to share one connection across multiple regions

### 4. Use TGW Efficiently

- TGW charges per attachment ($0.05/hr per VPC attached)
- Group small VPCs into fewer attachments
- Use TGW peering instead of multiple VPN connections

### 5. Choose the Right Load Balancer

| LB Type | Cost/Month | When to Use |
|---------|-----------|-------------|
| ALB | ~$16 + LCU | HTTP/HTTPS apps |
| NLB | ~$16 + NLCU | TCP/UDP, fixed IP needed |
| CLB | ~$18 + LCU | Legacy (avoid) |

### 6. Reduce Egress with CDN

- CloudFront data transfer is **cheaper** than direct S3 egress
- CloudFront: $0.085/GB (US); S3 direct: $0.09/GB
- Plus CloudFront caches → less origin load

### 7. Compress and Optimize Data

```bash
# Enable compression on ALB
aws elbv2 modify-load-balancer-attributes \
  --load-balancer-arn arn:aws:elasticloadbalancing:us-east-1:xxx:loadbalancer/app/web-alb/xxx \
  --attributes Key=routing.http.compression,Value=true

# Use gzip/brotli at origin — smaller payloads = less data transfer
```

### 8. Monitor Cross-AZ Traffic

Cross-AZ data transfer is $0.01/GB each way. Co-locate services in the same AZ when possible:

```bash
# Identify cross-AZ traffic sources
# VPC Flow Logs can show traffic between AZs
# Query: source AZ ≠ destination AZ
```

### 9. Avoid Data Transfer "Hairpinning"

```
# Bad: Traffic goes NAT → Internet → IGW back into same region
# Good: Use VPC Endpoint or route directly
```

### 10. Use Savings Plans / Reserved Capacity

- Compute Savings Plans apply to NAT Gateway, ALB, NLB
- Direct Connect has 1-year and 3-year terms

## Cost Monitoring Tools

### AWS Cost Explorer

```bash
# View data transfer costs
aws ce get-cost-and-usage \
  --time-period Start=$(date -d '30 days ago' +%Y-%m-%d),End=$(date +%Y-%m-%d) \
  --granularity MONTHLY \
  --metrics "UnblendedCost" \
  --group-by Type=DIMENSION,Key=SERVICE \
  --filter '{"Dimensions": {"Key": "Service", "Values": ["Amazon Elastic Compute Cloud - Compute", "Amazon Simple Storage Service"]}}'
```

### VPC Flow Logs → Athena

Track top cost drivers:

```sql
-- Data transfer by destination (egress)
SELECT dstaddr, SUM(bytes) as total_bytes,
  CASE
    WHEN SUM(bytes) > 1099511627776 THEN '>1TB'
    WHEN SUM(bytes) > 107374182400 THEN '>100GB'
    ELSE '<100GB'
  END as size_tier
FROM vpc_flow_logs
WHERE action='ACCEPT'
  AND dstaddr NOT LIKE '10.%'
  AND dstaddr NOT LIKE '172.1[6-9].%'
  AND dstaddr NOT LIKE '172.2[0-9].%'
  AND dstaddr NOT LIKE '172.3[0-1].%'
  AND dstaddr NOT LIKE '192.168.%'
GROUP BY dstaddr
ORDER BY total_bytes DESC
LIMIT 20;
```

## Budget Alerts

```bash
aws budgets create-budget \
  --account-id 123456789012 \
  --budget '{
    "BudgetName": "Network-Transfer",
    "BudgetLimit": {"Amount": 5000, "Unit": "USD"},
    "TimeUnit": "MONTHLY",
    "BudgetType": "COST",
    "CostFilters": {"Service": ["Amazon EC2", "Amazon S3", "AWS Direct Connect"]},
    "CostTypes": {"IncludeTax": false}
  }' \
  --notifications-with-subscribers '[
    {
      "Notification": {
        "ComparisonOperator": "GREATER_THAN",
        "Threshold": 80,
        "ThresholdType": "PERCENTAGE",
        "NotificationType": "ACTUAL"
      },
      "Subscribers": [{"SubscriptionType": "EMAIL", "Address": "netops@example.com"}]
    }
  ]'
```

## CLI Commands for Cost Analysis

```bash
# EC2 data transfer
aws ec2 describe-instances \
  --query 'Reservations[*].Instances[*].{ID:InstanceId,AZ:Placement.AvailabilityZone,Type:InstanceType}' \
  --output table

# NAT Gateway data processing
aws cloudwatch get-metric-statistics \
  --namespace AWS/NATGateway \
  --metric-name BytesOutToDestination \
  --dimensions Name=NatGatewayId,Value=nat-xxx \
  --start-time $(date -d '7 days ago') --end-time $(date +%s) \
  --period 86400 --statistics Sum

# Direct Connect port hours
aws directconnect describe-connections
```

## Gotchas

- **Data transfer IN is free** but out is not — don't optimize for inbound
- **NAT Gateway has both hourly + data charges** — big bill if high traffic
- **Cross-region data**: costs add up fast — $1/GB for 100TB
- **CloudFront**: first 1TB free, then cheaper than direct egress
- **VPC Endpoints (Interface)**: $7.20/month each — remove unused ones
- **TGW attachments add up** at $36/month per VPC
- **No data compression** in transit by default — implement at app level
- **Data transfer costs are hard to predict** — monitor with Cost Explorer
