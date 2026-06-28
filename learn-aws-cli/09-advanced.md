# 09 — Advanced: Retries, Endpoints, Proxies & Pagination

## Retries

Three retry modes:

| Mode | Behavior | Use Case |
|------|----------|----------|
| `legacy` | Default (v1 behavior) | Backward compatibility |
| `standard` | Exponential backoff + jitter | General purpose (recommended) |
| `adaptive` | Standard + client-side rate limiting | High-throughput, bursty workloads |

```bash
# Configure in ~/.aws/config
aws configure set retry_mode standard
aws configure set max_attempts 5

# Or per profile
[profile prod]
retry_mode = adaptive
max_attempts = 10
```

**View retry logs:**
```bash
aws s3 ls --debug 2>&1 | grep -i retry
```

## Endpoints

By default, AWS CLI uses the standard public endpoints. Override for testing or custom setups.

### Single command
```bash
aws s3 list-buckets --endpoint-url http://localhost:4566
```

### Global setting
```bash
aws configure set endpoint_url http://localhost:4566
```

### Service-specific (in config)
```ini
[default]
services = my-endpoints

[services my-endpoints]
s3 =
  endpoint_url = https://s3.custom-endpoint.com
dynamodb =
  endpoint_url = https://dynamodb.custom-endpoint.com
```

### FIPS endpoints
```bash
aws configure set use_fips_endpoint true
```

### Dual-stack (IPv4 + IPv6)
```bash
aws configure set use_dualstack_endpoint true
```

## HTTP Proxies

```bash
# Environment variables
export HTTP_PROXY=http://proxy.example.com:8080
export HTTPS_PROXY=http://proxy.example.com:8080
export NO_PROXY=169.254.169.254   # skip proxy for metadata
```

**Authenticated proxies:**
```bash
export HTTP_PROXY=http://username:password@proxy.example.com:8080
```

## Pagination Control

```bash
# Page size (API calls per page)
aws dynamodb scan --table-name MyTable --page-size 100

# Max items (total results returned)
aws s3api list-objects --bucket my-bucket --max-items 1000

# Manual pagination in scripts
next_token=""
while true; do
  result=$(aws ec2 describe-instances \
    --max-items 100 \
    --starting-token "$next_token")
  
  # Process result...
  
  next_token=$(echo "$result" | jq -r '.NextToken')
  [ "$next_token" = "null" ] && break
done
```

## Filtering with `--query` (JMESPath)

Advanced JMESPath expressions:

```bash
# Filter by value
aws ec2 describe-instances \
  --query 'Reservations[*].Instances[?State.Name==`running`].InstanceId'

# Sort by field
aws ec2 describe-instances \
  --query 'Reservations[*].Instances[*] | sort_by(@, &LaunchTime)[].[InstanceId,LaunchTime]'

# Get count
aws ec2 describe-instances \
  --query 'length(Reservations[*].Instances[*])'

# Projections with pipe
aws ec2 describe-instances \
  --query 'Reservations[].Instances[].[InstanceId,InstanceType,Tags[?Key==`Name`].Value|[0]]' \
  --output table
```

## jq — Better JSON Processing

Pipe CLI output to `jq` for complex queries:

```bash
# Pretty-print specific fields
aws ec2 describe-instances | jq '.Reservations[].Instances[] | {id: .InstanceId, type: .InstanceType}'

# Filter by condition
aws ec2 describe-instances | jq '[.Reservations[].Instances[] | select(.State.Name == "running")]'

# Group by type
aws ec2 describe-instances | jq '[.Reservations[].Instances[] | group_by(.InstanceType)[] | {type: .[0].InstanceType, count: length}]'
```

---

### GUI Equivalent
**AWS Console** — no pagination control needed (infinite scroll). Retries are automatic. Endpoints are not configurable. These are CLI/API-specific features.

### Next: [10 — Troubleshooting](./10-troubleshooting.md)
