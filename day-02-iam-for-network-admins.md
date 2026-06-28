# Day 2 — IAM for Network Administrators

## Concept

Identity and Access Management (IAM) controls **who** can do **what** in AWS. As a network admin, you'll need to create users/roles for network services, manage access keys for the CLI, and understand how permissions affect network resources.

## Core Components

### Users
- Represents a person or service
- Has a **long-term password** (Console) and/or **access keys** (CLI/API)
- Can be a member of groups

### Groups
- Collection of users
- Attach policies to groups, not individual users (best practice)
- Example: `NetworkAdmins`, `SecurityAuditors`

### Roles
- **No long-term credentials** — temporary security tokens
- Assumed by users, services, or EC2 instances
- **Most important for network admins** — used by EC2, Lambda, VPN, etc.
- Example: `EC2-VPCServiceRole`, `NetworkAdminRole`

### Policies
- JSON document that defines permissions
- Written in IAM policy language
- **Allow** or **Deny** actions on resources

## IAM Policy Structure

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ec2:DescribeVpcs",
        "ec2:DescribeSubnets",
        "ec2:DescribeRouteTables",
        "ec2:CreateSecurityGroup",
        "ec2:AuthorizeSecurityGroupIngress"
      ],
      "Resource": "*"
    }
  ]
}
```

## Sample Network Admin Policy

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ec2:*",
        "vpc:*",
        "elasticloadbalancing:*",
        "route53:*",
        "directconnect:*",
        "cloudfront:*",
        "shield:*",
        "wafv2:*",
        "network-firewall:*"
      ],
      "Resource": "*"
    }
  ]
}
```

## Least Privilege Principle

**Only grant the permissions needed** — not full admin access.

```json
{
  "Effect": "Allow",
  "Action": [
    "ec2:DescribeVpcs",
    "ec2:DescribeSubnets",
    "ec2:DescribeRouteTables"
  ],
  "Resource": "*"
}
```

## CLI Setup for Network Admin

```bash
# Configure CLI with IAM user access keys
aws configure

# Verify who you are
aws sts get-caller-identity

# Check effective permissions
aws iam simulate-principal-policy \
  --policy-source-arn arn:aws:iam::123456789012:user/network-admin \
  --action-names ec2:CreateVpc ec2:DeleteVpc
```

## Key CLI Commands

```bash
# List users
aws iam list-users

# Create a group
aws iam create-group --group-name NetworkAdmins

# Attach managed policy to group
aws iam attach-group-policy \
  --group-name NetworkAdmins \
  --policy-arn arn:aws:iam::aws:policy/AmazonVPCFullAccess

# Create user
aws iam create-user --user-name netadmin

# Add user to group
aws iam add-user-to-group --group-name NetworkAdmins --user-name netadmin

# Create access keys
aws iam create-access-key --user-name netadmin

# Create role for EC2
aws iam create-role \
  --role-name NetworkMonitorRole \
  --assume-role-policy-document file://trust-policy.json

# Attach policy to role
aws iam attach-role-policy \
  --role-name NetworkMonitorRole \
  --policy-arn arn:aws:iam::aws:policy/AmazonVPCReadOnlyAccess
```

## Console

**IAM → Users** — create and manage users
**IAM → Roles** — create and assign roles
**IAM → Policies** — create custom policies
**IAM → Access Analyzer** — analyze public/ cross-account access

## Gotchas

- IAM is **global** — not region-specific
- Changes can take **seconds to minutes** to propagate
- Root user has **full unrestricted access** — enable MFA, use it only for account-level tasks
- **Access keys are displayed once** — save them immediately
- Delete unused access keys regularly
- Use **IAM Access Analyzer** to detect unintended public access
- Roles are preferred over long-term access keys

## Design Notes

- Create separate IAM users for each network admin
- Use groups for permission management
- Use roles for EC2 instances instead of storing keys on them
- Enable CloudTrail to audit all IAM actions
- Enable MFA for all users
- Rotate access keys every 90 days
