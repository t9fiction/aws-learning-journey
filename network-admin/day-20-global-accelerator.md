# Day 20 — Global Accelerator

## Concept

AWS Global Accelerator improves the **availability and performance** of your applications by directing traffic through AWS's global network infrastructure — bypassing the public internet.

## How It Works

```
User (Japan)
    │
    ▼
   Anycast IP (static)
    │
    ▼
  AWS Edge Location (Tokyo)
    │
    ▼
  AWS Global Network (private fiber)
    │
    ▼
  Application Endpoint (us-east-1 ALB)
```

**Instead of:**
```
User (Japan)
    │
    ▼
  Public Internet (unpredictable, many hops)
    │
    ▼
  Application (us-east-1)
```

## Why Use It?

| Benefit | What It Means |
|---------|---------------|
| **60% less latency** | User traffic enters AWS edge closest to them, travels on AWS network |
| **Static IPs** | Two static Anycast IPs (IPv4) — never change |
| **Instant failover** | Health check detects failure in <1 second, routes to healthy endpoint |
| **TCP optimization** | Termination at edge, optimized TCP back to origin |
| **No cache** | Not a CDN — improves performance for **dynamic** content |
| **Protocols** | TCP and UDP |

## Architecture

```
Global Accelerator
    ├── Listener (port 80/443)
    │       ├── Endpoint Group (us-east-1)
    │       │       ├── ALB (healthy)
    │       │       └── NLB (healthy)
    │       └── Endpoint Group (eu-west-1)
    │               └── ALB (healthy)
    │
    └── Two static Anycast IPs
```

### Components

- **Listener**: process inbound connections (port + protocol)
- **Endpoint Groups**: region-based groups of endpoints (with traffic dials)
- **Endpoints**: ALB, NLB, EC2 instance, Elastic IP

## Create Global Accelerator

```bash
# 1. Create accelerator
aws globalaccelerator create-accelerator \
  --name "prod-accelerator" \
  --ip-address-type IPV4 \
  --enabled

# 2. Create listener (HTTPS)
aws globalaccelerator create-listener \
  --accelerator-arn arn:aws:globalaccelerator:::accelerator/xxx \
  --port-ranges From=443,To=443 \
  --protocol TCP \
  --client-affinity NONE

# 3. Create endpoint group (us-east-1)
aws globalaccelerator create-endpoint-group \
  --listener-arn arn:aws:globalaccelerator:::listener/xxx/xxx \
  --endpoint-group-region us-east-1 \
  --traffic-dial-percentage 100 \
  --health-check-port 80 \
  --health-check-protocol HTTP \
  --health-check-path /health

# 4. Add endpoint (ALB)
aws globalaccelerator add-endpoints \
  --endpoint-group-arn arn:aws:globalaccelerator:::endpoint-group/xxx/xxx \
  --endpoint-configurations '[
    {
      "EndpointId": "arn:aws:elasticloadbalancing:us-east-1:xxx:loadbalancer/app/web-alb/xxx",
      "Weight": 128,
      "ClientIPPreservationEnabled": true
    }
  ]'
```

## Traffic Dials

Control the percentage of traffic sent to each region:

```bash
# Send 100% to us-east-1, 0% to eu-west-1
aws globalaccelerator update-endpoint-group \
  --endpoint-group-arn arn:aws:globalaccelerator:::endpoint-group/xxx/xxx \
  --traffic-dial-percentage 100
```

Useful for:
- Gradual rollouts (10% → 50% → 100%)
- Blue/green deployment by region
- Draining traffic from a region

## Client IP Preservation

Controls whether the backend sees the user's real IP.

```bash
aws globalaccelerator add-endpoints \
  --endpoint-group-arn arn:aws:globalaccelerator:::endpoint-group/xxx/xxx \
  --endpoint-configurations '[
    {
      "EndpointId": "arn:aws:elasticloadbalancing:...",
      "ClientIPPreservationEnabled": true
    }
  ]'
```

- **ALB**: source IP preserved by default
- **NLB**: source IP preserved by default
- **EC2 (via NLB)**: set `ClientIPPreservationEnabled=true`

## DNS Names

Global Accelerator provides:

- **Static IPs**: 2 Anycast IPs (you can bring your own via BYOIP)
- **DNS Name**: `a123456.awsglobalaccelerator.com`
- Use Route 53 ALIAS record to point your domain to the DNS name

## Accelerated Site-to-Site VPN

Use Global Accelerator to accelerate VPN traffic:

```bash
# Create accelerated VPN connection
aws ec2 create-vpn-connection \
  --type ipsec.1 \
  --customer-gateway-id cgw-xxx \
  --transit-gateway-id tgw-xxx \
  --options "EnableAcceleration=true"
```

**How it works**: VPN traffic enters at nearest edge → travels on AWS global network → exits at the region.

## Pricing

| Component | Cost |
|-----------|------|
| Accelerator | $0.025/hour ($18/month) |
| Data transfer (us-east-1 → internet) | $0.06/GB (premium) |
| Data transfer (edge → us-east-1) | $0.01/GB |

Total: ~20-30% premium over standard data transfer for the performance improvement.

## CLI Commands

```bash
aws globalaccelerator create-accelerator
aws globalaccelerator describe-accelerator
aws globalaccelerator update-accelerator
aws globalaccelerator delete-accelerator
aws globalaccelerator create-listener
aws globalaccelerator create-endpoint-group
aws globalaccelerator update-endpoint-group
aws globalaccelerator add-endpoints
aws globalaccelerator remove-endpoints
```

## Global Accelerator vs CloudFront

| Feature | Global Accelerator | CloudFront |
|---------|-------------------|------------|
| Layer | 4 (TCP/UDP) | 7 (HTTP/HTTPS) |
| Caching | No | Yes |
| Static IPs | Yes (2 Anycast) | No (must use DNS) |
| Protocol | TCP, UDP | HTTP, HTTPS, WebSocket |
| Use case | Dynamic content, APIs | Static content, cacheable |
| Failover | <1 second | Depends on TTL |
| Cost | Data transfer premium | Standard data transfer |

**Use both**: Global Accelerator for TCP optimization + failover, CloudFront for caching.

## Console

**Global Accelerator → Accelerators** — create and manage
**Global Accelerator → Listener** — configure ports
**Global Accelerator → Endpoint Groups** — traffic dials

## Gotchas

- **Two static IPs always allocated** — one of two addresses is always enabled
- **Cannot delete individual IPs** — both come together
- **Health checks**: endpoint groups must have at least one healthy endpoint or traffic is rejected
- **Weighted endpoints**: distribute traffic across endpoints within a group using weights
- **BYOIP**: you can bring your own IP addresses (requires 24-month commitment)
- **No WebSocket support** — use CloudFront or ALB directly
- **Client IP preservation disabled**: by default the backend sees the edge IP, not the client — enable `ClientIPPreservationEnabled`
- **DNS name changes if accelerator is deleted** — use CNAME/ALIAS, not A record for the accelerator DNS name
