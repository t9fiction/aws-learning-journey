# Day 4 — Subnets, Route Tables & Internet Gateway

## Subnets

A subnet is a **range of IPs** within a VPC, tied to a **single Availability Zone**.

### Public vs Private Subnet

| Feature | Public Subnet | Private Subnet |
|---------|--------------|----------------|
| Route to IGW | Yes | No |
| Auto-assign public IP | Usually yes | Usually no |
| Internet access | Direct | Via NAT Gateway |
| Use cases | Load balancers, bastion hosts, NAT gateways | Databases, app servers, internal services |

### Create a Subnet

```bash
aws ec2 create-subnet \
  --vpc-id vpc-12345678 \
  --cidr-block 10.0.1.0/24 \
  --availability-zone us-east-1a
```

### Auto-Assign Public IP

```bash
# Enable auto-assign public IP for subnet
aws ec2 modify-subnet-attribute \
  --subnet-id subnet-12345678 \
  --map-public-ip-on-launch
```

## Route Tables

Each subnet must be associated with **one route table** (can be the main/default or custom).

A route table contains **routes** — each route has:
- **Destination** — CIDR block (e.g. `0.0.0.0/0` for internet, `10.0.0.0/16` for local)
- **Target** — where to send traffic (IGW, NAT, Peering, etc.)

### Implicit Local Route

Every route table has an **automatic local route**:

| Destination | Target |
|-------------|--------|
| `10.0.0.0/16` | local |

This is the **VPC's CIDR** — all traffic within the VPC uses this. **Cannot be deleted**.

### Create Route Table

```bash
aws ec2 create-route-table --vpc-id vpc-12345678
```

### Add Routes

```bash
# Route all internet traffic (0.0.0.0/0) to Internet Gateway
aws ec2 create-route \
  --route-table-id rtb-12345678 \
  --destination-cidr-block 0.0.0.0/0 \
  --gateway-id igw-12345678
```

### Associate Route Table with Subnet

```bash
aws ec2 associate-route-table \
  --route-table-id rtb-12345678 \
  --subnet-id subnet-12345678
```

### Main Route Table

- Every VPC has one "main" route table
- Subnets **without explicit association** use the main table
- You can replace the main table but cannot delete it

## Internet Gateway (IGW)

An IGW is a **horizontally-scaled, redundant, managed** gateway that allows VPC-to-internet communication.

### Key Facts
- **One VPC can have only one IGW** attached (and vice versa)
- IGW performs **NAT for instances with public IPs** (1:1 mapping of private IP to public IP)
- No bandwidth limits
- No additional cost (only data transfer out)
- Acts as a **target in route tables** — doesn't block traffic, just routes it

### Create & Attach IGW

```bash
# Create
aws ec2 create-internet-gateway

# Attach to VPC
aws ec2 attach-internet-gateway \
  --internet-gateway-id igw-12345678 \
  --vpc-id vpc-12345678
```

## Complete Public Subnet Setup

```bash
# 1. Create VPC
VPC_ID=$(aws ec2 create-vpc --cidr-block 10.0.0.0/16 --query 'Vpc.VpcId' --output text)

# 2. Enable DNS hostnames
aws ec2 modify-vpc-attribute --vpc-id $VPC_ID --enable-dns-hostnames

# 3. Create subnet
SUBNET_ID=$(aws ec2 create-subnet \
  --vpc-id $VPC_ID \
  --cidr-block 10.0.1.0/24 \
  --availability-zone us-east-1a \
  --query 'Subnet.SubnetId' --output text)

# 4. Enable auto-assign public IP
aws ec2 modify-subnet-attribute \
  --subnet-id $SUBNET_ID \
  --map-public-ip-on-launch

# 5. Create IGW
IGW_ID=$(aws ec2 create-internet-gateway --query 'InternetGateway.InternetGatewayId' --output text)
aws ec2 attach-internet-gateway --internet-gateway-id $IGW_ID --vpc-id $VPC_ID

# 6. Create custom route table
RTB_ID=$(aws ec2 create-route-table --vpc-id $VPC_ID --query 'RouteTable.RouteTableId' --output text)

# 7. Add route to IGW
aws ec2 create-route --route-table-id $RTB_ID \
  --destination-cidr-block 0.0.0.0/0 --gateway-id $IGW_ID

# 8. Associate route table with subnet
aws ec2 associate-route-table --route-table-id $RTB_ID --subnet-id $SUBNET_ID

echo "VPC: $VPC_ID, Subnet: $SUBNET_ID, IGW: $IGW_ID, Route Table: $RTB_ID"
```

## Traffic Flow: Instance to Internet

```
Instance (10.0.1.10)
  └─ ENI sends packet to VPC router
       └─ Route lookup: 0.0.0.0/0 → igw-xxx
            └─ IGW does 1:1 NAT (private IP ↔ public IP)
                 └─ Packet reaches internet
```

## CLI Commands Summary

```bash
# Subnet operations
aws ec2 create-subnet
aws ec2 delete-subnet
aws ec2 describe-subnets
aws ec2 modify-subnet-attribute

# Route table operations
aws ec2 create-route-table
aws ec2 delete-route-table
aws ec2 describe-route-tables
aws ec2 create-route
aws ec2 delete-route
aws ec2 replace-route
aws ec2 associate-route-table
aws ec2 disassociate-route-table

# Internet Gateway operations
aws ec2 create-internet-gateway
aws ec2 delete-internet-gateway
aws ec2 attach-internet-gateway
aws ec2 detach-internet-gateway
```

## Gotchas

- **Subnets cannot span AZs** — one subnet = one AZ
- Deleting a subnet: must have **no running instances**, no ENIs
- You can't resize a subnet — need to delete and re-create
- Route table associations **override** the main table — explicit > main
- A subnet can only be associated with **one route table** at a time
- IGW can only be attached to **one VPC**
- **IGW does not get an IP address** — it's a logical gateway
- The local route cannot be deleted or modified
- When `0.0.0.0/0` points at IGW, that subnet is "public"
