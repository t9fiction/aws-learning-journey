# 08 — Productivity: Auto-Prompt, Aliases, Wizards & Completion

## Auto-Prompt (Interactive Mode)

Auto-prompt turns the CLI into an **interactive shell** — suggests commands, shows parameters, and auto-completes as you type.

```bash
# Enable it
aws configure set cli_auto_prompt on
```

**Two modes:**

| Mode | Behavior |
|------|----------|
| `on` | Interactive — shows parameter prompts & completions |
| `enabled` | Same as `on` but can be toggled at runtime |
| `off` | Standard CLI mode (default) |

**Toggle at runtime** (when in `enabled` mode):
```
# Press Ctrl+E to toggle auto-prompt on/off
```

**Example session with auto-prompt:**
```
$ aws ec2 describe-instances
> instance-ids: i-123
> --filters: Name=tag:Name,Values=web
> --max-items: 10
```

## Command Completion (bash/zsh)

Tab-completion for commands, subcommands, and parameters.

### Linux/macOS (bash)
```bash
# Install completion (one-time)
complete -C "$(which aws_completer)" aws

# Add to ~/.bashrc for persistence
echo 'complete -C "$(which aws_completer)" aws' >> ~/.bashrc
```

### zsh
```bash
# Add to ~/.zshrc
echo "source $(dirname $(which aws_completer))/aws_zsh_completer.sh" >> ~/.zshrc
```

### Windows (PowerShell)
```powershell
# Add to your PowerShell profile
Add-Content $PROFILE "`nRegister-ArgumentCompleter -Native -CommandName aws -ScriptReference aws_completer"
```

## Aliases

Create shortcuts for long commands.

### Step 1: Create the alias file
```bash
mkdir -p ~/.aws/cli
touch ~/.aws/cli/alias
```

### Step 2: Add aliases
```ini
[toplevel]

# List all EC2 instances
ec2list = ec2 describe-instances

# List S3 buckets
s3ls = s3 ls

# Get the caller identity (who am I?)
whoami = sts get-caller-identity

# Launch a t2.micro instance
launch-micro = ec2 run-instances --image-id ami-0abcdef123 --instance-type t2.micro --key-name MyKey

# List IAM users as a table
iamusers = iam list-users --output table

# Tail CloudWatch logs
cw-tail = logs tail --follow
```

### Step 3: Use aliases
```bash
aws ec2list
aws whoami
aws iamusers
```

**Note**: Aliases are **toplevel only** — they replace the entire command after `aws`. You can still pass additional args:
```bash
aws ec2list --query 'Reservations[*].Instances[*].InstanceId'
```

## Wizards

Wizards provide step-by-step guided configuration.

```bash
# Configure SSO step by step
aws configure sso

# Configure only the sso-session
aws configure sso-session
```

Wizards prompt you for each value interactively.

## Disable the Pager

By default, long output goes through `less`. Disable it:
```bash
# Per-session
export AWS_PAGER=""

# Permanent
aws configure set cli_pager cat

# Or just use
aws s3 ls --no-cli-pager
```

## Useful Profile Tricks

```bash
# Set a different region per command
aws ec2 describe-instances --region eu-west-1

# Override output format for a command
aws iam list-users --output table

# Use a specific profile
aws s3 ls --profile prod

# Set default profile for terminal session
export AWS_PROFILE=prod
```

---

### GUI Equivalent
**AWS Management Console** — no direct equivalent for these CLI productivity features. The Console is point-and-click by design. These CLI features make the command line faster than the Console.

### Next: [09 — Advanced Topics](./09-advanced.md)
