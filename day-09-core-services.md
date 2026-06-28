# Day 9 — Core AWS Services for Network Admins

## Concept

Beyond the pure networking services, a network admin must understand how **key AWS services** interact with the network — S3, EBS, RDS, CloudWatch, and CloudTrail. These services generate traffic, have network implications, and are essential for monitoring.

---

## S3 (Simple Storage Service)

Object storage — stores files as objects in buckets. Reachable via internet or VPC Endpoints.

### S3 Networking Essentials

| Feature | Network Detail |
|---------|---------------|
| **Access via internet** | S3 public endpoint: `s3.us-east-1.amazonaws.com` |
| **Access via VPC** | Gateway Endpoint (free) or Interface Endpoint (paid) |
| **Access via DX/VPN** | Gateway Endpoint works (VPC only); Interface Endpoint for on-prem |
| **Bandwidth** | Automatically scales to thousands of requests/sec |
| **Transfer acceleration** | Uses CloudFront edge for faster uploads |
| **S3 Transfer Acceleration** | `s3-accelerate.amazonaws.com` |

### Bucket Policies vs IAM

S3 access control can be at the user level (IAM) or bucket level (policy). For network admins, **bucket policies** are important for restricting access to specific VPCs or endpoints.

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Deny",
    "Principal": "*",
    "Action": "s3:*",
    "Resource": "arn:aws:s3:::my-bucket/*",
    "Condition": {
      "StringNotEquals": {
        "aws:SourceVpce": "vpce-xxxxxxxx"
      }
    }
  }]
}
```

### CLI Commands

```bash
# List buckets
aws s3 ls

# Upload
aws s3 cp file.txt s3://my-bucket/

# Download
aws s3 cp s3://my-bucket/file.txt ./

# Sync directory
aws s3 sync ./folder s3://my-bucket/ --delete

# Generate presigned URL (time-limited access)
aws s3 presign s3://my-bucket/file.txt --expires-in 3600

# Enable versioning
aws s3api put-bucket-versioning \
  --bucket my-bucket \
  --versioning-configuration Status=Enabled
```

### S3 Transfer Acceleration

Faster uploads by routing traffic through CloudFront edge locations:

```bash
aws s3 configure s3-accelerate   # enable in config
# Or per command:
aws s3 cp largefile.dat s3://my-bucket/ --s3-accelerate
```

---

## EBS (Elastic Block Store)

Block storage volumes for EC2 instances. Think: **network-attached virtual hard drives**.

### Volume Types

| Type | Max IOPS | Max Throughput | Use Case |
|------|----------|---------------|----------|
| **gp3** | 16,000 | 1,000 MB/s | General purpose (default) |
| **gp2** | 16,000 | 250 MB/s | Legacy general purpose |
| **io1** | 64,000 | 1,000 MB/s | High-performance databases |
| **io2** | 256,000 | 4,000 MB/s | Mission-critical, highest durability |
| **st1** | 500 | 500 MB/s | Throughput-optimized (HDD) |
| **sc1** | 250 | 250 MB/s | Cold storage (HDD) |

### EBS and the Network

- EBS traffic goes over the **network**, separate from data traffic
- Uses the same ENI but a **different VLAN**
- Traffic stays within the AZ — **cannot attach EBS across AZs**
- EBS-optimized instances have dedicated bandwidth for EBS
- **Nitrix instances** have EBS bandwidth separate from data bandwidth

```bash
# Create a gp3 volume
aws ec2 create-volume \
  --volume-type gp3 \
  --size 100 \
  --availability-zone us-east-1a

# Attach to instance
aws ec2 attach-volume \
  --volume-id vol-12345678 \
  --instance-id i-12345678 \
  --device /dev/xvdf

# Create snapshot (for backup / migration)
aws ec2 create-snapshot \
  --volume-id vol-12345678 \
  --description "Daily backup"

# List snapshots
aws ec2 describe-snapshots --owner-ids self
```

### EBS Snapshot Networking

Snapshots are stored in S3. Transferring snapshots between regions uses inter-region data transfer ($0.02/GB):

```bash
# Copy snapshot cross-region
aws ec2 copy-snapshot \
  --source-region us-east-1 \
  --source-snapshot-id snap-xxx \
  --destination-region eu-west-1 \
  --description "DR copy"
```

---

## RDS (Relational Database Service)

Managed databases — MySQL, PostgreSQL, MariaDB, Oracle, SQL Server, Aurora.

### RDS Networking

- Runs in a **private subnet** (never public)
- Accessed via **DNS name** (CNAME) that points to the instance or reader endpoint
- **Multi-AZ**: synchronous replication across AZs (latency impact)
- **Read replicas**: can be cross-region (incurs data transfer costs)
- **Proxy**: RDS Proxy sits between app and DB — connection pooling, faster failover

```bash
# Create RDS in private subnets
aws rds create-db-instance \
  --db-instance-identifier mydb \
  --db-instance-class db.t3.medium \
  --engine postgres \
  --master-username admin \
  --master-user-password ChangeMe123! \
  --vpc-security-group-ids sg-database \
  --db-subnet-group-name my-db-subnets \
  --multi-az \
  --storage-type gp3 \
  --allocated-storage 100
```

### DNS Endpoints

| Endpoint | Purpose |
|----------|---------|
| **CNAME** (instance) | Primary — points to current primary (Multi-AZ) |
| **Read Replica** | Individual endpoints for each read replica |
| **Cluster** (Aurora) | Writer + Reader endpoints |
| **RDS Proxy** | Proxy endpoint for connection pooling |

### Security Group Configuration

```bash
# Allow app servers to access the DB
aws ec2 authorize-security-group-ingress \
  --group-id sg-database \
  --protocol tcp \
  --port 5432 \
  --source-group sg-app-server
