# 02 — Configuration & Setup

## Quick Setup — `aws configure`

The fastest way to get started:

```bash
$ aws configure
AWS Access Key ID [None]: AKIAIOSFODNN7EXAMPLE
AWS Secret Access Key [None]: wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
Default region name [None]: us-west-2
Default output format [None]: json
```

This creates two files:

- **`~/.aws/config`** — stores region, output format, profiles
- **`~/.aws/credentials`** — stores access keys (secret!)

## Configuration File (`~/.aws/config`)

```ini
[default]
region = us-west-2
output = json

[profile prod]
region = eu-west-1
output = json
```

## Credentials File (`~/.aws/credentials`)

```ini
[default]
aws_access_key_id = AKIAIOSFODNN7EXAMPLE
aws_secret_access_key = wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY

[prod]
aws_access_key_id = AKIA...EXAMPLE
aws_secret_access_key = ...EXAMPLE
```

## Named Profiles

Switch between different accounts/roles:

```bash
# Run a command using a specific profile
aws s3 ls --profile prod

# Set a profile for a session
export AWS_PROFILE=prod
```

## Environment Variables

Override config without editing files:

```bash
export AWS_ACCESS_KEY_ID=AKIA...EXAMPLE
export AWS_SECRET_ACCESS_KEY=...EXAMPLE
export AWS_DEFAULT_REGION=us-west-2
export AWS_DEFAULT_OUTPUT=json
export AWS_PROFILE=myprofile   # or use a named profile
```

## Credential Precedence (Order Matters)

AWS checks credentials in this order (first found wins):

1. **Command-line options** (`--region`, `--profile`, etc.)
2. **Environment variables** (`AWS_ACCESS_KEY_ID`, etc.)
3. **Shared credentials file** (`~/.aws/credentials`)
4. **AWS config file** (`~/.aws/config`)
5. **Container credentials** (ECS task roles)
6. **EC2 instance metadata** (IAM roles on EC2)

> **Rule of thumb**: Command flags > env vars > files > instance metadata.

## Supported Config File Settings

| Setting | What it does |
|---------|-------------|
| `region` | Default AWS Region |
| `output` | Default output format (json, yaml, text, table) |
| `cli_pager` | Pager program (e.g., `less`, or `cat` to disable) |
| `cli_auto_prompt` | Enable auto-prompt mode (`on`, `off`, `enabled`) |
| `max_attempts` | Max retry attempts (default: 5) |
| `retry_mode` | Retry strategy (`legacy`, `standard`, `adaptive`) |
| `s3.max_concurrent_requests` | Max concurrent S3 transfers (default: 10) |
| `s3.max_queue_size` | Max S3 task queue size (default: 1000) |

Example:
```ini
[default]
region = us-west-2
output = json
cli_pager = cat
max_attempts = 10
retry_mode = standard
```

## Command-line Options

Global options you can pass to any command:

| Flag | Alias | Purpose |
|------|-------|---------|
| `--region` | | AWS Region |
| `--output` | | Output format (`json`, `yaml`, `text`, `table`) |
| `--profile` | | Named profile |
| `--debug` | | Debug logging |
| `--endpoint-url` | | Custom service endpoint |
| `--no-paginate` | | Disable pagination |
| `--cli-read-timeout` | | Read timeout in seconds |
| `--cli-connect-timeout` | | Connect timeout in seconds |

---

### GUI Equivalent
**AWS Management Console** → **IAM → Users → Security credentials** to create access keys.  
**Region selector** in the top-right of the Console.

### Next: [03 — Authentication](./03-authentication.md)
