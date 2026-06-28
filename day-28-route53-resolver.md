# Day 28 — Route 53 Resolver

## Concept

Route 53 Resolver provides **DNS resolution** between your VPC and your on-premises network. It's a managed DNS resolver that allows hybrid DNS — VPC resources can resolve on-prem hostnames and vice versa.

## DNS in a VPC (Default Behavior)

By default, every VPC has DNS resolution through the **VPC DNS Server** at `10.0.0.2` (the second IP in the VPC CIDR):

- **VPC + AmazonProvidedDNS**: resolves Internet domains + the private DNS names of EC2 instances
- **Custom DHCP options**: you can override the DNS server
- **DNS resolution**: must be enabled on the VPC

```bash
aws ec2 modify-vpc-attribute \
  --vpc-id vpc-12345678 \
  --enable-dns-support

aws ec2 modify-vpc-attribute \
  --vpc-id vpc-12345678 \
  --enable-dns-hostnames
```

## Route 53 Resolver Components

### Inbound Endpoint

Resolve **your on-prem DNS queries** against your VPC. On-prem servers forward queries to this endpoint.

```
On-prem DNS ──► Route 53 Resolver Inbound Endpoint ──► VPC resources
                     │
                     ▼
               Route 53 Private Hosted Zones
```

```bash
aws route53resolver create-resolver-endpoint \
  --name "inbound-endpoint" \
  --direction INBOUND \
  --security-group-ids sg-resolver \
  --ip-addresses \
    SubnetId=subnet-1a,Ip=10.0.1.100 \
    SubnetId=subnet-1b,Ip=10.0.2.100
```

### Outbound Endpoint

Resolve **VPC DNS queries** against your on-prem DNS. The VPC forwards queries to this endpoint.

```
VPC resources ──► Route 53 Resolver Outbound Endpoint ──► On-prem DNS
```

```bash
aws route53resolver create-resolver-endpoint \
  --name "outbound-endpoint" \
  --direction OUTBOUND \
  --security-group-ids sg-resolver \
  --ip-addresses \
    SubnetId=subnet-1a,Ip=10.0.1.101 \
    SubnetId=subnet-1b,Ip=10.0.2.101
```

### Resolver Rules

Tell the outbound endpoint which domain to forward to which DNS server.

#### Forward Rule (VPC → On-prem)

```bash
aws route53resolver create-resolver-rule \
  --creator-request-id $(date +%s) \
  --rule-type FORWARD \
  --domain-name "corp.example.com" \
  --target-ips \
    Ip=10.0.1.200,Port=53 \
    Ip=10.0.1.201,Port=53 \
  --resolver-endpoint-id rslvr-out-xxx
```

#### System Rule (Automatic)

Route 53 Resolver automatically handles:
- Amazon-provided DNS
- Route 53 Private Hosted Zones
- This cannot be modified

#### Auto-Defined Rules (from other accounts)

For shared Route 53 Private Zones across accounts.

### Associate Rule with VPC

```bash
aws route53resolver associate-resolver-rule \
  --resolver-rule-id rslvr-rr-xxx \
  --vpc-id vpc-12345678
```

## Complete Hybrid DNS Flow

```
Scenario: EC2 in VPC needs to resolve "db.corp.example.com"

1. EC2 queries VPC DNS (10.0.0.2)
2. VPC DNS checks:
   - Is it a Private Hosted Zone? → No (corp.example.com is on-prem)
3. Matches forward rule for corp.example.com
4. Forwards to Outbound Endpoint
5. Outbound Endpoint queries on-prem DNS (10.0.1.200)
6. On-prem DNS responds with 192.168.1.50
7. Response flows back: VPC DNS → EC2
```

## Reverse DNS (On-prem → VPC)

```
Scenario: On-prem server needs to resolve "web.internal.example.com"

1. On-prem DNS queries Inbound Endpoint (10.0.1.100)
2. Inbound Endpoint checks Private Hosted Zone for internal.example.com
3. Returns the VPC private IP
4. On-prem server connects directly to 10.0.1.50
```

## Resolver Query Logging

Log all DNS queries sent by Route 53 Resolver to CloudWatch Logs or S3:

```bash
aws route53resolver create-resolver-query-log-config \
  --name "dns-query-log" \
  --destination-arn arn:aws:logs:us-east-1:xxx:log-group:/aws/route53resolver/*

aws route53resolver associate-resolver-query-log-config \
  --resolver-query-log-config-id rqlc-xxx \
  --vpc-id vpc-12345678
```

## CLI Commands

```bash
# Endpoints
aws route53resolver create-resolver-endpoint
aws route53resolver list-resolver-endpoints
aws route53resolver update-resolver-endpoint
aws route53resolver delete-resolver-endpoint

# Rules
aws route53resolver create-resolver-rule
aws route53resolver list-resolver-rules
aws route53resolver associate-resolver-rule
aws route53resolver disassociate-resolver-rule

# Query Logging
aws route53resolver create-resolver-query-log-config
aws route53resolver list-resolver-query-log-configs
aws route53resolver associate-resolver-query-log-config
```

## DNS Firewall

Combine with Route 53 Resolver DNS Firewall to block DNS queries to known bad domains:

```bash
# Create domain list
aws route53resolver create-firewall-domain-list \
  --name "blocked-malware" \
  --domains file://malware-domains.txt

# Create firewall rule group
aws route53resolver create-firewall-rule-group \
  --name "dns-firewall-group"

# Add rule — block the domains
aws route53resolver create-firewall-rule \
  --firewall-rule-group-id fwrg-xxx \
  --firewall-domain-list-id fwdl-xxx \
  --priority 100 \
  --action BLOCK \
  --block-response NXDOMAIN

# Associate with VPC
aws route53resolver associate-firewall-rule-group \
  --firewall-rule-group-id fwrg-xxx \
  --vpc-id vpc-12345678 \
  --priority 100
```

## Console

**Route 53 → Resolver → Inbound endpoints** — create inbound
**Route 53 → Resolver → Outbound endpoints** — create outbound
**Route 53 → Resolver → Rules** — forward rules
**Route 53 → Resolver → Firewall** — DNS firewall
**Route 53 → Resolver → Query logs** — logging configuration

## Costs

| Component | Cost |
|-----------|------|
| Inbound endpoint | $0.125/hour per AZ |
| Outbound endpoint | $0.125/hour per AZ |
| DNS queries | Included (no additional charge) |
| Query logging | CloudWatch Logs standard rates |
| DNS Firewall | $0.001/hour per rule group + $0.0004 per thousand queries |

## Gotchas

- **Endpoints are AZ-scoped** — create in 2+ AZs for HA
- **Inbound endpoint IPs** must be in subnets that can reach your on-prem network
- **Outbound endpoint uses rules** — without a matching rule, traffic won't forward
- **Cross-account** Route 53 Private Zones require RAM sharing
- **DNS Firewall works on outbound traffic only** — it filters VPC → internet DNS queries
- **Query logging cost** can be significant for high-traffic VPCs
- **Cannot use Route 53 Resolver to resolve .amazonaws.com** internally — those go to the VPC endpoint DNS
