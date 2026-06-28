# Day 8 — VPC Peering

## Concept

VPC Peering connects **two VPCs** so they can communicate as if they were on the same network — using **private IPs only**, no internet gateway needed.

## Key Facts

| Property | Value |
|----------|-------|
| **One-to-one** | A peering connection links exactly two VPCs (no transitive routing) |
| **Same or cross-region** | Can peer within a region or across regions |
| **Same or cross-account** | Can peer your VPC with another account's VPC |
| **No overlapping CIDRs** | CIDR blocks **must not overlap** |
| **No transitive routing** | If A peers with B and B peers with C, A cannot reach C |
| **No VPN/Gateway sharing** | Cannot share VPN/Direct Connect through peering |
| **Private IP only** | Traffic stays within AWS network |
| **No bandwidth limit** | No extra charge for peering (only data transfer) |
| **Encryption** | Traffic is encrypted by default |

## Transitive Routing Problem

```
VPC A ─── Peer ─── VPC B ─── Peer ─── VPC C
  │                    │
  └────── CANNOT ──────┘       ✗ No transitive routing
```

**Solution**: Use **Transit Gateway** (Day 9) for mesh/net topology.

## Creating a Peering Connection

### Same Account

```bash
# Request peering
aws ec2 create-vpc-peering-connection \
  --vpc-id vpc-12345678 \
  --peer-vpc-id vpc-87654321 \
  --region us-east-1
```

### Cross-Account

```bash
aws ec2 create-vpc-peering-connection \
  --vpc-id vpc-12345678 \
  --peer-vpc-id vpc-87654321 \
  --peer-owner-id 999999999999 \
  --peer-region us-west-2
```

### Accept Peering (requester or accepter)

```bash
aws ec2 accept-vpc-peering-connection \
  --vpc-peering-connection-id pcx-12345678
```

## The Critical Step: Route Tables

Creating the peering does **nothing** until you add routes. You must update **both** VPCs' route tables.

```bash
# In VPC A's route table — tell it where to find VPC B
aws ec2 create-route \
  --route-table-id rtb-vpc-a \
  --destination-cidr-block 10.1.0.0/16 \   # VPC B's CIDR
  --vpc-peering-connection-id pcx-12345678

# In VPC B's route table — tell it where to find VPC A
aws ec2 create-route \
  --route-table-id rtb-vpc-b \
  --destination-cidr-block 10.0.0.0/16 \   # VPC A's CIDR
  --vpc-peering-connection-id pcx-12345678
```

### Multiple Subnets

If VPC A has public and private subnets with different route tables, you need a route **in each route table** that needs to reach VPC B.

## Cross-Region Peering

Same CLI commands but specify the peer's region.

```bash
aws ec2 create-vpc-peering-connection \
  --vpc-id vpc-12345678 \
  --peer-vpc-id vpc-abcdef01 \
  --peer-region eu-west-1

# Accept on the accepter side
aws ec2 accept-vpc-peering-connection \
  --vpc-peering-connection-id pcx-xxx \
  --region eu-west-1
```

**Cross-region peering adds:**
- Higher latency (traffic leaves region)
- Cross-region data transfer costs ($0.01-$0.02/GB each way)
- No extra hourly cost for the peering itself

## Security Group References

You can reference SGs across peered VPCs (same region only).

```bash
# Allow access from SG in another VPC (via peering)
aws ec2 authorize-security-group-ingress \
  --group-id sg-in-vpc-a \
  --protocol tcp \
  --port 3306 \
  --source-group sg-in-vpc-b \     # SG in peer VPC
  --group-owner 111111111111       # Account that owns VPC B
```

**Cross-region peering**: Cannot reference SGs — use CIDR instead.

## DNS Resolution

Control how DNS names resolve across peered VPCs.

```bash
# Enable DNS resolution from accepter to requester
aws ec2 modify-vpc-peering-connection-options \
  --vpc-peering-connection-id pcx-12345678 \
  --requester-peering-connection-options AllowDnsResolutionFromRemoteVpc=true \
  --accepter-peering-connection-options AllowDnsResolutionFromRemoteVpc=true
```

## CLI Commands

```bash
aws ec2 create-vpc-peering-connection
aws ec2 accept-vpc-peering-connection
aws ec2 reject-vpc-peering-connection
aws ec2 delete-vpc-peering-connection
aws ec2 describe-vpc-peering-connections
aws ec2 modify-vpc-peering-connection-options
aws ec2 create-route                   # to add peering to route table
```

## Common Patterns

### Hub-and-Spoke (with peering limitations)

```
         ┌──────────┐
         │  Shared  │
         │ Services │
         │ VPC (Hub)│
         └────┬─────┘
       ┌──────┼──────┐
       │      │      │
   ┌───┴──┐ ┌─┴───┐ ┌┴───┐
   │ Dev  │ │ QA  │ │Prod│
   └──────┘ └─────┘ └────┘
```

Each spoke peers with the hub. **Spokes cannot reach each other** (no transitive routing). For true transitive routing, use Transit Gateway.

## Gotchas

- **Overlapping CIDRs cannot peer** — plan your IP space carefully
- **No transitive routing** — each pair needs its own peering connection
- **100 peering connections per VPC** (soft limit)
- **Cross-region peering** has higher latency and data transfer costs
- **Route propagation**: peering routes must be added **manually** to route tables
- **DNS hostnames**: by default, instances in peered VPCs resolve each other's DNS names using the private DNS
- **IPv6** peering is supported
- **Edge-to-edge routing restriction**: If VPC A (peered with VPC B) has a VPN, VPC B cannot use that VPN to reach on-premises
