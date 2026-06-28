# Day 16 — CloudFront (CDN)

## Concept

CloudFront is a **content delivery network (CDN)** that caches content at **400+ edge locations** worldwide and delivers it to users with low latency.

## Architecture

```
User (Tokyo)
  │
  ▼
Edge Location (Tokyo) ── Miss ──► CloudFront
  │                                    │
  ├── Hit: Serve cached content         │
  └── Miss: Fetch from origin           │
                                        ▼
                                   Origin Server
                                   (S3, ALB, HTTP)
```

## Key Components

### Distribution
The CloudFront "instance" — defines how content is delivered.

- **Web distribution**: for HTTP/HTTPS content
- **RTMP distribution**: for streaming media (legacy — use MediaLive now)

### Origins
Where CloudFront fetches content when there's a **cache miss**.

| Origin Type | Use Case |
|-------------|----------|
| S3 bucket | Static files, images, downloads |
| ALB/EC2 | Dynamic content, API |
| Custom HTTP | Any web server on any cloud |
| MediaStore | Video streaming |
| S3 + OAI (Origin Access Identity) | Restrict S3 access to only CloudFront |

### Behaviors
Path-based rules for how CloudFront handles different URL patterns:

- `/images/*` → cache for 1 year
- `/api/*` → no cache, forward all headers
- `/*` → cache for 24 hours

### Edge Locations
- **400+ PoPs** in 90+ cities
- **Regional Edge Caches**: larger cache between origin and edge (adds another layer)

## Create a Distribution

### S3 + CloudFront (Static Website)

```bash
# 1. Create S3 bucket (private)
aws s3api create-bucket --bucket my-static-site --region us-east-1

# 2. Create CloudFront Origin Access Identity (OAI)
OAI_ID=$(aws cloudfront create-cloud-front-origin-access-identity \
  --cloud-front-origin-access-identity-config \
    CallerReference=$(date +%s),Comment="OAI for my static site" \
  --query 'CloudFrontOriginAccessIdentity.Id' --output text)

# 3. Create distribution
aws cloudfront create-distribution \
  --distribution-config '{
    "CallerReference": "'$(date +%s)'",
    "Aliases": {"Quantity": 0},
    "DefaultRootObject": "index.html",
    "Origins": {
      "Quantity": 1,
      "Items": [{
        "Id": "S3Origin",
        "DomainName": "my-static-site.s3.us-east-1.amazonaws.com",
        "S3OriginConfig": {
          "OriginAccessIdentity": "origin-access-identity/cloudfront/'$OAI_ID'"
        }
      }]
    },
    "DefaultCacheBehavior": {
      "TargetOriginId": "S3Origin",
      "ViewerProtocolPolicy": "redirect-to-https",
      "AllowedMethods": {
        "Quantity": 2,
        "Items": ["GET", "HEAD"],
        "CachedMethods": {"Quantity": 2, "Items": ["GET", "HEAD"]}
      },
      "ForwardedValues": {
        "QueryString": false,
        "Cookies": {"Forward": "none"}
      },
      "MinTTL": 0,
      "DefaultTTL": 86400,
      "MaxTTL": 31536000
    },
    "PriceClass": "PriceClass_100",
    "Enabled": true
  }'
```

### ALB Origin

```bash
aws cloudfront create-distribution \
  --distribution-config '{
    "CallerReference": "'$(date +%s)'",
    "Aliases": {"Quantity": 1, "Items": ["app.example.com"]},
    "Origins": {
      "Quantity": 1,
      "Items": [{
        "Id": "ALBOrigin",
        "DomainName": "my-alb-xxx.us-east-1.elb.amazonaws.com",
        "CustomOriginConfig": {
          "HTTPPort": 80,
          "HTTPSPort": 443,
          "OriginProtocolPolicy": "https-only",
          "OriginSslProtocols": {"Quantity": 1, "Items": ["TLSv1.2"]}
        }
      }]
    },
    "DefaultCacheBehavior": {
      "TargetOriginId": "ALBOrigin",
      "ViewerProtocolPolicy": "redirect-to-https",
      "AllowedMethods": {
        "Quantity": 7,
        "Items": ["GET","HEAD","OPTIONS","PUT","POST","PATCH","DELETE"],
        "CachedMethods": {"Quantity": 2, "Items": ["GET","HEAD"]}
      },
      "DefaultTTL": 0,
      "MaxTTL": 0,
      "MinTTL": 0,
      "ForwardedValues": {
        "QueryString": true,
        "Cookies": {"Forward": "all"},
        "Headers": {"Quantity": 2, "Items": ["Authorization", "Origin"]}
      }
    },
    "PriceClass": "PriceClass_All",
    "Enabled": true,
    "ViewerCertificate": {
      "ACMCertificateArn": "arn:aws:acm:us-east-1:xxx:certificate/xxx",
      "SSLSupportMethod": "sni-only"
    },
    "Aliases": {"Quantity": 1, "Items": ["app.example.com"]}
  }'
```

