# 05 — S3 Essentials

## Two Command Sets

| Command Set | What it does |
|-------------|-------------|
| `aws s3` | High-level — file-based transfer (cp, sync, mv, rm) |
| `aws s3api` | Low-level — direct S3 API access (create/list/delete buckets, object management) |

## aws s3 — High-Level Commands

### List buckets
```bash
aws s3 ls
```

### List objects in a bucket
```bash
aws s3 ls s3://my-bucket/
aws s3 ls s3://my-bucket/path/to/
```

### Copy files (upload/download/copy between buckets)
```bash
# Upload a file
aws s3 cp myfile.txt s3://my-bucket/

# Download a file
aws s3 cp s3://my-bucket/myfile.txt ./

# Copy between buckets
aws s3 cp s3://source-bucket/file.txt s3://dest-bucket/file.txt

# Upload with recursive (directory)
aws s3 cp myfolder/ s3://my-bucket/ --recursive

# Download with recursive
aws s3 cp s3://my-bucket/ ./myfolder --recursive

# Copy all files matching pattern
aws s3 cp s3://source/ s3://dest/ --recursive --exclude "*.tmp"
```

### Sync (incremental — only newer/missing files)
```bash
# Sync local to S3 (upload only what's new/changed)
aws s3 sync ./local-folder/ s3://my-bucket/

# Sync S3 to local (download only what's new/changed)
aws s3 sync s3://my-bucket/ ./local-folder/

# Sync between buckets
aws s3 sync s3://source-bucket/ s3://dest-bucket/

# Exclude/include patterns
aws s3 sync ./folder s3://my-bucket/ --exclude "*.log" --include "*.txt"
```

### Move files
```bash
aws s3 mv myfile.txt s3://my-bucket/
aws s3 mv s3://source/file.txt s3://dest/file.txt
```

### Delete files
```bash
aws s3 rm s3://my-bucket/file.txt
aws s3 rm s3://my-bucket/ --recursive   # delete everything in a bucket
```

### Presigned URLs (time-limited access)
```bash
# Generate a URL valid for 1 hour (3600 seconds)
aws s3 presign s3://my-bucket/file.txt --expires-in 3600
```

## aws s3api — Low-Level Commands

### Create a bucket
```bash
aws s3api create-bucket --bucket my-unique-name --region us-west-2
```

### List all buckets
```bash
aws s3api list-buckets --query 'Buckets[*].Name'
```

### Bucket versioning
```bash
aws s3api put-bucket-versioning \
  --bucket my-bucket \
  --versioning-configuration Status=Enabled
```

### List object versions
```bash
aws s3api list-object-versions --bucket my-bucket
```

### Set bucket policy
```bash
aws s3api put-bucket-policy \
  --bucket my-bucket \
  --policy file://policy.json
```

### Static website hosting
```bash
aws s3api put-bucket-website \
  --bucket my-bucket \
  --website-configuration '{
    "IndexDocument": {"Suffix": "index.html"},
    "ErrorDocument": {"Key": "error.html"}
  }'
```

## S3 Transfer Optimization Settings

In `~/.aws/config` under `[default]` or per-profile:

```ini
[default]
s3 =
  max_concurrent_requests = 20
  max_queue_size = 10000
  multipart_threshold = 64MB
  multipart_chunksize = 16MB
  max_bandwidth = 100MB/s
```

| Setting | Default | What it does |
|---------|---------|-------------|
| `max_concurrent_requests` | 10 | Parallel uploads/downloads |
| `max_queue_size` | 1000 | Max queued tasks |
| `multipart_threshold` | 8MB | File size to trigger multipart |
| `multipart_chunksize` | 8MB | Per-part size for multipart |
| `max_bandwidth` | none | Throttle bandwidth |

## Common Patterns

```bash
# Upload a website folder
aws s3 sync ./public/ s3://my-website-bucket/ --delete

# Backup a directory to S3 with date folder
aws s3 sync /data/ s3://backups/$(date +%Y-%m-%d)/

# Copy files newer than a date
aws s3 cp s3://bucket/ . --recursive --include "*" --exclude "*" \
  --query 'Contents[?LastModified > `2024-01-01`]'
```

---

### GUI Equivalent
**S3 Console** → Buckets → Upload/Download/Copy/Move/Delete.  
For presigned URLs: right-click object → "Share with a presigned URL".  
For website hosting: Properties → Static website hosting.

### Next: [06 — EC2 Essentials](./06-ec2-essentials.md)