```

---

## CloudWatch

Monitoring service for AWS resources and applications.

### What Network Admins Monitor

| Metric | Where | What It Tells You |
|--------|-------|-------------------|
| `NetworkIn` | EC2 | Inbound traffic (bytes) |
| `NetworkOut` | EC2 | Outbound traffic (bytes) |
| `NetworkPacketsIn` | EC2 | Inbound packet count |
| `NetworkPacketsOut` | EC2 | Outbound packet count |
| `ActiveConnectionCount` | NLB | Current connections |
| `NewConnectionCount` | ALB/NLB | New connections per second |
| `RejectedConnectionCount` | ALB | Connections rejected (over limit) |
| `BytesProcessed` | NAT Gateway | Data processed (billing) |
| `TunnelState` | VPN | VPN tunnel health |
| `PacketDropCount` | Network Firewall | Packets dropped |

```bash
# Get EC2 network metrics
aws cloudwatch get-metric-statistics \
  --namespace AWS/EC2 \
  --metric-name NetworkOut \
  --dimensions Name=InstanceId,Value=i-12345678 \
  --start-time $(date -d '1 hour ago') --end-time $(date +%s) \
  --period 300 \
  --statistics Sum

# Create alarm for high network out
aws cloudwatch put-metric-alarm \
  --alarm-name HighNetworkOut \
  --alarm-description "Alarm if outbound traffic exceeds 1GB in 5 minutes" \
  --metric-name NetworkOut \
  --namespace AWS/EC2 \
  --statistic Sum \
  --period 300 \
  --threshold 1073741824 \
  --comparison-operator GreaterThanThreshold \
  --evaluation-periods 2 \
  --dimensions Name=InstanceId,Value=i-12345678 \
  --alarm-actions arn:aws:sns:us-east-1:xxx:network-alerts
```

### Logs (CloudWatch Logs)

Send application/system logs to CloudWatch:

```bash
# Create log group
aws logs create-log-group --log-group-name /var/log/syslog

# Create log stream
aws logs create-log-stream \
  --log-group-name /var/log/syslog \
  --log-stream-name ip-10-0-1-50

# Put log events
aws logs put-log-events \
  --log-group-name /var/log/syslog \
  --log-stream-name ip-10-0-1-50 \
  --log-events timestamp=$(date +%s%3N),message="User connected from 203.0.113.50"
```

### Logs Insights (query logs)

```bash
# In CloudWatch Console → Logs Insights
fields @timestamp, @message
| filter @message like /ssh.*Failed password/
| stats count() by srcAddr
| sort count desc
| limit 20
```

### Contributor Insights

Built-in rules to detect top contributors from VPC Flow Logs:

- Top-N IP addresses by traffic
- Top-N port usage
- Top-N rejected connections

---

## CloudTrail

Audit API activity in your account. Every API call is logged.

### For Network Admins, Monitor:

| Event | What It Means |
|-------|---------------|
| `AuthorizeSecurityGroupIngress` | Someone opened a port |
| `CreateSecurityGroup` | New SG created |
| `CreateVpc` | New VPC created |
| `AttachInternetGateway` | Making a VPC public |
| `ModifyVpcAttribute` | Changing DNS settings |
| `CreateVpcPeeringConnection` | Cross-VPC connection |
| `DeleteFlowLogs` | Someone turned off flow logs |

```bash
# Create a trail
aws cloudtrail create-trail \
  --name network-audit-trail \
  --s3-bucket-name my-cloudtrail-logs \
  --is-multi-region-trail \
  --include-global-service-events

# Look up events
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=EventName,AttributeValue=AuthorizeSecurityGroupIngress \
  --start-time $(date -d '24 hours ago') --end-time $(date +%s)
```

### CloudTrail Insights

Detects unusual API activity (e.g., unusually high rate of `AuthorizeSecurityGroupIngress` calls):

```bash
aws cloudtrail put-insight-selectors \
  --trail-arn arn:aws:cloudtrail:us-east-1:xxx:trail/network-trail \
  --insight-selectors InsightType=ApiCallRate
```

---

## CLI Commands Summary

```bash
# S3
aws s3 ls / cp / mv / sync / rm / presign
aws s3api create-bucket / put-bucket-policy

# EBS
aws ec2 create-volume / attach-volume / delete-volume
aws ec2 create-snapshot / copy-snapshot / describe-snapshots

# RDS
aws rds create-db-instance / describe-db-instances / delete-db-instance
aws rds create-db-snapshot / describe-db-snapshots

# CloudWatch
aws cloudwatch get-metric-statistics
aws cloudwatch put-metric-alarm
aws cloudwatch describe-alarms
aws logs create-log-group / put-log-events

# CloudTrail
aws cloudtrail create-trail
aws cloudtrail lookup-events
aws cloudtrail put-insight-selectors
```

## Gotchas

- **S3 Gateway Endpoint is free** — always use it for VPC-to-S3 traffic
- **EBS snapshots to S3** — no additional data transfer cost within same region
- **RDS Multi-AZ replication** is synchronous — appens within same region, ~1ms latency
- **RDS cross-region read replicas** incur data transfer costs
- **CloudWatch metrics** for EC2 network: `Sum` for bytes, `Average` for packets
- **CloudWatch Logs** costs: ~$0.50/GB ingested + $0.03/GB storage
- **CloudTrail** costs: $0.10/100,000 events (first copy free to S3)
- **Don't put RDS in public subnets** — always use private subnets + security groups
