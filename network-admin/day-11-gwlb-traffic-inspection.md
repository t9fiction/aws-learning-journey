# Day 11 — Gateway Load Balancer & Traffic Inspection

## Gateway Load Balancer (GWLB)

A **layer 3/4 gateway** that transparently forwards traffic to virtual appliances (firewalls, IDS/IPS, packet inspection) — then returns it to the original destination.

### Architecture

```
Internet
  │
  ▼
IGW
  │
  ▼
Route table: 0.0.0.0/0 → GWLB Endpoint
  │
  ▼
GWLB Endpoint (VPC)
  │
  ▼
Gateway Load Balancer
  │
  ├── Appliance AZ-A (Palo Alto, Fortinet, etc.)
  └── Appliance AZ-B (Palo Alto, Fortinet, etc.)
  │
  ▼
GWLB returns traffic to original path
  │
  ▼
Destination instance (web server)
```

### How It Works

1. Route table points to **GWLB endpoint** as next hop
2. GWLB endpoint **encapsulates traffic** (GENEVE protocol) and sends to GWLB
3. GWLB distributes traffic across appliance targets
4. Appliances inspect/process and return traffic
5. GWLB endpoint **de-encapsulates** and forwards to original destination

### Key Facts

- **GENEVE protocol**: GWLB uses GENEVE encapsulation (UDP port 6081)
- **Source IP preservation**: the appliance sees the original client IP
- **Flow stickiness**: all packets in a flow go to the same appliance
- **Health checks**: GWLB health-checks appliances before sending traffic
- **Scaling**: appliances can be in an Auto Scaling group
- **No SNAT**: GWLB doesn't change source/destination IPs

### Create GWLB

```bash
# 1. Create target group (appliance instances)
aws elbv2 create-target-group \
  --name fw-tg \
  --protocol GENEVE \
  --port 6081 \
  --vpc-id vpc-12345678 \
  --health-check-protocol TCP \
  --health-check-port 8080

# 2. Register appliance instances
aws elbv2 register-targets \
  --target-group-arn arn:aws:elasticloadbalancing:us-east-1:xxx:targetgroup/fw-tg/xxx \
  --targets Id=i-firewall-1 Id=i-firewall-2

# 3. Create Gateway Load Balancer
aws elbv2 create-load-balancer \
  --name gwlb-prod \
  --type gateway \
  --subnets subnet-firewall-1a subnet-firewall-1b

# 4. Create listener
aws elbv2 create-listener \
  --load-balancer-arn arn:aws:elasticloadbalancing:us-east-1:xxx:loadbalancer/gateway/gwlb-prod/xxx \
  --default-actions Type=forward,TargetGroupArn=arn:aws:elasticloadbalancing:us-east-1:xxx:targetgroup/fw-tg/xxx

# 5. Create VPC Endpoint for GWLB (in the VPC being protected)
aws ec2 create-vpc-endpoint \
  --vpc-endpoint-type GatewayLoadBalancer \
  --service-name com.amazonaws.vpce.us-east-1.vpce-svc-xxx \
  --subnet-ids subnet-web-1a subnet-web-1b
```

### Deploy Pattern

```
Inspection VPC (dedicated for security appliances):

Appliance AZ-A              Appliance AZ-B
  ┌──────────────┐          ┌──────────────┐
  │ Palo Alto FW │          │ Palo Alto FW │
  └──────┬───────┘          └──────┬───────┘
         │                         │
         └────────┬────────────────┘
                  │
           Gateway Load Balancer
                  │
        GWLB Endpoint (inspection VPC)
                  │
         VPC Peering / TGW
                  │
        GWLB Endpoint (workload VPC)
                  │
         Route: 0.0.0.0/0 → GWLBE
```

### GWLB vs VPC Network Firewall

| Feature | Network Firewall | GWLB |
|---------|-----------------|------|
| **Inspection engine** | AWS-managed (Suricata) | Your own appliance |
| **Custom rules** | Suricata rules | Vendor-specific (PAN-OS, FortiOS) |
| **TLS decryption** | AWS-managed | Vendor-managed |
| **Third-party integration** | No | Yes (Palo Alto, Fortinet, Check Point, etc.) |
| **Cost** | $0.395/hr + data | $0.0225/hr + data + appliance cost |
| **Latency** | Lower | Slightly higher (encapsulation) |

## TGW Appliance Mode

Transit Gateway can send traffic through appliances for inspection by attaching to GWLB:

```
VPC-A ── TGW ── GWLB Endpoint ── GWLB ── Appliance ── TGW ── VPC-B
```

Enable appliance mode on the TGW VPC attachment:

```bash
aws ec2 modify-transit-gateway-vpc-attachment \
  --transit-gateway-attachment-id tgw-attach-xxx \
  --options "ApplianceModeSupport=enable"
```

This ensures that traffic between VPCs flows through the appliance.

## VPC Traffic Mirroring (Recap Day 26)

Also part of the traffic inspection toolkit — copies full packets to a monitoring appliance:

```bash
aws ec2 create-traffic-mirror-session \
  --network-interface-id eni-web-xxx \
  --traffic-mirror-target-id tmt-ids-xxx \
  --traffic-mirror-filter-id tmf-xxx \
  --session-number 1 \
  --packet-length 65535
```

## Use Case Comparison

| Need | Solution |
|------|----------|
| All internet traffic inspected by Palo Alto | GWLB + Palo Alto in inspection VPC |
| East-west VPC traffic inspected | TGW Appliance Mode + GWLB |
| Full packet capture for compliance | Traffic Mirroring to IDS |
| Web app protected from SQLi/XSS | WAF (Day 22) |
| VPC-level stateful firewall | Network Firewall (Day 24) |
| NGFW features (IPS, URL filtering, TLS decrypt) | GWLB + third-party appliance |

## CLI Commands

```bash
# GWLB
aws elbv2 create-load-balancer --type gateway
aws elbv2 create-target-group --protocol GENEVE
aws elbv2 create-listener
aws ec2 create-vpc-endpoint --type GatewayLoadBalancer

# TGW Appliance Mode
aws ec2 modify-transit-gateway-vpc-attachment --options ApplianceModeSupport=enable

# Traffic Mirroring
aws ec2 create-traffic-mirror-session
aws ec2 describe-traffic-mirror-sessions
```

## Console

**EC2 → Load Balancers** — Gateway Load Balancer type
**VPC → Endpoints** — GWLB Endpoint type
**VPC → Traffic Mirroring** — sessions, targets, filters

## Gotchas

- **GENEVE port 6081 must be open** on appliance security groups and NACLs
- **GWLB endpoints are regional** — deploy per AZ for HA
- **Appliance scaling**: use Auto Scaling to add/remove appliances
- **Flow stickiness**: GWLB sends all packets of a flow to the same appliance (5-tuple hash)
- **Inspection VPC pattern**: put appliances in a separate VPC and route traffic through it
- **MTU**: GENEVE adds 58 bytes overhead — ensure path MTU is at least 1558
- **Traffic Mirroring costs**: $0.015/hr per session — don't leave unused sessions running
- **GWLB availability**: not available in all regions (check before designing)
