# Day 15 — Elastic Load Balancers (ALB, NLB, CLB)

## Concept

Elastic Load Balancing (ELB) **distributes incoming traffic** across multiple targets (EC2, Lambda, IPs, containers) in one or more Availability Zones.

## Three Types

| Feature | ALB (Application) | NLB (Network) | CLB (Classic) |
|---------|-------------------|---------------|---------------|
| OSI Layer | 7 (HTTP/HTTPS/gRPC) | 4 (TCP/UDP/TLS) | 4/7 (legacy) |
| Protocols | HTTP, HTTPS, gRPC | TCP, UDP, TLS | HTTP, HTTPS, TCP, SSL |
| Target types | Instances, IPs, Lambda, ALB | Instances, IPs, ALB | Instances only |
| Stickiness | Cookie-based | Source IP | Cookie-based |
| WebSockets | Yes | Yes | No |
| Path-based routing | Yes | No | No |
| Host-based routing | Yes | No | No |
| Fixed IP | No | Yes (per AZ) | No |
| Slow start | Yes | No | Yes |
| **Price** | ~$0.0225/hr + LCU | ~$0.0225/hr + NLCU | ~$0.025/hr |
| **Best for** | HTTP apps, microservices | TCP/UDP apps, game servers, private link | Legacy apps |

**Recommendation**: Use ALB for HTTP/HTTPS. Use NLB for TCP/UDP. **Avoid CLB** (legacy, less features).

## ALB — Application Load Balancer

### Components

- **Listener**: listens on a port (e.g., 443) with a protocol
- **Rules**: conditions + actions (forward to target group, redirect, fixed response)
- **Target Group**: group of targets (EC2, IPs, Lambda) with health checks
- **Listener rules**: path-based (`/api/*`), host-based (`api.example.com`), HTTP methods

### Create ALB

```bash
# 1. Create target group
aws elbv2 create-target-group \
  --name web-tg \
  --protocol HTTP \
  --port 80 \
  --vpc-id vpc-12345678 \
  --health-check-protocol HTTP \
  --health-check-path /health

# 2. Register targets
aws elbv2 register-targets \
  --target-group-arn arn:aws:elasticloadbalancing:us-east-1:xxx:targetgroup/web-tg/xxx \
  --targets Id=i-12345678 Id=i-87654321

# 3. Create ALB Security Group
aws ec2 create-security-group \
  --group-name alb-sg \
  --description "ALB security group" \
  --vpc-id vpc-12345678

aws ec2 authorize-security-group-ingress \
  --group-id sg-alb \
  --protocol tcp \
  --port 443 \
  --cidr 0.0.0.0/0

# 4. Create ALB
aws elbv2 create-load-balancer \
  --name web-alb \
  --subnets subnet-1a subnet-1b \
  --security-groups sg-alb \
  --type application

# 5. Create listener
aws elbv2 create-listener \
  --load-balancer-arn arn:aws:elasticloadbalancing:us-east-1:xxx:loadbalancer/app/web-alb/xxx \
  --protocol HTTPS \
  --port 443 \
  --certificates CertificateArn=arn:aws:acm:us-east-1:xxx:certificate/xxx \
  --default-actions Type=forward,TargetGroupArn=arn:aws:elasticloadbalancing:us-east-1:xxx:targetgroup/web-tg/xxx
```

### Path-Based Routing

```bash
# Add listener rule: /api/* → api-tg
aws elbv2 create-rule \
  --listener-arn arn:aws:elasticloadbalancing:us-east-1:xxx:listener/app/web-alb/xxx/xxx \
  --priority 10 \
  --conditions Field=path-pattern,Values=['/api/*'] \
  --actions Type=forward,TargetGroupArn=arn:aws:elasticloadbalancing:us-east-1:xxx:targetgroup/api-tg/xxx
```

### Host-Based Routing

```bash
# Route api.example.com → api-tg
aws elbv2 create-rule \
  --listener-arn arn:aws:elasticloadbalancing:us-east-1:xxx:listener/app/web-alb/xxx/xxx \
  --priority 20 \
  --conditions Field=host-header,Values=['api.example.com'] \
  --actions Type=forward,TargetGroupArn=arn:aws:elasticloadbalancing:us-east-1:xxx:targetgroup/api-tg/xxx
```

### Redirect (HTTP → HTTPS)

```bash
aws elbv2 create-listener \
  --load-balancer-arn arn:aws:elasticloadbalancing:us-east-1:xxx:loadbalancer/app/web-alb/xxx \
  --protocol HTTP --port 80 \
  --default-actions Type=redirect,RedirectConfig='{Protocol=HTTPS,Port=443,StatusCode=HTTP_301}'
```

### Fixed Response

```bash
aws elbv2 create-listener \
  --load-balancer-arn arn:aws:elasticloadbalancing:us-east-1:xxx:loadbalancer/app/web-alb/xxx \
  --protocol HTTP --port 80 \
  --default-actions Type=fixed-response,FixedResponseConfig='{ContentType="text/plain",MessageBody="Service Unavailable",StatusCode="503"}'
```

## NLB — Network Load Balancer

### Key Features
- **Preserves source IP** — backend sees the real client IP
- **Fixed IP** — one static IP per AZ (good for firewall whitelisting)
- **Ultra-low latency** (~100 microseconds)
- **Handles millions of requests/sec**
- **TLS termination** (if needed)

