# Day 26 — VPC Flow Logs

## Concept

VPC Flow Logs capture **IP traffic metadata** flowing in and out of network interfaces in your VPC. They record who talked to whom, on which port, what protocol, and whether it was accepted or rejected.

## What Flow Logs Capture

| Field | Example | Description |
|-------|---------|-------------|
| `version` | 2 | Flow log format version |
| `account-id` | 123456789012 | AWS account |
| `interface-id` | eni-12345678 | ENI that traffic passed through |
| `srcaddr` | 10.0.1.50 | Source IP |
| `dstaddr` | 203.0.113.50 | Destination IP |
| `srcport` | 54321 | Source port |
| `dstport` | 443 | Destination port |
| `protocol` | 6 | 6=TCP, 17=UDP, 1=ICMP |
| `packets` | 10 | Number of packets in the flow |
| `bytes` | 1200 | Number of bytes |
| `start` | 1625097600 | Start time (Unix epoch) |
| `end` | 1625097900 | End time |
| `action` | ACCEPT / REJECT | Whether SG/NACL allowed or blocked |
| `log-status` | OK / NODATA / SKIPDATA | Log delivery status |

### Sample Flow Log Entry

```
2 123456789012 eni-12345678 10.0.1.50 203.0.113.50 54321 443 6 10 1200 1625097600 1625097900 ACCEPT OK
```

## Levels

| Level | Captures |
|-------|----------|
| **VPC** | Traffic for all ENIs in the VPC |
| **Subnet** | Traffic for all ENIs in the subnet |
| **ENI** | Traffic for a specific ENI |

## Create Flow Logs

### To S3

```bash
aws ec2 create-flow-logs \
  --resource-type VPC \
  --resource-ids vpc-12345678 \
  --log-destination-type s3 \
  --log-destination arn:aws:s3:::my-flow-log-bucket \
  --traffic-type ALL \         # ALL, ACCEPT, or REJECT
  --max-aggregation-interval 60    # 60 or 600 seconds
```

### To CloudWatch Logs

```bash
# Create log group first
aws logs create-log-group --log-group-name vpc-flow-logs

# Create flow logs
aws ec2 create-flow-logs \
  --resource-type VPC \
  --resource-ids vpc-12345678 \
  --log-destination-type cloud-watch-logs \
  --log-group-name vpc-flow-logs \
  --traffic-type ALL \
  --max-aggregation-interval 60
```

### Custom Format

Instead of the default format, specify exact fields:

```bash
aws ec2 create-flow-logs \
  --resource-type VPC \
  --resource-ids vpc-12345678 \
  --log-destination-type s3 \
  --log-destination arn:aws:s3:::my-flow-log-bucket \
  --traffic-type ALL \
  --log-format '${version} ${account-id} ${interface-id} ${srcaddr} ${dstaddr} ${srcport} ${dstport} ${protocol} ${packets} ${bytes} ${start} ${end} ${action} ${log-status} ${vpc-id} ${subnet-id} ${instance-id} ${tcp-flags} ${type} ${pkt-srcaddr} ${pkt-dstaddr}'
```

Available custom fields: `vpc-id`, `subnet-id`, `instance-id`, `tcp-flags`, `type`, `pkt-srcaddr`, `pkt-dstaddr`, `region`, `az-id`, `sublocation-type`, `sublocation-id`

## Analyzing Flow Logs

### With AWS CLI (basic)

```bash
# List flow logs
aws ec2 describe-flow-logs

# S3 — get latest log
LATEST=$(aws s3 ls s3://my-flow-log-bucket/AWSLogs/123456789012/vpcflowlogs/us-east-1/ --recursive | sort | tail -1 | awk '{print $4}')
aws s3 cp s3://my-flow-log-bucket/$LATEST - | gunzip | head -20
```

### With Amazon Athena

1. Create an Athena table:

