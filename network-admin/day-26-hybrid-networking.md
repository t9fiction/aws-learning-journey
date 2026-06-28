# Day 26 — Hybrid Networking

## Concept

Hybrid networking connects your **on-premises data center** with your **AWS VPCs** — combining VPN, Direct Connect, and Transit Gateway to create a seamless network.

## Why Hybrid?

| Use Case | Why |
|----------|-----|
| **Migration** | Gradual lift-and-shift — keep on-prem while migrating |
| **Legacy Apps** | Apps that can't move to the cloud yet |
| **Compliance** | Data must remain on-prem for regulatory reasons |
| **Hybrid Workforce** | Users need access to both on-prem and cloud resources |
| **Disaster Recovery** | On-prem as DR for cloud (or vice versa) |

## Reference Architecture

```
                      ┌─────────────────────────────┐
                      │      AWS Cloud              │
                      │  ┌───────────────────────┐  │
                      │  │   Transit Gateway      │  │
                      │  └──┬─────┬──────┬───────┘  │
                      │     │     │      │          │
                      │  ┌──┴┐ ┌──┴──┐ ┌┴──────┐   │
                      │  │VPC│ │VPC2 │ │Shared │   │
                      │  │ A │ │  B  │ │ Svc   │   │
                      │  └───┘ └─────┘ └───────┘   │
                      └──────────┬──────────────────┘
                                 │
    On-Prem              ┌───────┴────────┐
    ┌──────────────┐     │  Direct Connect │     ┌──────────────┐
    │  Data Center  │─────┤  + VPN Backup   ├─────│  Branch      │
    │  10.0.0.0/8  │     └────────────────┘     │  Office       │
    └──────────────┘                              │  192.168.0/24 │
                                                  └──────────────┘
```

## Connectivity Options

| Connection | Bandwidth | Latency | Cost | Best For |
|-----------|-----------|---------|------|----------|
| **Site-to-Site VPN** | Up to 1.25 Gbps (ECMP) | 10-50ms | Low | Backup, small sites |
| **Direct Connect** | 50Mbps - 100Gbps | 1-5ms | Medium-High | Primary, high-volume |
| **Direct Connect + VPN** | DX primary, VPN failover | Low | Medium | Production HA |
| **Client VPN** | Per-user | Depends | Per-connection | Remote users |

## Primary + Backup (DX + VPN)

The most common production pattern:

```
Primary: Direct Connect (active)
Backup:  Site-to-Site VPN (standby, used if DX fails)
```

### How It Works

- DX has a better BGP prefix (more specific)
- If DX fails, VPN route takes over
- Use BGP communities to influence routing

### Setup

```bash
# 1. Direct Connect with VGW
DX_VGW=$(aws ec2 create-vpn-gateway --type ipsec.1 --query 'VpnGateway.VpnGatewayId' --output text)
aws ec2 attach-vpn-gateway --vpn-gateway-id $DX_VGW --vpc-id vpc-12345678

# 2. VPN Connection on the same VGW (backup)
aws ec2 create-vpn-connection \
  --type ipsec.1 \
  --customer-gateway-id cgw-xxx \
  --vpn-gateway-id $DX_VGW \
  --options "{\"EnableAcceleration\": true}"

# 3. Enable route propagation
aws ec2 enable-vgw-route-propagation \
  --route-table-id rtb-private --gateway-id $DX_VGW
```

## TGW as Central Hub

Transit Gateway is the **centerpiece** of hybrid networking:

```
On-prem ── DX ── TGW ── VPC-Prod
              │    └── VPC-Dev
              │    └── VPC-Shared
              │
              └── VPN (backup)

TGW Route Table:
  10.0.0.0/8 → VPC attachments
  192.168.0.0/16 → VPN/BGP-learned
  0.0.0.0/0 → Drop (no internet from TGW)
  More specific routes → Override as needed
```

## BGP Routing in Hybrid Environments

### BGP Best Practices

| Practice | Why |
|----------|-----|
| **Use private ASN** (64512-65534) | Don't conflict with public BGP |
| **Advertise specific prefixes** | /24 or more specific for on-prem CIDRs |
| **Use BGP communities** | AWS uses communities to identify route source |
| **Prefix limits** | Set limits to prevent route table explosion |
| **MD5 authentication** | Secure BGP sessions |
| **Graceful restart** | Avoid routing flaps |

