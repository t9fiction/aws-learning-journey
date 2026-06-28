# Day 14 — Route 53 (DNS)

## Concept

Amazon Route 53 is a **cloud DNS service** — it translates domain names (like `example.com`) to IP addresses, and provides domain registration, health checking, and routing policies.

## Key Components

### Hosted Zones
A container for DNS records for a domain.

- **Public Hosted Zone**: DNS for internet-facing domains (resolves from anywhere)
- **Private Hosted Zone**: DNS for internal VPC resources (resolves only within your VPCs)

```bash
# Create public hosted zone
aws route53 create-hosted-zone \
  --name example.com \
  --caller-reference $(date +%s)

# Create private hosted zone (associated with VPC)
aws route53 create-hosted-zone \
  --name internal.example.com \
  --caller-reference $(date +%s) \
  --vpc VPCRegion=us-east-1,VPCId=vpc-12345678

# Associate another VPC (via Route 53 CLI)
# Must use the route53domains or direct API
aws route53 associate-vpc-with-hosted-zone \
  --hosted-zone-id Z1234567890 \
  --vpc VPCRegion=eu-west-1,VPCId=vpc-87654321
```

### DNS Record Types

| Type | Example | What It Does |
|------|---------|-------------|
| A | `example.com → 1.2.3.4` | Map name to IPv4 address |
| AAAA | `example.com → 2001:db8::1` | Map name to IPv6 address |
| CNAME | `www → example.com` | Alias of one name to another |
| MX | `@ → mail.example.com (priority 10)` | Mail exchange |
| TXT | `@ → "v=spf1 include:..."` | Text (SPF, DKIM, verification) |
| NS | `@ → ns-xxx.awsdns-xx.net` | Name servers for delegation |
| SOA | (auto) | Start of Authority — zone metadata |
| ALIAS | (Route 53 special) | Like CNAME but works at zone apex |

### ALIAS Records (Route 53 Exclusive)

