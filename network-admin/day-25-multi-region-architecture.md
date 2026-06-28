# Day 25 — Multi-Region Architecture

## Concept

Running workloads across **multiple AWS regions** for disaster recovery, lower latency for global users, or data sovereignty compliance.

## Why Multi-Region?

| Reason | Explanation |
|--------|-------------|
| **Disaster Recovery** | Survive a region-wide outage |
| **Latency** | Serve users from the nearest region |
| **Data Residency** | Keep data within specific geographic boundaries |
| **Compliance** | Meet regulatory requirements (GDPR, etc.) |
| **Business Continuity** | No single region of failure |

## Multi-Region Connectivity

### Option 1: VPC Peering (cross-region)

```bash
# Peer VPC in us-east-1 with VPC in eu-west-1
aws ec2 create-vpc-peering-connection \
  --vpc-id vpc-us-east \
  --peer-vpc-id vpc-eu-west \
  --peer-region eu-west-1
```

**Pros**: Simple, no extra cost.  
**Cons**: No transitive routing, manual route tables.

### Option 2: Transit Gateway + TGW Peering

```
TGW (us-east-1) ─── Peering ─── TGW (eu-west-1)
     │                              │
  VPC-A                           VPC-B
  VPC-C                           VPC-D
```

```bash
aws ec2 create-transit-gateway-peering-attachment \
  --transit-gateway-id tgw-us-east-1 \
  --peer-transit-gateway-id tgw-eu-west-1 \
  --peer-region eu-west-1 \
  --peer-account-id 999999999999
```

**Pros**: Transitive routing, centralized management.  
**Cons**: TGW costs in each region.

### Option 3: PrivateLink (cross-region)

Create a VPC Endpoint Service in one region, consumers create endpoints in other regions.

**Pros**: No peering needed, fine-grained access control.  
**Cons**: More complex, per-consumer configuration.

### Option 4: VPN (cross-region)

Connect VPCs in different regions via VPN.

**Pros**: No peering limits.  
**Cons**: Over internet (unless DX in both regions), latency, bandwidth limits.

## Disaster Recovery Strategies

### Backup & Restore (RTO: hours, RPO: 24h)

```
Primary (us-east-1)         DR (eu-west-1)
  S3 Bucket ─── Cross-region replication ───► S3 Bucket
  RDS Snapshot ─── Manual/automated copy ──► RDS Snapshot
```

- Periodically copy data (RDS snapshots, S3 replication)
- In DR event: restore, update DNS
- Lowest cost, highest RTO/RPO

### Pilot Light (RTO: tens of minutes, RPO: minutes)

```
Primary (us-east-1)         DR (eu-west-1)
  Full stack running        Core services running (small)
                            DB replica syncing
                            Other services ready to scale
```

- Run minimal DR stack (DB replica, small compute)
- In DR: scale up DR, update DNS
- Moderate cost

### Warm Standby (RTO: minutes, RPO: seconds)

```
Primary (us-east-1)         DR (eu-west-1)
  Full stack running        Same stack, minimal size
                            DB synchronous replication
```

- DR runs at reduced capacity
- In DR: scale up, redirect traffic
- Higher cost

### Multi-Region Active-Active (RTO: <1 minute, RPO: near-zero)

```
us-east-1 ───┐
              ├── Route 53 Latency-Based Routing
eu-west-1 ───┘
              ├── Global DynamoDB / Aurora Global Database
              └── S3 Cross-Region Replication
```

- Both regions serve traffic simultaneously
- Complex data conflict resolution
- Highest cost, lowest RTO

## Data Replication Strategies

| Service | Cross-Region Replication | RPO |
|---------|-------------------------|-----|
| **S3** | S3 Cross-Region Replication (CRR) | 15 minutes |
| **RDS** | Cross-region read replica → promote | Seconds |
| **Aurora** | Aurora Global Database | <1 second |
| **DynamoDB** | Global Tables | <1 second |
| **Redshift** | Cross-region snapshot copy | Hours |
| **ElastiCache** | Global Datastore | 1 second |
| **SQS** | Manual — drain queue | N/A |

