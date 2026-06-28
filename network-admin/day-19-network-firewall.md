# Day 19 — AWS Network Firewall

## Concept

AWS Network Firewall is a **managed stateful firewall** for your VPC. It provides **inspection and filtering** of traffic at the VPC level — both east-west (between subnets) and north-south (internet/VPN/Direct Connect).

## Why Network Firewall?

| Feature | Benefit |
|---------|---------|
| **Stateful inspection** | Track connections, not just packets |
| **Intrusion Prevention (IPS)** | Suricata-based rules engine |
| **Domain filtering** | Allow/block by domain (FQDN) |
| **Protocol detection** | Identify protocols even on non-standard ports |
| **Centralized management** | Firewall policy across multiple VPCs |
| **Managed by AWS** | No instance to patch or scale |

## Architecture

```
Internet
    │
    ▼
  IGW
    │
    ▼
  Firewall (inspect traffic)
    │
    ├── Traffic matches ALLOW rule → forwarded to subnet
    └── Traffic matches DROP rule → blocked
    │
    ▼
  Public Subnet (web servers)
```

### Traffic Flow

Network Firewall **must be in its own subnet** and uses a **Gateway Load Balancer-like** architecture:

```
Source Subnet
     │
     ▼
Firewall Subnet (contains Network Firewall endpoint)
     │
     ▼
Destination Subnet

# Route table entries:
# Source → Firewall → Destination
# The route table has the firewall endpoint as a next-hop
```

## Components

### Firewall Policy
Central configuration — rules, rule groups, and default behavior.

```bash
aws network-firewall create-firewall-policy \
  --firewall-policy-name "strict-egress" \
  --firewall-policy '{
    "StatelessDefaultActions": ["aws:drop"],
    "StatelessFragmentDefaultActions": ["aws:drop"],
    "StatelessCustomActions": [],
    "StatefulRuleGroupReferences": [
      {
        "ResourceArn": "arn:aws:network-firewall:us-east-1:xxx:stateful-rulegroup/domain-filter/xxx"
      }
    ]
  }'
```

### Firewall
The deployed instance — placed in a VPC with firewall subnets.

```bash
aws network-firewall create-firewall \
  --firewall-name "vpc-firewall" \
  --firewall-policy-arn arn:aws:network-firewall:us-east-1:xxx:firewall-policy/strict-egress/xxx \
  --vpc-id vpc-12345678 \
  --subnet-mappings SubnetId=subnet-firewall-1a \
  --delete-protection
```

### Rule Groups

#### Stateful Rules (Suricata Compatible)
For connection tracking — supports Suricata rule syntax.

```bash
aws network-firewall create-rule-group \
  --type STATEFUL \
  --rule-group-name "block-malware-domains" \
  --capacity 100 \
  --rules '{
    "RulesSource": {
      "RulesString": "drop tcp $HOME_NET any -> $EXTERNAL_NET 443 (msg:\"block bad domain\"; tls_sni; content:\".evil.com\"; nocase; endswith; sid:100; rev:1;)"
    }
  }'
```

#### Stateless Rules (Simple 5-tuple)
Fast path — match by source/dest IP, port, protocol.

```bash
aws network-firewall create-rule-group \
  --type STATELESS \
  --rule-group-name "allow-rfc1918" \
  --capacity 100 \
  --rules '{
    "RulesSource": {
      "StatelessRulesAndCustomActions": {
        "StatelessRules": [{
          "RuleDefinition": {
            "MatchAttributes": {
              "Sources": [{"AddressDefinition": "10.0.0.0/8"}],
              "Destinations": [{"AddressDefinition": "10.0.0.0/8"}]
            },
            "Actions": ["aws:pass"]
          },
          "Priority": 1
        }]
      }
    }
  }'
```

## Rule Types Compared

| Feature | Stateless | Stateful |
|---------|-----------|----------|
| What it inspects | 5-tuple (IP, port, protocol) | Full connection tracking |
| Performance | Faster | More resource intensive |
| Return traffic | Must match rule too | Automatically allowed |
| Custom rules | Simple | Suricata syntax (complex) |
| Use case | Quick filtering, known IPs | Deep inspection, IPS, domains |

## Common Configurations

### Domain Filtering (Egress Control)

Block or allow outbound traffic to specific domains:

