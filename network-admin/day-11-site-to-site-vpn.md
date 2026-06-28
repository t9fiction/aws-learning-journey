# Day 11 — Site-to-Site VPN

## Concept

Site-to-Site VPN connects your **on-premises network** to your **AWS VPC** over the internet via encrypted IPSec tunnels.

## Architecture

```
On-Premises                     AWS
  ┌──────────┐              ┌──────────┐
  │ Customer │─── IPSec ───►│  VPC     │
  │  Router  │   Tunnel 1   │  ┌───────┤
  │ (CGW)    │◄────────────│  │ VGW   │
  │          │─── IPSec ───►│  └───────┤
  └──────────┘   Tunnel 2   │  Subnet  │
      │                      └──────────┘
  10.0.0.0/16               172.16.0.0/16
```

## Components

### Customer Gateway (CGW)
- Represents your on-premises router/firewall in AWS
- Stores: **BGP ASN** (private or public), **public IP** of your router
- Does NOT create anything on your side — just a logical object

```bash
aws ec2 create-customer-gateway \
  --type ipsec.1 \
  --public-ip 203.0.113.12 \
  --bgp-asn 65000
```

### Virtual Private Gateway (VGW)
- The **AWS-side VPN concentrator**
- Attached to one VPC
- Handles IPSec termination

```bash
# Create VGW
aws ec2 create-vpn-gateway --type ipsec.1

# Attach to VPC
aws ec2 attach-vpn-gateway \
  --vpn-gateway-id vgw-12345678 \
  --vpc-id vpc-12345678
```

### VPN Connection
Links the CGW to the VGW with 2 tunnels (for HA).

```bash
aws ec2 create-vpn-connection \
  --type ipsec.1 \
  --customer-gateway-id cgw-12345678 \
  --vpn-gateway-id vgw-12345678 \
  --options "StaticRoutesOnly=false"
```

### Download Configuration

```bash
aws ec2 describe-vpn-connections \
  --vpn-connection-ids vpn-12345678
```

Then download the device-specific config from Console:
**VPC → Site-to-Site VPN Connections → Download Configuration**

Choose your device vendor (Cisco, Juniper, Palo Alto, etc.).

## Route Propagation

Enable route propagation on your VPC route tables so instances know how to reach on-prem.

```bash
# Enable route propagation on the route table
aws ec2 enable-vgw-route-propagation \
  --route-table-id rtb-private \
  --gateway-id vgw-12345678
```

**What this does**: Automatically adds routes for on-prem CIDRs learned via BGP (or static routes) to the route table.

### Static vs Dynamic (BGP) Routing

| Feature | Static | Dynamic (BGP) |
|---------|--------|---------------|
| Route announcement | Manual | Automatic |
| Failover | Manual or scripted | Automatic |
| Configuration | Simple | Complex |
| On-prem CIDR changes | Must update VPN config | Learned automatically |
| CGW BGP ASN | Not needed | Required |

## Tunnel Details

- **2 tunnels per VPN connection** for redundancy
- Tunnels go down **independently**
- Each tunnel has unique: public IP, pre-shared key, encryption options
- Dead Peer Detection (DPD) — detects tunnel failure
- Re-key every 1 hour or 22 GB of traffic

### Encryption Options

| Parameter | Options |
|-----------|---------|
| Encryption | AES128, AES256 |
| Integrity | SHA-1, SHA-256 |
| DH Groups | 2, 14, 15, 16, 17, 18, 19, 20, 21, 22, 23, 24 |
| Phase 1 lifetime | Default 1 day |
| Phase 2 lifetime | Default 1 hour |
| DPD timeout | Default 30 seconds |

## CLI Commands

```bash
# Customer Gateway
aws ec2 create-customer-gateway
aws ec2 describe-customer-gateways
aws ec2 delete-customer-gateway

# VPN Gateway
aws ec2 create-vpn-gateway
aws ec2 describe-vpn-gateways
aws ec2 attach-vpn-gateway
aws ec2 detach-vpn-gateway
aws ec2 delete-vpn-gateway

# VPN Connection
aws ec2 create-vpn-connection
aws ec2 describe-vpn-connections
aws ec2 delete-vpn-connection

# Route propagation
aws ec2 enable-vgw-route-propagation
aws ec2 disable-vgw-route-propagation
```

## Complete Setup Flow

```bash
# 1. Create CGW (your router info)
CGW_ID=$(aws ec2 create-customer-gateway \
  --type ipsec.1 --public-ip 203.0.113.12 --bgp-asn 65000 \
  --query 'CustomerGateway.CustomerGatewayId' --output text)

# 2. Create VGW and attach to VPC
VGW_ID=$(aws ec2 create-vpn-gateway --type ipsec.1 \
  --query 'VpnGateway.VpnGatewayId' --output text)
aws ec2 attach-vpn-gateway --vpn-gateway-id $VGW_ID --vpc-id vpc-12345678

# 3. Create VPN Connection
VPN_ID=$(aws ec2 create-vpn-connection \
  --type ipsec.1 \
  --customer-gateway-id $CGW_ID \
  --vpn-gateway-id $VGW_ID \
  --options "StaticRoutesOnly=false" \
  --query 'VpnConnection.VpnConnectionId' --output text)

# 4. Enable route propagation on private route tables
for RTB in $(aws ec2 describe-route-tables \
  --filters Name=vpc-id,Values=vpc-12345678 \
  --query 'RouteTables[?Associations[?Main!=`true` || Main==`false`]].RouteTableId' \
  --output text); do
  aws ec2 enable-vgw-route-propagation \
    --route-table-id $RTB --gateway-id $VGW_ID
done

# 5. Download configuration
aws ec2 describe-vpn-connections --vpn-connection-ids $VPN_ID
```

## Redundancy Best Practice

```
On-prem Router 1 ── Tunnel 1 ──► VGW (AZ 1)
On-prem Router 2 ── Tunnel 2 ──► VGW (AZ 2)

# Two CGWs, one VGW
aws ec2 create-customer-gateway --public-ip 203.0.113.12
aws ec2 create-customer-gateway --public-ip 203.0.113.13

# VPN with two CGWs
aws ec2 create-vpn-connection \
  --customer-gateway-id cgw-1 \
  --vpn-gateway-id vgw-xxx

aws ec2 create-vpn-connection \
  --customer-gateway-id cgw-2 \
  --vpn-gateway-id vgw-xxx
```

## Console

**VPC → Site-to-Site VPN Connections** — full management
**VPC → Customer Gateways** — manage CGWs
**VPC → Virtual Private Gateways** — manage VGWs

## Gotchas

- **Maximum transmission unit (MTU)**: 1436 bytes for VPN (vs 1500 for Direct Connect)
- **1 Gbps per tunnel** (can use ECMP with multiple tunnels for >1 Gbps)
- **VPN cannot do multicast**
- **TCP MSS clamping** should be set to 1387 to avoid fragmentation
- **Tunnel status**: tunnels show as UP after configuration on both sides
- **Pre-shared keys** are generated by AWS — you can modify them
- **CloudWatch metrics**: `TunnelState`, `TunnelDataIn`, `TunnelDataOut`
- When using a VPC attached to both IGW and VGW, be careful with route precedence
- Use **Accelerated Site-to-Site VPN** for better performance (uses Global Accelerator)