### Create NLB

```bash
# 1. Create target group (TCP)
aws elbv2 create-target-group \
  --name nlb-tg \
  --protocol TCP \
  --port 443 \
  --vpc-id vpc-12345678 \
  --target-type ip

# 2. Create NLB
aws elbv2 create-load-balancer \
  --name nlb-internal \
  --type network \
  --subnets subnet-1a subnet-1b \
  --scheme internal

# 3. Create listener
aws elbv2 create-listener \
  --load-balancer-arn arn:aws:elasticloadbalancing:us-east-1:xxx:loadbalancer/net/nlb-internal/xxx \
  --protocol TCP --port 443 \
  --default-actions Type=forward,TargetGroupArn=arn:aws:elasticloadbalancing:us-east-1:xxx:targetgroup/nlb-tg/xxx
```

### NLB with Elastic IP (for whitelisting)

```bash
# Create NLB in public subnet with EIP
aws ec2 allocate-address --domain vpc

aws elbv2 create-load-balancer \
  --name nlb-public \
  --type network \
  --subnets subnet-1a subnet-1b \
  --scheme internet-facing \
  --ip-address-type ipv4
```

## Cross-Zone Load Balancing

Distributes traffic evenly across all AZs (not just round-robin per AZ).

```bash
# Enable for ALB
aws elbv2 modify-target-group-attributes \
  --target-group-arn arn:aws:elasticloadbalancing:us-east-1:xxx:targetgroup/web-tg/xxx \
  --attributes Key=load_balancing.cross_zone.enabled,Value=true

# Enable for NLB
aws elbv2 modify-load-balancer-attributes \
  --load-balancer-arn arn:aws:elasticloadbalancing:us-east-1:xxx:loadbalancer/net/nlb-internal/xxx \
  --attributes Key=load_balancing.cross_zone.enabled,Value=true
```

## Connection Draining (Deregistration Delay)

Time the LB waits for in-flight requests to complete before de-registering a target.

```bash
# Default: 300 seconds
aws elbv2 modify-target-group-attributes \
  --target-group-arn arn:aws:elasticloadbalancing:us-east-1:xxx:targetgroup/web-tg/xxx \
  --attributes Key=deregistration_delay.timeout_seconds,Value=120
```

## Stickiness (Session Affinity)

```bash
# ALB — cookie-based
aws elbv2 modify-target-group-attributes \
  --target-group-arn arn:aws:elasticloadbalancing:us-east-1:xxx:targetgroup/web-tg/xxx \
  --attributes \
    Key=stickiness.enabled,Value=true \
    Key=stickiness.type,Value=app_cookie \
    Key=stickiness.app_cookie.cookie_name,Value=AWSALB

# NLB — source IP based
aws elbv2 modify-target-group-attributes \
  --target-group-arn arn:aws:elasticloadbalancing:us-east-1:xxx:targetgroup/nlb-tg/xxx \
  --attributes \
    Key=stickiness.enabled,Value=true \
    Key=stickiness.type,Value=source_ip
```

## Health Checks

```bash
aws elbv2 configure-health-check \
  --target-group-arn arn:aws:elasticloadbalancing:us-east-1:xxx:targetgroup/web-tg/xxx \
  --health-check-protocol HTTP \
  --health-check-port 80 \
  --health-check-path /health \
  --health-check-interval-seconds 30 \
  --healthy-threshold-count 3 \
  --unhealthy-threshold-count 3 \
  --health-check-timeout-seconds 5
```

## CLI Commands

```bash
# Load Balancer
aws elbv2 create-load-balancer
aws elbv2 describe-load-balancers
aws elbv2 delete-load-balancer

# Target Groups
aws elbv2 create-target-group
aws elbv2 describe-target-groups
aws elbv2 delete-target-group
aws elbv2 register-targets
aws elbv2 deregister-targets

# Listeners & Rules
aws elbv2 create-listener
aws elbv2 create-rule
aws elbv2 describe-rules

# Attributes
aws elbv2 modify-load-balancer-attributes
aws elbv2 modify-target-group-attributes
```

## Access Logs

```bash
# Enable access logs (store in S3)
aws elbv2 modify-load-balancer-attributes \
  --load-balancer-arn arn:aws:elasticloadbalancing:us-east-1:xxx:loadbalancer/app/web-alb/xxx \
  --attributes \
    Key=access_logs.s3.enabled,Value=true \
    Key=access_logs.s3.bucket,Value=my-elb-logs \
    Key=access_logs.s3.prefix,Value=ALB
```

## Console

**EC2 → Load Balancers** — create and manage
**EC2 → Target Groups** — manage target groups

## Gotchas

- **ALB requires at least 2 subnets in different AZs**
- **NLB with EIP** — each AZ gets its own EIP
- **ALB DNS name changes** — use Route 53 alias to resolve it
- **Security Groups**: ALB supports SGs, NLB does not (use NACL instead)
- **Idle timeout**: ALB default 60 seconds (configurable)
- **Slow start**: ALB allows targets to warm up before sending full traffic
- **Client IP preservation**: NLB preserves by default; ALB does not (unless using Proxy Protocol v2)
- **NLB can handle a static IP address** — important for firewall whitelisting
