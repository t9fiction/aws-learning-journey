# Day 29 — PrivateLink & VPC Lattice

## PrivateLink

### Concept

AWS PrivateLink lets you expose your services **privately** to other VPCs or on-prem networks — without traversing the internet, using NAT, or needing VPC Peering/Transit Gateway.

Think of it as: "I have a service running in my VPC. Other accounts can access it without their traffic ever going over the internet."

### Architecture

```
Consumer VPC                    Service VPC
  ┌──────────────┐              ┌───────────────────┐
  │ EC2 Instance  │──── ENI ───►│ NLB (service)      │
  │ (private IP)  │             │  ┌───────────────┐ │
  └──────────────┘             │  │ Application   │ │
       │                       │  │ (EC2, ECS,    │ │
  Interface Endpoint           │  │  Lambda)      │ │
  (ENI in consumer subnet)     │  └───────────────┘ │
                               └───────────────────┘
```

### Two Roles

| Role | What They Do |
|------|-------------|
| **Service Provider** | Creates a NLB, registers targets, creates a VPC Endpoint Service |
| **Service Consumer** | Creates an Interface Endpoint in their VPC to connect |

### Service Provider Setup

```bash
# 1. Create NLB for your service
aws elbv2 create-load-balancer \
  --name svc-nlb --type network \
  --subnets subnet-1a subnet-1b

aws elbv2 create-target-group --name svc-tg --protocol TCP --port 80 --vpc-id vpc-svc
aws elbv2 register-targets --target-group-arn tg-xxx --targets Id=i-xxx
aws elbv2 create-listener --load-balancer-arn nlb-xxx --protocol TCP --port 80 \
  --default-actions Type=forward,TargetGroupArn=tg-xxx

# 2. Create VPC Endpoint Service
aws ec2 create-vpc-endpoint-service-configuration \
  --network-load-balancer-arns arn:aws:elasticloadbalancing:us-east-1:xxx:loadbalancer/net/svc-nlb/xxx \
  --acceptance-required true \
  --private-dns-name svc.example.com
```

### Service Consumer Setup

```bash
# 1. Create Interface Endpoint in your VPC
aws ec2 create-vpc-endpoint \
  --vpc-id vpc-consumer \
  --vpc-endpoint-type Interface \
  --service-name com.amazonaws.vpce.us-east-1.vpce-svc-xxx \
  --subnet-ids subnet-consumer-1a \
  --security-group-ids sg-consumer
```

### Acceptance

If `acceptance-required` is `true`, the provider must accept:

```bash
aws ec2 accept-vpc-endpoint-connections \
  --service-id vpce-svc-xxx \
  --vpc-endpoint-ids vpce-xxx
```

### Private DNS Name

Service providers can configure a private DNS name so consumers connect via a friendly name instead of the endpoint ID:

```bash
aws ec2 modify-vpc-endpoint-service-configuration \
  --service-id vpce-svc-xxx \
  --private-dns-name svc.internal.example.com
```

Consumers set `--private-dns-enabled` when creating the endpoint.

### CLI Commands

```bash
# Provider
aws ec2 create-vpc-endpoint-service-configuration
aws ec2 describe-vpc-endpoint-service-configurations
aws ec2 accept-vpc-endpoint-connections
aws ec2 reject-vpc-endpoint-connections

# Consumer
aws ec2 create-vpc-endpoint  # type=Interface
aws ec2 describe-vpc-endpoints
aws ec2 delete-vpc-endpoints
```

### Pricing

| Component | Cost |
|-----------|------|
| Endpoint (consumer) | $0.01/hour per AZ |
| Data processed | $0.01/GB |
| Service (provider) | Free |

---

## VPC Lattice

### Concept

VPC Lattice is a **service-to-service networking** layer for connecting, monitoring, and securing communication between services — across accounts and VPCs.

Unlike PrivateLink which is one-to-one, Lattice creates a **service mesh** for your applications.

### Key Differences from PrivateLink

| Feature | PrivateLink | VPC Lattice |
|---------|-------------|-------------|
| Connection model | Point-to-point (one consumer → one provider) | Mesh (many services, many consumers) |
| Traffic visibility | Limited | Built-in access logs, metrics |
| Auth policies | NACL/SG-controlled | Lattice service policies (built-in) |
| Cross-account | Yes | Yes |
| Use case | Expose one service to one consumer | Full service mesh with discovery |

### Components

```
VPC Lattice Structure:

Service Network (logical container for related services)
  ├── Service: auth-svc
  │   ├── Target Group (EC2, Lambda, ECS)
  │   ├── Listener (HTTP:80)
  │   └── Service Policy (who can access)
  ├── Service: api-svc
  │   └── ...
  └── Service: web-svc
      └── ...

VPC Associations:
  ├── VPC-A (consumers) ← associated with Service Network
  └── VPC-B (providers) ← associated with Service Network
```

### Setup Service Network

```bash
# 1. Create service network
aws vpc-lattice create-service-network \
  --name "prod-mesh" \
  --auth-type AWS_IAM

# 2. Associate VPCs
aws vpc-lattice create-service-network-vpc-association \
  --service-network-identifier sn-xxx \
  --vpc-identifier vpc-consumer

aws vpc-lattice create-service-network-vpc-association \
  --service-network-identifier sn-xxx \
  --vpc-identifier vpc-service
```

### Create a Service

```bash
# 1. Create target group
aws vpc-lattice create-target-group \
  --name "api-targets" \
  --type INSTANCE \
  --config '{"port": 8080, "protocol": "HTTP", "vpcIdentifier": "vpc-service"}'

# 2. Register targets
aws vpc-lattice register-targets \
  --target-group-identifier tg-xxx \
  --targets Id=i-xxx,Port=8080

# 3. Create service
aws vpc-lattice create-service \
  --name "api-service" \
  --auth-type AWS_IAM \
  --certificate-arn arn:aws:acm:us-east-1:xxx:certificate/xxx
```

### Auth Policy

Control which services/accounts can access the service:

```bash
aws vpc-lattice put-auth-policy \
  --resource-identifier svc-xxx \
  --policy '{
    "Version": "2012-10-17",
    "Statement": [{
      "Effect": "Allow",
      "Principal": {"AWS": "arn:aws:iam::999999999999:root"},
      "Action": "vpc-lattice:Invoke",
      "Condition": {"StringEquals": {"vpc-lattice:ServiceNetwork": "sn-xxx"}}
    }]
  }'
```

### CLI Commands

```bash
aws vpc-lattice create-service-network
aws vpc-lattice create-service
aws vpc-lattice create-target-group
aws vpc-lattice register-targets
aws vpc-lattice create-service-network-vpc-association
aws vpc-lattice put-auth-policy
aws vpc-lattice create-listener
aws vpc-lattice create-rule
```

### Pricing

| Component | Cost |
|-----------|------|
| Service Network | $0.01/hour |
| Service | $0.025/hour |
| VPC Association | $0.01/hour per VPC |
| Data processed | $0.02/GB |

### Gotchas

- **Lattice is regional** — cross-region not supported
- **Requires IAM auth or custom auth** — not just "network open"
- **Target groups support**: EC2 instances, Lambda, ECS, IP addresses
- **Doesn't replace VPC connectivity** — you still need routing for non-service traffic
- **Service policies can restrict which VPCs can access the service**
- **Access logs** can be sent to S3, CloudWatch Logs, or Kinesis
- **Health checks**: built-in for target groups by default