## Caching & TTL

| Setting | Description |
|---------|-------------|
| **Min TTL** | Minimum cache duration |
| **Default TTL** | Used when origin doesn't send Cache-Control |
| **Max TTL** | Maximum cache duration |
| **Cache-Control** headers from origin | Override default TTL |
| **Invalidation** | Force cache clear (cost: free for first 1000 paths/month) |

### Cache Invalidation

```bash
aws cloudfront create-invalidation \
  --distribution-id E1234567890ABCD \
  --paths "/*" "/images/*" "/index.html"
```

## Security

### Origin Access Identity (OAI)
Restricts S3 bucket to only be readable by CloudFront:

```bash
# S3 bucket policy
aws s3api put-bucket-policy \
  --bucket my-static-site \
  --policy '{
    "Version": "2012-10-17",
    "Statement": [{
      "Effect": "Allow",
      "Principal": {"AWS": "arn:aws:iam::cloudfront:user/CloudFront Origin Access Identity '$OAI_ID'"},
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::my-static-site/*"
    }]
  }'
```

### Origin Access Control (OAC) — Newer than OAI
Recommanded for S3 origins:

```bash
aws cloudfront create-origin-access-control \
  --origin-access-control-config '{
    "Name": "my-oac",
    "Description": "OAC for S3",
    "SigningProtocol": "sigv4",
    "SigningBehavior": "always"
  }'
```

### WAF Integration
Attach a WAF web ACL to CloudFront:

```bash
aws cloudfront update-distribution \
  --id E1234567890ABCD \
  --distribution-config '... "WebACLId": "arn:aws:wafv2:us-east-1:xxx:global/webacl/my-waf/xxx" ...'
```

### Geo-Restriction
Block or allow countries:

```bash
aws cloudfront update-distribution \
  --distribution-config '...
    "Restrictions": {
      "GeoRestriction": {
        "RestrictionType": "blacklist",
        "Quantity": 2,
        "Items": ["CN", "RU"]
      }
    }
  ...'
```

## Signed URLs & Cookies (Premium Content)

```bash
# Create key pair for CloudFront signed URLs
openssl genrsa -out private_key.pem 2048
openssl rsa -pubout -in private_key.pem -out public_key.pem

# Upload public key to CloudFront
aws cloudfront create-public-key \
  --public-key-config '{
    "CallerReference": "key1",
    "Name": "my-signing-key",
    "EncodedKey": "'$(cat public_key.pem | base64 -w0)'"
  }'

# Require signed URLs in distribution behavior
# (Or use Lambda@Edge for custom auth)
```

## Custom Error Responses

```bash
aws cloudfront update-distribution \
  --distribution-config '...
    "CustomErrorResponses": {
      "Quantity": 1,
      "Items": [{
        "ErrorCode": 404,
        "ResponsePagePath": "/404.html",
        "ResponseCode": "404",
        "ErrorCachingMinTTL": 300
      }]
    }
  ...'
```

## CLI Commands

```bash
aws cloudfront create-distribution
aws cloudfront get-distribution
aws cloudfront update-distribution
aws cloudfront delete-distribution
aws cloudfront list-distributions
aws cloudfront create-invalidation
aws cloudfront list-invalidations
aws cloudfront create-origin-access-control
aws cloudfront create-public-key
```

## Routing & Design Notes

### DNS
- CloudFront gives you a domain: `d123.cloudfront.net`
- Use Route 53 ALIAS record to point your custom domain to it
- CloudFront accepts SSL certificates from ACM (must be in **us-east-1**)

### Origin Selection
- Use multiple origins with **cache behaviors** to split traffic
- `/api/*` → ALB (no cache), rest → S3 (heavy cache)

## Console

**CloudFront → Distributions** — create, update, invalidate
**CloudFront → Origin Access Controls** — manage OACs
**CloudFront → Public Keys** — for signed URLs

## Gotchas

- **ACM certificates must be in us-east-1** (even if your app is in another region)
- **First byte latency**: can be high if origin is far from edge
- **Cache invalidation**: first 1000 paths/month free, $0.005/path after
- **Minimum TTL**: set appropriately — too short defeats caching, too long means stale content
- **Origin failover**: configure primary and secondary origins
- **Real-time logs**: can stream to Kinesis Data Streams
- **Edge Functions**: Lambda@Edge (up to 5s, Node.js/Python) or CloudFront Functions (up to 100µs, JavaScript)
- **WebSocket**: CloudFront supports WebSocket since 2021
- **Price classes**: `PriceClass_100` (US+Europe only), `PriceClass_200` (US+Europe+Asia), `PriceClass_All` (worldwide)
