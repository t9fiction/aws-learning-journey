# Day 10 — AWS Organizations, RAM & Config

## AWS Organizations

A service for **centrally managing multiple AWS accounts** as a group. Essential for any enterprise network admin.

### Structure

```
Organization Root
  ├── Organizational Unit (OU): Security
  │   ├── Account: Security (GuardDuty, Config, Logs)
  │   └── Account: Network (TGW, DX, VPN)
  ├── OU: Infrastructure
  │   ├── Account: Prod
  │   ├── Account: Staging
  │   └── Account: Dev
  └── OU: Workloads
      └── Account: Team-A
```

### Service Control Policies (SCPs)

SCPs set **permission guardrails** — they restrict what accounts can do, even if IAM allows it. For network admins, SCPs are critical for governance.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Deny",
      "Action": [
        "ec2:DeleteFlowLogs",
        "ec2:DeleteVpcFlowLogs"
      ],
      "Resource": "*"
    },
    {
      "Effect": "Deny",
      "Action": [
        "ec2:CreateInternetGateway",
        "ec2:AttachInternetGateway"
      ],
      "Resource": "*",
      "Condition": {
        "StringNotEquals": {
          "aws:PrincipalTag/NetworkAdmin": "true"
        }
      }
    }
  ]
}
```

### Common Network SCPs

| SCP | Why |
|-----|-----|
| Deny `DeleteFlowLogs` | Prevent disabling network audit |
| Deny `DetachInternetGateway` on prod VPCs | Prevent accidental internet disconnection |
| Deny `ModifyVpcAttribute` for DNS settings | Prevent DNS misconfiguration |
| Deny creating SGs that allow 0.0.0.0/0 on port 22/3389 | Enforce security posture |
| Require specific resource tags on VPCs | Enforce cost allocation |

### AWS Config

Continuously **evaluates resource configuration** against rules and detects drift.

### Why Network Admins Need Config

| What Config Detects | Why It Matters |
|--------------------|----------------|
| SG allows unrestricted ingress on port 22 | Security violation |
| VPC Flow Logs disabled | Audit/compliance gap |
| Route table has no routes to NAT Gateway | Outbound connectivity broken |
| SG not attached to any ENI | Unused resources |
| EBS volume not encrypted | Compliance violation |
| VPC default SG open to all traffic | Security risk |

### Setup

```bash
# 1. Enable Config (per-region)
aws configservice put-configuration-recorder \
  --configuration-recorder name=default,roleARN=arn:aws:iam::xxx:role/ConfigRole

aws configservice put-delivery-channel \
  --delivery-channel name=default,s3BucketName=my-config-bucket

aws configservice start-configuration-recorder \
  --configuration-recorder-name default

# 2. Add managed rules
aws configservice put-config-rule \
  --config-rule '{
    "ConfigRuleName": "restricted-common-ports",
    "Source": {
      "Owner": "AWS",
      "SourceIdentifier": "RESTRICTED_INCOMING_TRAFFIC"
    }
  }'

aws configservice put-config-rule \
  --config-rule '{
    "ConfigRuleName": "vpc-flow-logs-enabled",
    "Source": {
      "Owner": "AWS",
      "SourceIdentifier": "VPC_FLOW_LOGS_ENABLED"
    }
  }'

aws configservice put-config-rule \
  --config-rule '{
    "ConfigRuleName": "s3-bucket-public-read-prohibited",
    "Source": {
      "Owner": "AWS",
      "SourceIdentifier": "S3_BUCKET_PUBLIC_READ_PROHIBITED"
    }
  }'
```

### Useful Managed Rules for Network Admins

| Rule | What It Checks |
|------|---------------|
| `VPC_FLOW_LOGS_ENABLED` | Flow logs are on |
| `RESTRICTED_INCOMING_TRAFFIC` | SGs don't allow unrestricted SSH/RDP |
| `EC2_SECURITY_GROUP_ATTACHED_TO_ENI` | No unused SGs |
| `VPC_DEFAULT_SECURITY_GROUP_CLOSED` | Default SG has no rules |
| `S3_BUCKET_PUBLIC_READ_PROHIBITED` | No public S3 buckets |
| `INTERNET_GATEWAY_AUTHORIZED_VPC_ONLY` | IGW only on approved VPCs |
| `EC2_INSTANCE_NO_PUBLIC_IP` | EC2 in private subnet has no public IP |

### Config Remediation

Auto-fix non-compliant resources using Systems Manager automation:

```bash
aws configservice put-remediation-configuration \
  --config-rule-name "restricted-common-ports" \
  --remediation-configuration '{
    "TargetType": "SSM_DOCUMENT",
    "TargetId": "AWS-DisablePublicAccessForSecurityGroup",
    "Parameters": {
      "GroupId": {"ResourceValue": {"Value": "RESOURCE_ID"}}
    },
    "Automatic": true,
    "MaximumAutomaticAttempts": 3,
    "RetryAttemptSeconds": 60
  }'