```bash
# Allow only approved domains
aws network-firewall create-rule-group \
  --type STATEFUL \
  --rule-group-name "allow-approved-domains" \
  --capacity 100 \
  --rules '{
    "RulesSource": {
      "RulesString": "pass tcp $HOME_NET any -> $EXTERNAL_NET 443 (msg:\"allow example.com\"; tls_sni; content:\".example.com\"; endswith; sid:1; rev:1;)\npass tcp $HOME_NET any -> $EXTERNAL_NET 443 (msg:\"allow aws.com\"; tls_sni; content:\".amazonaws.com\"; endswith; sid:2; rev:1;)\ndrop tcp $HOME_NET any -> $EXTERNAL_NET any (msg:\"block all other\"; sid:999; rev:1;)"
    }
  }'
```

### Intrusion Prevention (IPS)

Use AWS-managed threat intelligence rules:

```bash
aws network-firewall create-rule-group \
  --type STATELESS \
  --rule-group-name "ips-threat-signatures" \
  --capacity 100 \
  --rules '{
    "RulesSource": {
      "RulesString": "alert tcp $EXTERNAL_NET any -> $HOME_NET any (msg:\"ET MALWARE Possible Malware Download\"; content:\"|ff d8 ff e0|\"; within:4096; classtype:trojan-activity; sid:2000001; rev:1;)"
    }
  }'
```

### Block All Traffic Except Approved

Default deny approach:

1. Create stateless rule group with `aws:drop` as default action
2. Add stateless pass rules for approved CIDRs/protocols
3. Apply stateful rule group for content inspection

## Monitoring & Logging

### Alerts

```bash
aws network-firewall update-firewall-policy \
  --firewall-policy-name "strict-egress" \
  --firewall-policy '{
    "StatefulEngineOptions": {"RuleOrder": "DEFAULT_ACTION_ORDER"},
    "StatefulDefaultActions": ["aws:drop"]
  }'
```

### Logging to S3, CloudWatch, or Kinesis

```bash
aws network-firewall update-logging-configuration \
  --firewall-name "vpc-firewall" \
  --logging-configuration '{
    "LogDestinationConfigs": [
      {
        "LogType": "ALERT",
        "LogDestinationType": "S3",
        "LogDestination": {"bucketName": "nfw-logs", "prefix": "alerts"}
      },
      {
        "LogType": "FLOW",
        "LogDestinationType": "CloudWatchLogs",
        "LogDestination": {"logGroup": "/aws/network-firewall/flows"}
      }
    ]
  }'
```

### Metrics in CloudWatch

- `Packets` — packets inspected
- `Bytes` — bytes inspected
- `DropCount` — packets dropped by stateful rules
- `PassCount` — packets allowed
- `AlertCount` — rule alerts triggered

## Integration with Other Services

| Service | How They Work Together |
|---------|----------------------|
| **VPC** | Firewall placed in dedicated firewall subnet |
| **Route Tables** | Routes point to firewall endpoint |
| **Transit Gateway** | TGW routes traffic to firewall for inspection |
| **AWS WAF** | WAF at application layer, Network Firewall at network layer |
| **Shield** | Shield protects infrastructure while NFW inspects traffic |
| **CloudWatch** | Metrics, alarms, logging |

## CLI Commands

```bash
aws network-firewall create-firewall
aws network-firewall update-firewall
aws network-firewall delete-firewall
aws network-firewall describe-firewall
aws network-firewall create-firewall-policy
aws network-firewall update-firewall-policy
aws network-firewall create-rule-group
aws network-firewall update-rule-group
aws network-firewall update-logging-configuration
```

## Console

**VPC → Network Firewall → Firewalls** — manage firewalls
**VPC → Network Firewall → Network Firewall Policies** — manage policies
**VPC → Network Firewall → Rule Groups** — manage rule groups

## Gotchas

- **Firewall must be in its own subnet** — dedicated subnet, no other resources
- **Route tables must point to the firewall endpoint** — traffic won't be inspected otherwise
- **Stateful rules are Suricata-based** — learn Suricata syntax for custom rules
- **Capacity**: rule groups have capacity limits — estimate before creating
- **Endpoints cost**: $0.395/hour per AZ + data processing ($0.065/GB)
- **Dual-stack**: supports IPv4 and IPv6
- **Encryption**: can decrypt and inspect TLS if you provide certificates
- **Fail mode**: OPEN or CLOSED — what happens if the firewall is unhealthy
- **Automatic updates**: security signatures can be updated automatically (if subscribed to AWS-managed rules)