### AWS BGP Communities

| Community | Meaning |
|-----------|---------|
| `7224:7100` | Route from DX |
| `7224:7200` | Route from VPN |
| `7224:7300` | Route from TGW |

## Route Propagation

### With VGW

```bash
# VPC route table learns on-prem CIDRs automatically
aws ec2 enable-vgw-route-propagation \
  --route-table-id rtb-private \
  --gateway-id vgw-xxx
```

### With TGW

```bash
# TGW route table learns from attachments
aws ec2 enable-transit-gateway-route-table-propagation \
  --transit-gateway-route-table-id tgw-rtb-xxx \
  --transit-gateway-attachment-id tgw-attach-xxx
```

## DNS in Hybrid Environments

With Route 53 Resolver (Day 23):

```
On-prem DNS ─── Inbound Resolver Endpoint ─── VPC DNS
     │                                            │
     │                                            │
     └─── Outbound Resolver Endpoint ←─────────────┘
```

## Security in Hybrid Networks

### Encryption

| Traffic Path | Encryption |
|-------------|-----------|
| On-prem → VPN | IPSec (mandatory) |
| On-prem → DX | No encryption (physical security) — add VPN over DX or use TLS |
| On-prem → DX + VPN backup | IPSec on VPN, plain on DX (add MACsec for DX) |
| VPC → VPC | AWS backbone (encrypted) |

### MACsec on Direct Connect

For DX encryption if required:

```bash
aws directconnect update-connection \
  --connection-id dxcon-xxx \
  --encryption-mode should_encrypt
```

### Security Groups & NACLs

Apply consistent SG rules for VPC resources regardless of traffic source (on-prem or VPC).

## Monitoring Hybrid Connections

### CloudWatch Metrics

| Metric | What It Tells You |
|--------|-------------------|
| `TunnelState` | VPN tunnel health (0=down, 1=up) |
| `ConnectionState` | DX connection state |
| `BGPStatus` | BGP session health |
| `TunnelDataIn/Out` | VPN throughput |
| `VirtualInterfaceBgpMetrics` | DX BGP metrics |

### Alarms

```bash
# Alert if VPN goes down
aws cloudwatch put-metric-alarm \
  --alarm-name "VPN-Tunnel-Down" \
  --metric-name TunnelState \
  --namespace AWS/VPN \
  --statistic Minimum \
  --period 60 \
  --evaluation-periods 2 \
  --threshold 1 \
  --comparison-operator LessThanThreshold \
  --alarm-actions arn:aws:sns:us-east-1:xxx:network-alerts
```

## Failover Automation

```bash
# Script: check DX health, fail over to VPN if needed
#!/bin/bash
DX_STATUS=$(aws directconnect describe-connections \
  --connection-id dxcon-xxx \
  --query 'connections[0].connectionState' --output text)

if [ "$DX_STATUS" != "available" ]; then
  echo "DX is down. VPN should take over via BGP..."
  # Optionally adjust route metrics or notify
  aws sns publish --topic-arn arn:aws:sns:us-east-1:xxx:ops \
    --message "Direct Connect is DOWN. Failing over to VPN."
fi
```

## Cost Optimization

| Strategy | Savings |
|----------|---------|
| **VPN for low-bandwidth** | DX port costs $135+/month; VPN is free per-connection |
| **TGW segmentation** | Share DX/VPN across multiple VPCs |
| **Traffic prioritization** | Route critical traffic over DX, bulk over VPN |
| **NAT Gateway reduction** | Use VPC Endpoints to reduce NAT data processing costs |

## Gotchas

- **Route propagation**: you must enable it on each route table (VPC and TGW)
- **More specific routes win**: on-prem /24 beats VPC's default route
- **Asymmetric routing**: if one path goes in and another comes out — fix with route tables
- **DNS resolution**: hybrid DNS requires careful Route 53 Resolver configuration
- **MTU**: DX supports 9001 MTU, VPN supports 1436 — adjust TCP MSS accordingly
- **Failover testing**: must test DX → VPN failover regularly
- **BGP timers**: default 30s hold time — can be adjusted for faster convergence