## DNS Multi-Region with Route 53

### Failover Routing (Active-Passive)

```bash
aws route53 change-resource-record-sets \
  --hosted-zone-id Z1234567890 \
  --change-batch '{
    "Changes": [
      {"Action": "UPSERT","ResourceRecordSet": {
        "Name": "app.example.com","Type": "A",
        "SetIdentifier": "Primary","Failover": "PRIMARY",
        "AliasTarget": {"HostedZoneId": "Z35SXDOTRQ7X7K","DNSName": "us-east-alb.elb.amazonaws.com","EvaluateTargetHealth": true}
      }},
      {"Action": "UPSERT","ResourceRecordSet": {
        "Name": "app.example.com","Type": "A",
        "SetIdentifier": "Secondary","Failover": "SECONDARY",
        "AliasTarget": {"HostedZoneId": "Z...","DNSName": "eu-west-alb.elb.amazonaws.com","EvaluateTargetHealth": true}
      }}
    ]
  }'
```

### Latency-Based Routing (Active-Active)

```bash
aws route53 change-resource-record-sets \
  --hosted-zone-id Z1234567890 \
  --change-batch '{
    "Changes": [
      {"Action": "UPSERT","ResourceRecordSet": {
        "Name": "app.example.com","Type": "A",
        "SetIdentifier": "NA","Region": "us-east-1",
        "AliasTarget": {"HostedZoneId": "Z35SXDOTRQ7X7K","DNSName": "na-alb.elb.amazonaws.com","EvaluateTargetHealth": true}
      }},
      {"Action": "UPSERT","ResourceRecordSet": {
        "Name": "app.example.com","Type": "A",
        "SetIdentifier": "EU","Region": "eu-west-1",
        "AliasTarget": {"HostedZoneId": "Z...","DNSName": "eu-alb.elb.amazonaws.com","EvaluateTargetHealth": true}
      }}
    ]
  }'
```

## Cross-Region Security

### VPC Peering Encryption
Cross-region VPC peering traffic is automatically encrypted.

### VPN
IPSec encryption built in.

### PrivateLink
Traffic stays on AWS backbone.

## Global Accelerator for Multi-Region

```bash
aws globalaccelerator create-accelerator --name "global-app"
```

- Anycast IPs — users hit nearest edge
- Two endpoint groups (us-east-1, eu-west-1) with traffic dials
- Automatic failover (< 1 second)

## Cost Considerations

| Component | Multi-Region Cost Impact |
|-----------|------------------------|
| **Data transfer** | $0.01-0.02/GB between regions |
| **Compute** | 2x (running in 2 regions) |
| **Storage** | 2x + replication |
| **ELB** | Per-region costs |
| **Route 53** | Health checks + queries |
| **TGW peering** | $0.01/GB processed |

## CLI Commands

```bash
# Cross-region peering
aws ec2 create-vpc-peering-connection --peer-region eu-west-1

# TGW peering
aws ec2 create-transit-gateway-peering-attachment

# Global Accelerator
aws globalaccelerator create-accelerator

# S3 CRR
aws s3api put-bucket-replication --bucket source --replication-configuration file://crr.json
```

## Gotchas

- **Cross-region latency**: 10-100ms+ depending on regions
- **Data transfer costs**: not free between regions
- **Global services**: Route 53, CloudFront, IAM are global — no need for multi-region setup
- **Service limits**: some services aren't available in all regions
- **Data sovereignty**: some data can't leave certain countries
- **Split-brain**: in Active-Active, beware of conflicting writes — use DynamoDB Global Tables or Aurora Global Database
- **Failover testing**: test DR quarterly — scripts, DNS updates, data validation
