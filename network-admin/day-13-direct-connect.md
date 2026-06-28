# Day 13 — Direct Connect

## Concept

AWS Direct Connect (DX) is a **dedicated private network connection** from your on-premises data center to AWS. It bypasses the internet entirely — lower latency, higher bandwidth, more consistent performance.

## Why Direct Connect?

| Feature | VPN (Internet) | Direct Connect |
|---------|---------------|----------------|
| Connection | Shared internet | Dedicated fiber |
| Bandwidth | Up to 1 Gbps per tunnel | 50 Mbps — 100 Gbps |
| Latency | Variable, higher | Consistent, lower |
| SLA | None | 99.99% (with redundant connections) |
| Setup time | Hours | Weeks (physical fiber) |
| Cost | No recurring port fee | Monthly port fee + data transfer |
| Security | IPSec encryption | Private (no internet) |

## Physical Architecture

```
On-Prem DC                      AWS Direct Connect Location
  ┌──────────┐                 ┌──────────────────────┐
  │  Router   │─── Fiber ─────►│  Cross-Connect       │
  │  803.1Q   │   VLAN Tag     │  (Meet Me Room)      │
  └──────────┘                 └──────────┬───────────┘
       │                                    │
       │                            AWS Router
       │                                    │
       ◄────────── Virtual Interface ──────►│
       │                                    │
       ▼                                    ▼
   On-prem DC                          AWS Account
                              ┌──────────────────────┐
                              │  Direct Connect       │
                              │  Gateway (DGW)        │
                              └──────────┬───────────┘
                                          │
                                    Virtual Private
                                    Gateway (VGW) or
                                    Transit Gateway
```

## Connection Types

### Dedicated Connection
- Physical Ethernet port at a Direct Connect location
- You request a port, AWS provisions it
- Available speeds: 1 Gbps, 10 Gbps, 100 Gbps
- You need to work with the facility to complete cross-connect

```bash
aws directconnect create-connection \
  --location EqDC2 \
  --bandwidth 1Gbps \
  --connection-name "Primary-DC-Connection"
```

### Hosted Connection
- Purchased from an AWS Direct Connect Partner (Equinix, Megaport, etc.)
- Smaller bandwidths: 50 Mbps, 100 Mbps, 200 Mbps, 500 Mbps, 1 Gbps, 10 Gbps
- Faster provisioning (hours instead of weeks)

```bash
# Partner creates the hosted connection via AWS
# You accept it
aws directconnect accept-direct-connect-gateway-association-proposal \
  --direct-connect-gateway-id dgw-xxx \
  --proposal-id proposal-xxx \
  --associated-gateway-owner-account 999999999999
```

### LAG (Link Aggregation Group)
- Bundle multiple connections into one logical link
- Up to 4 connections, same bandwidth, same location

```bash
aws directconnect create-lag \
  --connection-name "Prod-LAG" \
  --connections-id conn-1 conn-2 \
  --number-of-connections 2
```

## Virtual Interfaces (VIFs)

After you have a connection, create Virtual Interfaces to use it.

### Private VIF
- Connects to a **VPC** (via VGW or TGW)
- Private IPs only (RFC 1918)
- Supports BGP routing

```bash
aws directconnect create-private-virtual-interface \
  --connection-id dxcon-xxx \
  --new-private-virtual-interface \
    virtualInterfaceName=Prod-VIF,vlan=100,asn=65000,\
    amazonAddress=172.16.0.2/30,customerAddress=172.16.0.1/30,\
    virtualGatewayId=vgw-12345678
```

### Public VIF
- Connects to **AWS public services** (S3, SQS, EC2 API, etc.)
- Uses public IPs
- Traffic stays off the internet

```bash
aws directconnect create-public-virtual-interface \
  --connection-id dxcon-xxx \
  --new-public-virtual-interface \
    virtualInterfaceName=Public-VIF,vlan=200,asn=65000,\
    amazonAddress=1.2.3.4/30,customerAddress=1.2.3.5/30,\
    routeFilterPrefixes=routeFilterPrefix=192.0.2.0/24
```

### Transit VIF
- Connects to a **Transit Gateway**
- Allows multiple VPCs through the same connection

```bash
aws directconnect create-transit-virtual-interface \
  --connection-id dxcon-xxx \
  --new-transit-virtual-interface \
    virtualInterfaceName=Transit-VIF,vlan=300,asn=65000,\
    directConnectGatewayId=dgw-xxx
```

## Direct Connect Gateway (DGW)

