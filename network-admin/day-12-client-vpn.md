# Day 12 — Client VPN

## Concept

AWS Client VPN is a **managed OpenVPN-based service** that lets individual users (laptops, phones) connect securely to your AWS VPC.

## Architecture

```
Remote User
  ┌──────────┐
  │ Laptop   │─── OpenVPN ───► Client VPN Endpoint
  │ OpenVPN  │   TLS 1.2+    │
  └──────────┘               ├── Target Network (subnet)
                              │      │
                              │      ▼
                              │  VPC Resources
                              └────────────────────
```

## Components

### Client VPN Endpoint
- The VPN server — users connect to this
- Has a DNS name (e.g., `cvpn-endpoint-xxx.prod.clientvpn.us-east-1.amazonaws.com`)
- Configured with server certificate (from ACM) for TLS

### Target Network
- A subnet in your VPC
- The endpoint creates an ENI in this subnet
- You can associate multiple subnets for HA

### Authorization Rules
- Controls which users can access which networks
- Grant access to specific CIDRs
- Can be per-group (based on certificates/AD groups)

### Client Certificate
- Each user needs a client certificate
- Same CA or different — your choice

## Setup

### Prerequisites
1. Server certificate + key in ACM (or uploaded)
2. Client certificates for each user
3. Security group for the endpoint

### Step 1: Create Server Certificate in ACM

```bash
# Request a certificate for VPN server
aws acm request-certificate \
  --domain-name "vpn.example.com" \
  --validation-method DNS

# Or upload existing
aws acm import-certificate \
  --certificate fileb://server.crt \
  --private-key fileb://server.key \
  --certificate-chain fileb://ca-chain.crt
```

### Step 2: Create Client VPN Endpoint

```bash
aws ec2 create-client-vpn-endpoint \
  --client-cidr-block "10.200.0.0/16" \
  --server-certificate-arn arn:aws:acm:us-east-1:xxx:certificate/xxx \
  --authentication-options Type=certificate-authentication,MutualAuthentication={ClientRootCertificateChainArn=arn:aws:acm:us-east-1:xxx:certificate/xxx} \
  --connection-log-options Enabled=false \
  --dns-servers 10.0.0.2
```

**Key Parameters:**
| Parameter | Description |
|-----------|-------------|
| `client-cidr-block` | IP pool for VPN clients (must not overlap VPC CIDR) |
| `server-certificate-arn` | Server cert from ACM |
| `authentication-options` | Certificate-based, SAML, or Active Directory |

### Step 3: Associate Target Network

```bash
aws ec2 associate-client-vpn-target-network \
  --client-vpn-endpoint-id cvpn-endpoint-xxx \
  --subnet-id subnet-12345678
```

### Step 4: Add Authorization Rule

```bash
# Grant all authenticated users access to the VPC
aws ec2 authorize-client-vpn-ingress \
  --client-vpn-endpoint-id cvpn-endpoint-xxx \
  --target-network-cidr 10.0.0.0/16 \
  --authorize-all-groups
```

### Step 5: Allow Inbound Traffic in Security Group

The endpoint's ENI needs SGs that allow inbound traffic from the client CIDR block.

### Step 6: Export Client Configuration

```bash
aws ec2 export-client-vpn-client-configuration \
  --client-vpn-endpoint-id cvpn-endpoint-xxx \
  --output text > client-config.ovpn
```

Distribute this `.ovpn` file + client certificate to users.

## Authentication Options

| Type | How It Works | Use Case |
|------|-------------|----------|
| **Certificate-based** | Client + server certificates | Simple, no AD required |
| **SAML** | Federated identity (Okta, Azure AD) | Enterprise SSO |
| **Active Directory** | AWS Managed Microsoft AD | Existing AD users |

### SAML Setup

```bash
aws ec2 create-client-vpn-endpoint \
  --authentication-options Type=federated-authentication,FederatedAuthentication={SAMLProviderArn=arn:aws:iam::xxx:saml-provider/Okta,SelfServiceSAMLProviderArn=...}
```

## Split Tunneling vs Full Tunnel

### Split Tunnel (default)
Only traffic destined for AWS goes through VPN; internet traffic goes directly.

```bash
# Add route for AWS VPC only
aws ec2 create-client-vpn-route \
  --client-vpn-endpoint-id cvpn-endpoint-xxx \
  --destination-cidr-block "10.0.0.0/8" \
  --target-vpc-subnet-id subnet-12345678
```

### Full Tunnel
All traffic goes through VPN.

```bash
# Add route for 0.0.0.0/0
aws ec2 create-client-vpn-route \
  --client-vpn-endpoint-id cvpn-endpoint-xxx \
  --destination-cidr-block "0.0.0.0/0" \
  --target-vpc-subnet-id subnet-12345678
```

## CLI Commands

```bash
# Endpoint management
aws ec2 create-client-vpn-endpoint
aws ec2 describe-client-vpn-endpoints
aws ec2 delete-client-vpn-endpoint
aws ec2 modify-client-vpn-endpoint

# Target network
aws ec2 associate-client-vpn-target-network
aws ec2 disassociate-client-vpn-target-network

# Authorization
aws ec2 authorize-client-vpn-ingress
aws ec2 revoke-client-vpn-ingress

# Routes
aws ec2 create-client-vpn-route
aws ec2 describe-client-vpn-routes
aws ec2 delete-client-vpn-route

# Client config export
aws ec2 export-client-vpn-client-configuration
```

## Client Software

Users need an **OpenVPN-compatible client**:
- **AWS Client VPN** (free, recommended)
- OpenVPN Connect
- Tunnelblick (macOS)
- Built-in Windows VPN (supports OpenVPN)

## Console

**VPC → Client VPN Endpoints** — full management
**AWS Client VPN** — download the desktop client

## Gotchas

- **Client CIDR block must not overlap** with VPC CIDR or on-prem CIDRs
- **Max 200 connections per endpoint** (default — can be increased)
- **Client CIDR must be /22 or larger** (minimum 1024 IPs)
- **Connections are charged per hour** ($0.01/hour per connection)
- **Mutual TLS**: both server and client certificates required
- **Self-service portal**: users can download their own client certificates
- **Maximum Transmission Unit**: 1500 bytes
- **Log connections** to CloudWatch Logs for auditing
- **VPN termination**: idle timeout default is 1 hour (configurable)
