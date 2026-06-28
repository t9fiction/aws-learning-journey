# 03 — Authentication & Access Credentials

## Overview

AWS CLI needs **credentials** to prove who you are. There are several ways to authenticate.

## 1. IAM Users (Access Keys)

Traditional method — generate a long-lived Access Key + Secret Key.

```bash
aws configure
# Enter your Access Key ID and Secret Access Key
```

**Console GUI Equivalent**: IAM → Users → *your user* → Security credentials → Create access key.

## 2. IAM Roles (STS Temporary Credentials)

More secure — temporary credentials that expire.

```bash
# Assume a role
aws sts assume-role \
  --role-arn "arn:aws:iam::123456789012:role/MyRole" \
  --role-session-name "MySession"

# Use the returned credentials as env vars
export AWS_ACCESS_KEY_ID=...
export AWS_SECRET_ACCESS_KEY=...
export AWS_SESSION_TOKEN=...
```

**Profile-based role assumption** (`~/.aws/config`):
```ini
[profile dev]
role_arn = arn:aws:iam::123456789012:role/Developer
source_profile = default
```

Then: `aws s3 ls --profile dev`

## 3. IAM Identity Center (SSO)

Best for organizations — single sign-on with AWS.

```bash
# Step 1: Configure SSO
aws configure sso

# Step 2: Login
aws sso login --profile my-sso-profile

# Step 3: Use it
aws s3 ls --profile my-sso-profile

# Step 4: Logout
aws sso logout
```

**Config file entry**:
```ini
[profile my-sso-profile]
sso_session = my-sso
sso_account_id = 123456789012
sso_role_name = ReadOnlyAccess
region = us-west-2

[sso-session my-sso]
sso_region = us-west-2
sso_start_url = https://my-sso-portal.awsapps.com/start
sso_registration_scopes = sso:account:access
```

## 4. MFA (Multi-Factor Authentication)

```bash
# Generate temporary credentials with MFA token
aws sts get-session-token \
  --serial-number arn:aws:iam::123456789012:mfa/MyUser \
  --token-code 123456
```

**For roles + MFA** (`~/.aws/config`):
```ini
[profile mfa-role]
role_arn = arn:aws:iam::123456789012:role/MFARole
source_profile = default
mfa_serial = arn:aws:iam::123456789012:mfa/MyUser
```

## 5. EC2 Instance Metadata

If you run CLI on an EC2 instance with an IAM role attached — **no credentials needed**:

```bash
# Just run commands — credentials are auto-fetched from instance metadata
aws s3 ls
```

The CLI uses the EC2 instance's IAM role automatically (last in precedence).

## 6. External Credential Providers

For custom credential sources (e.g., HashiCorp Vault, external processes):

```ini
[profile external]
credential_process = /path/to/custom-script.sh
```

The script must output JSON with `Version`, `AccessKeyId`, `SecretAccessKey`, and optionally `SessionToken` + `Expiration`.

## Full Credential Precedence (Highest to Lowest)

1. Command-line `--profile` / `--region` etc.
2. Environment variables
3. `~/.aws/credentials` file
4. `~/.aws/config` file
5. Container credentials (ECS)
6. EC2 instance metadata

---

### GUI Equivalent
**IAM → Users** for access keys.  
**IAM → Roles** for role configuration.  
**IAM Identity Center** dashboard for SSO management.  
**EC2 → Instance → Actions → Security → Modify IAM role** for instance profiles.

### Next: [04 — Using the CLI](./04-using-the-cli.md)
