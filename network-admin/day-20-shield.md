# Day 18 — AWS Shield (DDoS Protection)

## Concept

AWS Shield provides **DDoS protection** for your applications. Two tiers:

| Tier | Cost | Protection |
|------|------|------------|
| **Shield Standard** | **Free** | Automatic — protects all AWS customers |
| **Shield Advanced** | $3,000/month + data egress | Enhanced detection, mitigation, DDoS cost protection, 24x7 support |

## Shield Standard (Free, Automatic)

Every AWS customer gets Shield Standard automatically. It protects against:

| Attack Type | Example | How It Works |
|-------------|---------|-------------|
| SYN floods | Massive TCP SYN packets | AWS edge blocks before reaching your app |
| UDP floods | Amplification attacks | Absorbed at edge |
| Reflection attacks | NTP/DNS amplification | Blocked at network boundary |
| Layer 3/4 attacks | Bandwidth attacks | AWS infrastructure absorbs |

**No configuration needed** — it's always on.

## Shield Advanced ($3,000/month)

### Additional Protections

| Protection | What It Does |
|------------|-------------|
| **Layer 7 DDoS detection** | WAF integration for HTTP floods |
| **Application-layer DDoS** | Detects and mitigates web app attacks |
| **Near real-time visibility** | DDoS metrics in CloudWatch |
| **DDoS cost protection** | Get credits for scaling costs during attacks |
| **24x7 DDoS Response Team (DRT)** | AWS experts help during attacks |
| **Health-based detection** | Uses Route 53 health checks to distinguish real vs attack traffic |
| **Protection groups** | Group resources for unified management |

### Protected Resources

| Resource | What Shield Advanced Protects |
|----------|-----------------------------|
| CloudFront distributions | Edge-based DDoS + WAF |
| Route 53 hosted zones | DNS query flood protection |
| ALB / NLB / CLB | L3-L7 protection |
| Global Accelerator | Anycast-based mitigation |
| Elastic IPs (EC2, NAT) | L3-L4 protection |

### Enable Shield Advanced

```bash
# Subscribe
aws shield subscribe \
  --protection-group-arn arn:aws:shield:us-east-1:xxx:protection-group/MyGroup

# Add protection to a resource
aws shield create-protection \
  --name "WebALB-Protection" \
  --resource-arn arn:aws:elasticloadbalancing:us-east-1:xxx:loadbalancer/app/web-alb/xxx
```

### Protection Groups

Group resources for unified policy management:

```bash
aws shield create-protection-group \
  --protection-group-id "WebTier" \
  --aggregation MAX \
  --pattern ALL \
  --resource-type CLASSIC
```

Types:
- `ALL` — select all protected resources
- `ARBITRARY` — choose specific resources
- `BY_RESOURCE_TYPE` — all resources of a given type

### Health-Based Detection

Use Route 53 health checks to help Shield distinguish real users from attackers during a DDoS:

```bash
aws shield update-subscription \
  --health-check-arns '["arn:aws:route53:::healthcheck/xxx"]'
```

## Attack Response

### Automated Mitigation

| Attack Vector | Shield Advanced Response |
|---------------|------------------------|
| SYN floods | SYN cookies, rate limiting |
| UDP floods | Source-based throttling |
| DNS floods | DNS query filtering |
| HTTP floods (L7) | WAF rate-based rules (auto-configured) |
| SSL/TLS exhaustion | TCP connection handling |

### Call the DRT (DDoS Response Team)

```bash
# Engage DRT
aws shield engage-drt \
  --log-bucket my-shield-logs \
  --contact-notes "Experiencing HTTP flood attack, need mitigation tuning"
```

### Proactive Engagement (Optional)

Enable proactive engagement — AWS contacts you within 1 minute of detecting a likely DDoS:

```bash
aws shield enable-proactive-engagement
```

## Cost Protection

If you're under DDoS attack and need to scale, Shield Advanced gives **credits** for:

- EC2 Auto Scaling costs incurred during attack
- ELB scaling costs
- CloudFront data transfer overage
- WAF cost overage

## CLI Commands

```bash
aws shield create-protection
aws shield list-protections
aws shield describe-protection
aws shield delete-protection
aws shield create-protection-group
aws shield list-protection-groups
aws shield enable-proactive-engagement
aws shield update-subscription
aws shield engage-drt
```

## Monitoring

### CloudWatch Metrics

| Metric | Description |
|--------|-------------|
| `DDoSDetected` | Attack detected (0 or 1) |
| `DDoSProtectionGroupsBitRate` | Bits per second for a protection group |
| `DDoSProtectionGroupsPacketRate` | Packets per second for a protection group |

### Access Logs

Shield Advanced sends events to S3 (specify during subscription):

## Console

**WAF & Shield → Shield** — subscription, protections, groups
**WAF & Shield → DDoS Event** — attack reports

## Defense in Depth Strategy

```
Layer 7 (Application):  WAF + Shield Advanced
Layer 6/5:              CloudFront, Global Accelerator
Layer 4 (Transport):    NACLs, Shield Standard
Layer 3 (Network):      VPC design, Shield Standard
```

## Gotchas

- **Shield Advanced is $3,000/month per organization** (not per account) — consolidated billing counts as one
- **Data egress charges** during attacks: Shield Advanced may reduce or waive these
- **CloudFront only**: Shield Advanced works at CloudFront edge for L7 attacks
- **WAF required** for Layer 7 protection — Shield alone doesn't do L7
- **Health-based detection** needs Route 53 health checks configured
- **Proactive engagement** requires a CloudWatch dashboard + S3 bucket prepared
- **Cost protection** doesn't apply if you're running without Shield Advanced when the attack starts
- **DRT can't access your account** without explicit authorization — they need a support case