Unlike CNAME (which can't be used at the root domain), ALIAS records work at the apex:

```bash
# Route example.com → ALB (works at apex)
aws route53 change-resource-record-sets \
  --hosted-zone-id Z1234567890 \
  --change-batch '{
    "Changes": [{
      "Action": "CREATE",
      "ResourceRecordSet": {
        "Name": "example.com",
        "Type": "A",
        "AliasTarget": {
          "HostedZoneId": "Z35SXDOTRQ7X7K",
          "DNSName": "my-alb-1234567890.us-east-1.elb.amazonaws.com",
          "EvaluateTargetHealth": true
        }
      }
    }]
  }'
```

Supported targets for ALIAS:
- Elastic Load Balancers (ALB, NLB, CLB)
- CloudFront distributions
- Elastic Beanstalk environments
- API Gateway
- VPC Interface Endpoints
- S3 website buckets
- Global Accelerator

## Routing Policies

### Simple (Default)
Single record, one value. Route 53 returns all IPs, client picks.

```bash
aws route53 change-resource-record-sets \
  --hosted-zone-id Z1234567890 \
  --change-batch '{
    "Changes": [{
      "Action": "UPSERT",
      "ResourceRecordSet": {
        "Name": "app.example.com",
        "Type": "A",
        "TTL": 300,
        "ResourceRecords": [{"Value": "1.2.3.4"}]
      }
    }]
  }'
```

### Weighted
Distribute traffic across multiple endpoints with weights.

```bash
# 80% to primary, 20% to secondary
aws route53 change-resource-record-sets \
  --hosted-zone-id Z1234567890 \
  --change-batch '{
    "Changes": [
      {
        "Action": "UPSERT",
        "ResourceRecordSet": {
          "Name": "app.example.com", "Type": "A",
          "SetIdentifier": "Primary", "Weight": 80,
          "AliasTarget": {
            "HostedZoneId": "Z35SXDOTRQ7X7K",
            "DNSName": "primary-alb.elb.amazonaws.com",
            "EvaluateTargetHealth": true
          }
        }
      },
      {
        "Action": "UPSERT",
        "ResourceRecordSet": {
          "Name": "app.example.com", "Type": "A",
          "SetIdentifier": "Secondary", "Weight": 20,
          "AliasTarget": {
            "HostedZoneId": "Z35SXDOTRQ7X7K",
            "DNSName": "secondary-alb.elb.amazonaws.com",
            "EvaluateTargetHealth": true
          }
        }
      }
    ]
  }'
```

### Latency-Based
Route users to the endpoint with lowest latency.

```bash
aws route53 change-resource-record-sets \
  --hosted-zone-id Z1234567890 \
  --change-batch '{
    "Changes": [
      {"Action": "UPSERT","ResourceRecordSet": {
        "Name": "app.example.com","Type": "A",
        "SetIdentifier": "US-East","Region": "us-east-1",
        "AliasTarget": {
          "HostedZoneId": "Z35SXDOTRQ7X7K",
          "DNSName": "us-east-alb.elb.amazonaws.com",
          "EvaluateTargetHealth": true
        }
      }},
      {"Action": "UPSERT","ResourceRecordSet": {
        "Name": "app.example.com","Type": "A",
        "SetIdentifier": "EU-West","Region": "eu-west-1",
        "AliasTarget": {
          "HostedZoneId": "Z35SXDOTRQ7X7K",
          "DNSName": "eu-west-alb.elb.amazonaws.com",
          "EvaluateTargetHealth": true
        }
      }}
    ]
  }'
```

### Failover
Active-passive — if primary is unhealthy, route to secondary.

```bash
aws route53 change-resource-record-sets \
  --hosted-zone-id Z1234567890 \
  --change-batch '{
    "Changes": [
      {"Action": "UPSERT","ResourceRecordSet": {
        "Name": "app.example.com","Type": "A",
        "SetIdentifier": "Primary","Failover": "PRIMARY",
        "AliasTarget": {
          "HostedZoneId": "Z35SXDOTRQ7X7K",
          "DNSName": "primary-alb.elb.amazonaws.com",
          "EvaluateTargetHealth": true
        }
      }},
      {"Action": "UPSERT","ResourceRecordSet": {
        "Name": "app.example.com","Type": "A",
        "SetIdentifier": "Secondary","Failover": "SECONDARY",
        "HealthCheckId": "xxx",
        "AliasTarget": {
          "HostedZoneId": "Z35SXDOTRQ7X7K",
          "DNSName": "secondary-alb.elb.amazonaws.com",
          "EvaluateTargetHealth": true
        }
      }}
    ]
  }'
```

### Geolocation
Route based on where the user is geographically.

```bash
aws route53 change-resource-record-sets \
  --hosted-zone-id Z1234567890 \
  --change-batch '{
    "Changes": [
      {"Action": "UPSERT","ResourceRecordSet": {
        "Name": "app.example.com","Type": "A",
        "SetIdentifier": "North-America",
        "GeoLocation": {"ContinentCode": "NA"},
        "AliasTarget": {"HostedZoneId": "...","DNSName": "na-alb.elb.amazonaws.com","EvaluateTargetHealth": true}
      }},
      {"Action": "UPSERT","ResourceRecordSet": {
        "Name": "app.example.com","Type": "A",
        "SetIdentifier": "Europe",
        "GeoLocation": {"ContinentCode": "EU"},
        "AliasTarget": {"HostedZoneId": "...","DNSName": "eu-alb.elb.amazonaws.com","EvaluateTargetHealth": true}
      }}
    ]
  }'
```

### Geo-Proximity (Traffic Flow only)
Route based on geographic distance, with bias.

```bash
# Uses Route 53 Traffic Flow policy
# Create via Console — JSON-based rules with bias values
```

### IP-Based
Route based on the user's IP address (CIDR).

```bash
# Uses CIDR location — define CIDR blocks, then route
aws route53 create-cidr-collection \
  --name "Corporate-Offices"
aws route53 change-cidr-collection \
  --id cidr-collection-xxx \
  --changes '[
    {"Action":"PUT","LocationName":"HQ","CidrList":["203.0.113.0/24"]},
    {"Action":"PUT","LocationName":"Branch","CidrList":["198.51.100.0/24"]}
  ]'
```

## Health Checks

Route 53 can monitor endpoint health and adjust DNS responses accordingly.

```bash
# Create HTTP health check
aws route53 create-health-check \
  --caller-reference $(date +%s) \
  --health-check-config '{
    "Type": "HTTPS",
    "FullyQualifiedDomainName": "app.example.com",
    "Port": 443,
    "ResourcePath": "/health",
    "RequestInterval": 30,
    "FailureThreshold": 3
  }'
```

| Check Type | What It Verifies |
|------------|-----------------|
| HTTP/HTTPS | Endpoint returns 2xx or 3xx |
| TCP | TCP handshake succeeds |
| CloudWatch | CloudWatch alarm state (sends requests to a CW alarm instead of an endpoint) |
| Calculated | Combined status of multiple checks (OR, AND) |

## Domain Registration

```bash
# Check availability
aws route53domains check-domain-availability \
  --domain-name example.com

# Register
aws route53domains register-domain \
  --domain-name example.com \
  --duration-in-years 1 \
  --admin-contact 'FirstName=John,LastName=Doe,...' \
  --registrant-contact ...
```

## CLI Commands

```bash
# Hosted Zones
aws route53 create-hosted-zone
aws route53 list-hosted-zones
aws route53 delete-hosted-zone

# Records
aws route53 change-resource-record-sets
aws route53 list-resource-record-sets

# Health Checks
aws route53 create-health-check
aws route53 list-health-checks
aws route53 delete-health-check

# Domains
aws route53domains register-domain
aws route53domains list-domains
```

## Console

**Route 53 → Hosted Zones** — manage DNS records
**Route 53 → Health Checks** — create monitoring
**Route 53 → Registered Domains** — manage domain registration

## Gotchas

- **NS record delegation**: Route 53 provides 4 name servers — point your registrar at them
- **TTL**: DNS changes can take up to 48 hours (TTL-dependent), set low TTL (60-300) during migrations
- **Zone apex CNAME**: Cannot use CNAME at root — use ALIAS instead
- **Private zones**: Can be associated with VPCs in the same region only (for Private DNS), but via Route 53 APIs cross-region association is possible
- **Health check cost**: $0.75/month per health check
- **Query cost**: $0.40/month per million queries (first 1B free for public zones)
- **Weighted record weights**: must sum to a valid integer — can set to 0 to stop traffic
- **DNS query logging**: sends DNS queries to CloudWatch Logs
- **Route 53 Resolver**: for hybrid DNS (covered in Day 23)
