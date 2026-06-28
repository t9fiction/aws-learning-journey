# Day 22 — AWS WAF (Web Application Firewall)

## Concept

AWS WAF is a **web application firewall** that protects your web applications from common web exploits — SQL injection, cross-site scripting (XSS), DDoS, and bot traffic.

## What WAF Protects

| Resource | How to Associate |
|----------|-----------------|
| CloudFront | Create WAF in us-east-1, attach to distribution |
| ALB | Create WAF in the same region, attach to ALB |
| API Gateway (REST/HTTP) | Create WAF in same region |
| AppSync GraphQL | Attach WAF web ACL |
| Cognito User Pool | Attach WAF web ACL |

## Core Components

### Web ACL (Access Control List)
The main container — a set of rules that inspect and act on web requests.

```bash
aws wafv2 create-web-acl \
  --name "my-web-acl" \
  --scope REGIONAL \            # or CLOUDFRONT
  --default-action "Allow={}" \
  --description "Main WAF for production" \
  --visibility-config '{
    "SampledRequestsEnabled": true,
    "CloudWatchMetricsEnabled": true,
    "MetricName": "my-web-acl"
  }'
```

### Rules
Each rule has: **inspection criteria** + **action**.

### Rule Actions

| Action | Behavior |
|--------|----------|
| **Allow** | Request continues to origin |
| **Block** | Request is rejected with 403 |
| **Count** | Track but don't block (useful for testing) |
| **CAPTCHA** | Present captcha challenge |
| **Challenge** | Present silent challenge (JS) |
| **Custom response** | Return custom status code + body |

## Managed Rule Groups

AWS and partners provide pre-built rule groups:

### AWS Managed Rules (Recommended)

| Rule Group | What It Blocks |
|------------|---------------|
| **AWSManagedRulesCommonRuleSet** | SQLi, XSS, LFI, RFI, SSRF, size constraints |
| **AWSManagedRulesSQLiRuleSet** | SQL injection |
| **AWSManagedRulesKnownBadInputsRuleSet** | Known bad patterns, user-agent inspection |
| **AWSManagedRulesAmazonIpReputationList** | IPs from threat intelligence |
| **AWSManagedRulesAnonymousIpList** | VPN, proxy, Tor nodes |
| **AWSManagedRulesBotControlRuleSet** | Scrapers, crawlers, bots (with targeted/ common level) |
| **AWSManagedRulesATPRuleSet** | Account takeover protection |
| **AWSManagedRulesACFPRuleSet** | Account creation fraud prevention |

```bash
aws wafv2 associate-web-acl \
  --web-acl-arn arn:aws:wafv2:us-east-1:xxx:regional/webacl/my-web-acl/xxx \
  --resource-arn arn:aws:elasticloadbalancing:us-east-1:xxx:loadbalancer/app/web-alb/xxx
```

## Custom Rules

### IP Set (Block specific IP ranges)

```bash
# Create IP set
aws wafv2 create-ip-set \
  --name "blocked-ips" \
  --scope REGIONAL \
  --ip-address-version IPV4 \
  --addresses "203.0.113.0/24" "198.51.100.0/24"

# Use in rule
aws wafv2 update-web-acl \
  --name "my-web-acl" \
  --scope REGIONAL \
  --id xxx \
  --rules '[
    {
      "Name": "BlockBadIPs",
      "Priority": 0,
      "Action": {"Block": {}},
      "Statement": {
        "IPSetReferenceStatement": {
          "ARN": "arn:aws:wafv2:us-east-1:xxx:regional/ipset/blocked-ips/xxx"
        }
      },
      "VisibilityConfig": {
        "SampledRequestsEnabled": true,
        "CloudWatchMetricsEnabled": true,
        "MetricName": "BlockBadIPs"
      }
    }
  ]'
```

### SQL Injection Protection

```bash
aws wafv2 update-web-acl \
  --rules '[
    {
      "Name": "SQLiProtection",
      "Priority": 1,
      "Action": {"Block": {}},
      "Statement": {
        "SqliMatchStatement": {
          "FieldToMatch": {
            "Body": {},
            "UriPath": {},
            "QueryString": {}
          },
          "TextTransformations": [{"Priority": 0, "Type": "URL_DECODE"}]
        }
      },
      "VisibilityConfig": {"MetricName": "SQLiProtection",...}
    }
  ]'
```

### Rate Limiting