```

### Config Aggregator (Multi-Account)

View compliance across all accounts:

```bash
aws configservice put-configuration-aggregator \
  --configuration-aggregator-name "Org-Aggregator" \
  --organization-aggregation-source '{
    "RoleArn": "arn:aws:iam::xxx:role/ConfigAggregatorRole",
    "AwsRegions": ["us-east-1", "eu-west-1"],
    "AllAwsRegions": false
  }'
```

## Resource Access Manager (RAM)

Share network resources **across accounts** without VPC peering.

### What RAM Can Share

| Resource | Shared As |
|----------|-----------|
| **Subnets** | Other accounts launch instances into your subnet (no cross-account Peering needed) |
| **Transit Gateway** | Other accounts attach their VPCs to your TGW |
| **Prefix Lists** | Other accounts reference your prefix lists in SG rules |
| **Route 53 Resolver rules** | Share outbound DNS forwarding rules |
| **Licenses** | License Manager configurations |

### Share a Subnet

```bash
# Create a resource share
aws ram create-resource-share \
  --name "Shared-Prod-Subnets" \
  --resource-arns \
    arn:aws:ec2:us-east-1:111111111111:subnet/subnet-prod-1a \
    arn:aws:ec2:us-east-1:111111111111:subnet/subnet-prod-1b \
  --principals 999999999999
```

### Share a Transit Gateway

```bash
aws ram create-resource-share \
  --name "TGW-Share-Prod" \
  --resource-arns arn:aws:ec2:us-east-1:111111111111:transit-gateway/tgw-xxx \
  --principals 999999999999
```

The consumer then creates a TGW attachment to their VPC.

### RAM vs VPC Peering

| Feature | VPC Peering | RAM (Subnet Sharing) |
|---------|-------------|---------------------|
| **Network isolation** | Full separate VPC | Shared subnet (same VPC) |
| **Cross-account** | Yes | Yes |
| **Route management** | Manual per connection | Single VPC route table |
| **Security** | SG can reference peer SG | Standard SG rules |
| **Transitive routing** | No | Yes (VPC router handles) |
| **Use case** | Different apps need isolation | Shared services, same app |

## Service Quotas

Network quotas you'll hit regularly:

| Resource | Default Quota | Can Increase? |
|----------|--------------|---------------|
| VPCs per region | 5 | Yes |
| Subnets per VPC | 200 | Yes |
| SGs per VPC | 500 | Yes |
| Rules per SG | 60 inbound + 60 outbound | Yes |
| VPC peering per VPC | 100 | Yes |
| NAT Gateways per AZ | 5 | Yes |
| EIPs per region | 5 | Yes |
| TGW per region | 5 | Yes |
| TGW attachments | 5000 | Yes |

```bash
# View quotas
aws service-quotas list-service-quotas \
  --service-code vpc

# Request increase
aws service-quotas request-service-quota-increase \
  --service-code vpc \
  --quota-code L-F678F1CE \
  --desired-value 25
```

## CLI Commands

```bash
# Organizations
aws organizations create-organization
aws organizations create-ou
aws organizations create-account
aws organizations attach-policy  # SCP

# Config
aws configservice put-configuration-recorder
aws configservice put-config-rule
aws configservice describe-compliance-by-config-rule

# RAM
aws ram create-resource-share
aws ram get-resource-share-associations

# Service Quotas
aws service-quotas list-service-quotas
aws service-quotas request-service-quota-increase
```

## Console

**Organizations** — manage accounts, OUs, SCPs
**Config** — rules, compliance dashboard, aggregator
**RAM** — resource shares, shared with me
**Service Quotas** — view and request increases

## Gotchas

- **SCPs don't affect the management account** — only member accounts
- **Config costs**: $0.003 per configuration item recorded + $0.001 per rule evaluation
- **RAM-shared subnets**: the owner manages routing, the consumer launches resources
- **RAM sharing is within an Organization** unless using `--principals` for external accounts
- **Quota increases**: require AWS support level (Business or Enterprise for fast track)
- **Config aggregator**: must be in the same region as the source accounts' Config
- **SCPs are evaluated before IAM** — if SCP denies it, IAM allow doesn't matter