```sql
CREATE EXTERNAL TABLE vpc_flow_logs (
  version int,
  account string,
  interfaceid string,
  sourceaddress string,
  destinationaddress string,
  sourceport int,
  destinationport int,
  protocol int,
  numpackets int,
  numbytes bigint,
  starttime int,
  endtime int,
  action string,
  logstatus string
)
ROW FORMAT SERDE 'org.apache.hadoop.hive.serde2.lazy.LazySimpleSerDe'
LOCATION 's3://my-flow-log-bucket/AWSLogs/{account_id}/vpcflowlogs/{region}/'
TBLPROPERTIES ('skip.header.line.count'='1');
```

2. Useful queries:

```sql
-- Top talkers by bytes
SELECT sourceaddress, destinationaddress, SUM(numbytes) as total_bytes
FROM vpc_flow_logs
WHERE action = 'ACCEPT'
GROUP BY sourceaddress, destinationaddress
ORDER BY total_bytes DESC
LIMIT 20;

-- Rejected traffic (troubleshooting)
SELECT sourceaddress, destinationaddress, destinationport, COUNT(*) as attempts
FROM vpc_flow_logs
WHERE action = 'REJECT'
GROUP BY sourceaddress, destinationaddress, destinationport
ORDER BY attempts DESC;

-- SSH brute force attempts
SELECT sourceaddress, COUNT(*) as attempts
FROM vpc_flow_logs
WHERE destinationport = 22 AND action = 'REJECT'
GROUP BY sourceaddress
ORDER BY attempts DESC;

-- Traffic to specific IP
SELECT * FROM vpc_flow_logs
WHERE destinationaddress = '10.0.1.50'
ORDER BY starttime DESC;

-- Connection count by protocol
SELECT
  CASE protocol
    WHEN 6 THEN 'TCP'
    WHEN 17 THEN 'UDP'
    WHEN 1 THEN 'ICMP'
    ELSE 'OTHER'
  END as protocol_name,
  COUNT(*) as connections
FROM vpc_flow_logs
GROUP BY protocol;
```

### With CloudWatch Logs Insights

```bash
# Query in CloudWatch Logs Insights
fields @timestamp, srcaddr, dstaddr, dstport, action, bytes
| filter action = "REJECT"
| stats count() by dstaddr, dstport
| sort count desc
| limit 20
```

## Troubleshooting with Flow Logs

| Problem | Flow Log Pattern |
|---------|-----------------|
| Can't SSH to instance | `REJECT` on port 22 from your IP |
| Database connection refused | `REJECT` on port 3306/5432 |
| No internet access | `ACCEPT` on 0.0.0.0/0 but check NAT/IGW |
| Asymmetric routing | Packets go out via one path, return via another |
| Latency issues | High byte counts, packet loss |
| Security group too restrictive | REJECT followed by no ACCEPT |
| NACL blocking traffic | REJECT (NACLs have no log — flow logs show it) |

## Pricing

| Destination | Cost |
|-------------|------|
| S3 | Free to publish, S3 storage costs apply |
| CloudWatch Logs | $0.50/GB ingested, $0.03/GB storage |

## CLI Commands

```bash
aws ec2 create-flow-logs
aws ec2 describe-flow-logs
aws ec2 delete-flow-logs
```

## Console

**VPC → Your VPCs → Select VPC → Flow Logs tab** — create and view
**VPC → Subnets → Select subnet → Flow Logs tab**
**VPC → Network Interfaces → Select ENI → Flow Logs tab**

## Gotchas

- **Flow logs are not real-time** — aggregation interval is 60s or 600s
- **Does not capture all traffic**: traffic to Amazon DNS, Windows license activation, and Amazon metadata (169.254.169.254) may not be fully captured
- **S3 prefix structure**: `AWSLogs/{account-id}/vpcflowlogs/{region}/{yyyy}/{mm}/{dd}/`
- **Logs are GZIP compressed** in S3
- **Cannot enable flow logs on VPCs that are peered** directly — each VPC has its own flow logs
- **There is no charge for publishing to S3** but CloudWatch Logs ingestion costs
- **Flow logs capture metadata only** — not payload content
- **You cannot enable flow logs on a VPC that's shared via RAM** (the consumer account can't)
- **Aggregation interval** of 60 seconds gives ~10-minute delivery delay; 600 seconds gives ~30-minute delay