```bash
aws wafv2 update-web-acl \
  --rules '[
    {
      "Name": "RateLimit",
      "Priority": 2,
      "Action": {"Block": {}},
      "Statement": {
        "RateBasedStatement": {
          "Limit": 2000,
          "AggregateKeyType": "IP"
        }
      },
      "VisibilityConfig": {"MetricName": "RateLimit",...}
    }
  ]'
```

Rate limit options:
- **2000 requests in 5 minutes** from a single IP (typical starting point)
- Can aggregate by IP, forwarded IP (via X-Forwarded-For), or custom key (header, cookie, query param)

### Geographic (Geo) Blocking

```bash
aws wafv2 update-web-acl \
  --rules '[
    {
      "Name": "GeoBlock",
      "Priority": 3,
      "Action": {"Block": {}},
      "Statement": {
        "GeoMatchStatement": {
          "CountryCodes": ["CN", "RU"]
        }
      },
      "VisibilityConfig": {"MetricName": "GeoBlock",...}
    }
  ]'
```

### Header Inspection

```bash
aws wafv2 update-web-acl \
  --rules '[
    {
      "Name": "BlockBadHeaders",
      "Priority": 4,
      "Action": {"Count": {}},    # Use Count first to test
      "Statement": {
        "ByteMatchStatement": {
          "FieldToMatch": {
            "SingleHeader": {"Name": "x-forwarded-for"}
          },
          "SearchString": "127.0.0.1",
          "PositionalConstraint": "CONTAINS",
          "TextTransformations": [{"Priority": 0, "Type": "NONE"}]
        }
      },
      "VisibilityConfig": {"MetricName": "BlockBadHeaders",...}
    }
  ]'
```

### Bot Control

Bot Control is a managed rule group with two levels:

```bash
aws wafv2 update-web-acl \
  --rules '[
    {
      "Name": "AWS-AWSBotControl",
      "Priority": 5,
      "Action": {"Block": {}},
      "Statement": {
        "ManagedRuleGroupStatement": {
          "VendorName": "AWS",
          "Name": "AWSManagedRulesBotControlRuleSet",
          "ExcludedRules": [],
          "ManagedRuleGroupConfigs": [{
            "AWSManagedRulesBotControlRuleSet": {
              "InspectionLevel": "TARGETED"
            }
          }]
        }
      },
      "VisibilityConfig": {"MetricName": "BotControl",...},
      "OverrideAction": {"None": {}}
    }
  ]'
```

## Logging

Send WAF logs to S3, CloudWatch Logs, or Kinesis:

```bash
aws wafv2 put-logging-configuration \
  --logging-configuration '{
    "ResourceArn": "arn:aws:wafv2:us-east-1:xxx:regional/webacl/my-web-acl/xxx",
    "LogDestinationConfigs": [
      "arn:aws:firehose:us-east-1:xxx:deliverystream/aws-waf-logs-xxx"
    ],
    "RedactedFields": [
      {"SingleHeader": {"Name": "authorization"}},
      {"SingleHeader": {"Name": "cookie"}}
    ]
  }'
```

## CLI Commands

```bash
aws wafv2 create-web-acl
aws wafv2 update-web-acl
aws wafv2 get-web-acl
aws wafv2 delete-web-acl
aws wafv2 associate-web-acl
aws wafv2 disassociate-web-acl
aws wafv2 create-ip-set
aws wafv2 create-regex-pattern-set
aws wafv2 list-available-managed-rule-groups
aws wafv2 get-sampled-requests
```

## Console

**WAF & Shield → Web ACLs** — full management
**WAF & Shield → IP sets** — IP-based rules
**WAF & Shield → Bot Control** — bot protection dashboard

## Gotchas

- **WAF is regional** except for CloudFront (global)
- **Rule evaluation order**: rules are evaluated in priority order (lowest number first)
- **Default action**: what happens when no rule matches (Allow or Block)
- **Capacity Units (WCU)**: each web ACL has capacity limit (default 1500, up to 5000)
- **Managed rule groups consume WCU** — e.g., CommonRuleSet costs 700 WCU
- **Test with Count action** before switching to Block — avoid false positives
- **Sample requests**: WAF captures sample requests for debugging (last 3 hours)
- **Bot Control costs extra**: ~$10/month + additional per-request charges
- **CAPTCHA**: adds cost but reduces false positives from bot detection