A DGW lets you connect to **multiple VPCs in multiple regions** with one DX connection.

```
On-prem ── DX ── Private VIF ── DGW ── VGW (us-east-1) ── VPC A
                                   └── VGW (eu-west-1) ── VPC B
```

```bash
# Create DGW
aws directconnect create-direct-connect-gateway \
  --direct-connect-gateway-name "Main-DGW" \
  --amazon-side-asn 64512

# Associate DGW with VGW
aws directconnect create-direct-connect-gateway-association \
  --direct-connect-gateway-id dgw-xxx \
  --gateway-id vgw-yyy
```

## Complete End-to-End Setup

```bash
# 1. Request DX connection (or partner-hosted)
CONN_ID=$(aws directconnect create-connection \
  --location EqDC2 --bandwidth 1Gbps \
  --connection-name "Prod-DX" \
  --query 'connectionId' --output text)

# 2. Create Direct Connect Gateway
DGW_ID=$(aws directconnect create-direct-connect-gateway \
  --direct-connect-gateway-name "Main-DGW" --amazon-side-asn 64512 \
  --query 'directConnectGateway.directConnectGatewayId' --output text)

# 3. Create private VIF
aws directconnect create-private-virtual-interface \
  --connection-id $CONN_ID \
  --new-private-virtual-interface \
    virtualInterfaceName=Prod-VIF,vlan=100,asn=65000,\
    amazonAddress=172.16.0.2/30,customerAddress=172.16.0.1/30,\
    directConnectGatewayId=$DGW_ID

# 4. Create/attach VGW to VPC
VGW_ID=$(aws ec2 create-vpn-gateway --type ipsec.1 \
  --query 'VpnGateway.VpnGatewayId' --output text)
aws ec2 attach-vpn-gateway --vpn-gateway-id $VGW_ID --vpc-id vpc-12345678

# 5. Associate DGW with VGW
aws directconnect create-direct-connect-gateway-association \
  --direct-connect-gateway-id $DGW_ID \
  --gateway-id $VGW_ID

# 6. Enable route propagation
aws ec2 enable-vgw-route-propagation \
  --route-table-id rtb-private --gateway-id $VGW_ID
```

## Routing with Direct Connect

### Prefix Advertisement
- **Private VIF**: BGP advertises your on-prem CIDRs to AWS
- **Public VIF**: AWS advertises public IP ranges; you advertise your public IPs
- **Transit VIF**: BGP over TGW — routes propagate to all associated VPCs

### BGP Communities
AWS uses BGP communities to influence routing:

| Community | Meaning |
|-----------|---------|
| `7224:7100` | Route originates from AWS Direct Connect |
| `7224:7200` | Route originates from AWS Site-to-Site VPN |
| `7224:7300` | Route originates from AWS Transit Gateway |

## CLI Commands

```bash
# Connections
aws directconnect create-connection
aws directconnect describe-connections
aws directconnect delete-connection

# Virtual Interfaces
aws directconnect create-private-virtual-interface
aws directconnect create-public-virtual-interface
aws directconnect create-transit-virtual-interface
aws directconnect describe-virtual-interfaces

# Direct Connect Gateway
aws directconnect create-direct-connect-gateway
aws directconnect describe-direct-connect-gateways
aws directconnect delete-direct-connect-gateway
aws directconnect create-direct-connect-gateway-association

# LAG
aws directconnect create-lag
aws directconnect describe-lags

# Locations
aws directconnect describe-locations
```

## Console

**Direct Connect → Connections** — manage physical connections
**Direct Connect → Virtual Interfaces** — create/manage VIFs
**Direct Connect → Direct Connect Gateways** — manage DGWs

## Cost

| Component | Cost |
|-----------|------|
| 1 Gbps dedicated port | ~$135/month |
| 10 Gbps dedicated port | ~$1,350/month |
| Hosted connection (partner) | Varies (usually cheaper) |
| Data transfer out | Lower than internet rates ($0.02-$0.08/GB) |
| Data transfer in | Free |

## Gotchas

- **Physical connection lead time**: 2-8 weeks for dedicated
- **Cannot connect to a VPC directly** — need VGW or TGW
- **Private VIF connects to one VGW** — use DGW for multiple VPCs
- **BGP authentication**: MD5 password supported on VIFs
- **Jumbo frames**: DX supports 9001 MTU (not VPN)
- **VIF must be in DOWN state** to modify or delete
- **DX location availability**: not available in all cities
- **Use DX in multiple locations** for redundancy
- **DX + VPN backup**: common pattern — DX primary, VPN as backup
