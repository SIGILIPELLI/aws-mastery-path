# 08 · CloudFront & CDN Basics

The S3 static website from Level 1 (module 4) and the ALB from module 2
both serve every request from one region. **CloudFront** is AWS's CDN: it
caches your content at edge locations worldwide, so repeat requests are
served from a location near the visitor instead of round-tripping to your
origin every time, and it's also the standard way to put free, managed
HTTPS in front of an S3 bucket. This module covers fronting an S3 origin
with CloudFront, cache behavior, and invalidation.

## Core concepts

| Concept | What it is |
|---|---|
| **Distribution** | A CloudFront configuration — one or more origins, cache behaviors, and a global edge presence. |
| **Origin** | Where CloudFront fetches uncached content from — an S3 bucket, an ALB, an API Gateway endpoint. |
| **Origin Access Control (OAC)** | Lets CloudFront authenticate to a *private* S3 bucket, so the bucket itself stays non-public. |
| **Cache behavior** | Per-path-pattern rules: which origin, which headers/query strings/cookies vary the cache key, default/min/max TTL. |
| **Edge location** | A CloudFront point of presence that caches and serves content close to visitors. |
| **Invalidation** | A request to purge specific paths from edge caches before their TTL naturally expires. |

## Prepare a private S3 origin

Unlike the Level 1 S3 website module (public bucket policy), the
CloudFront-recommended pattern keeps the bucket **private** and lets only
CloudFront read it via OAC:

```bash
aws s3 mb s3://training-cf-origin-2026
aws s3 cp ./site/ s3://training-cf-origin-2026/ --recursive

aws cloudfront create-origin-access-control \
  --origin-access-control-config '{
    "Name": "training-oac",
    "OriginAccessControlOriginType": "s3",
    "SigningBehavior": "always",
    "SigningProtocol": "sigv4"
  }'
# OriginAccessControl.Id: E1A2B3C4D5E6F7
```

## Create the distribution

Save this as `distribution-config.json`:

```json
{
  "CallerReference": "training-dist-2026",
  "Comment": "Training CDN in front of S3",
  "Enabled": true,
  "DefaultRootObject": "index.html",
  "Origins": {
    "Quantity": 1,
    "Items": [{
      "Id": "training-s3-origin",
      "DomainName": "training-cf-origin-2026.s3.us-east-1.amazonaws.com",
      "OriginAccessControlId": "E1A2B3C4D5E6F7",
      "S3OriginConfig": { "OriginAccessIdentity": "" }
    }]
  },
  "DefaultCacheBehavior": {
    "TargetOriginId": "training-s3-origin",
    "ViewerProtocolPolicy": "redirect-to-https",
    "CachePolicyId": "658327ea-f89d-4fab-a63d-7e88639e58f6",
    "Compress": true
  }
}
```

```bash
aws cloudfront create-distribution --distribution-config file://distribution-config.json
# Distribution.Id: E2QWRUHAPOMQZL
# Distribution.DomainName: d111111abcdef8.cloudfront.net
# Distribution.Status: InProgress
```

`CachePolicyId: 658327ea-...` is the AWS-managed **CachingOptimized**
policy (a fixed, well-known ID in every account) — it caches based on the
URL path only, ignoring headers/cookies/query strings, the right default
for a static site. `ViewerProtocolPolicy: redirect-to-https` means every
plain-HTTP request gets redirected to HTTPS at the edge, at no extra cost,
using CloudFront's default certificate for `*.cloudfront.net`.

## Restrict the bucket to CloudFront only

```bash
cat > bucket-policy.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": { "Service": "cloudfront.amazonaws.com" },
    "Action": "s3:GetObject",
    "Resource": "arn:aws:s3:::training-cf-origin-2026/*",
    "Condition": {
      "StringEquals": {
        "AWS:SourceArn": "arn:aws:cloudfront::123456789012:distribution/E2QWRUHAPOMQZL"
      }
    }
  }]
}
EOF

aws s3api put-bucket-policy \
  --bucket training-cf-origin-2026 \
  --policy file://bucket-policy.json
```

With OAC and this policy, the bucket has **no public access at all** —
visiting the S3 URL directly returns `AccessDenied`, while the CloudFront
domain serves the content fine. This is a meaningfully better security
posture than Level 1's public-bucket-policy website pattern, at the cost
of losing the bucket's own website-redirect features (error pages,
redirect rules) — which CloudFront can also handle via its own
error-response configuration.

## Wait for deployment and test

```bash
aws cloudfront get-distribution --id E2QWRUHAPOMQZL --query "Distribution.Status"
# "Deployed"   (an initial deployment typically takes several minutes to propagate to all edge locations)

curl -I https://d111111abcdef8.cloudfront.net/
# HTTP/2 200
# x-cache: Miss from cloudfront

curl -I https://d111111abcdef8.cloudfront.net/
# x-cache: Hit from cloudfront
```

The `x-cache` response header is the fastest way to confirm whether
CloudFront served a request from cache (`Hit`) or had to fetch it from
the origin (`Miss`/`RefreshHit`).

## Invalidating the cache after an update

```bash
aws s3 cp ./site/index.html s3://training-cf-origin-2026/index.html

aws cloudfront create-invalidation \
  --distribution-id E2QWRUHAPOMQZL \
  --paths "/index.html"
# Invalidation.Status: InProgress
```

Uploading a new object to S3 does **not** automatically clear edges that
already cached the old version under the same path — an explicit
invalidation (or a versioned filename, e.g. `app.v2.js`, which sidesteps
invalidation entirely) is required to force edges to re-fetch it before
the cache TTL would naturally expire.

## Using a custom domain and your own certificate

```bash
# The certificate MUST be requested in us-east-1 regardless of where
# your other resources live — this is a hard CloudFront requirement.
aws acm request-certificate \
  --domain-name cdn.training.example.com \
  --validation-method DNS \
  --region us-east-1
# CertificateArn: arn:aws:acm:us-east-1:123456789012:certificate/...

aws cloudfront update-distribution \
  --id E2QWRUHAPOMQZL \
  --distribution-config file://distribution-config-with-cert.json
```

After validating the certificate (a Route 53 `CNAME` record, module 3),
add `Aliases` and `ViewerCertificate` referencing the ACM ARN to the
distribution config, then point a Route 53 alias record at the
distribution's `DomainName` — the same alias pattern module 3 used for
the ALB, just with a different `HostedZoneId`.

!!! warning "Distribution changes propagate globally, not instantly"
    Both initial creation and later config updates (`update-distribution`)
    take real time — often 5-15+ minutes — to reach every edge location
    worldwide. `get-distribution` reporting `Status: Deployed` means the
    change has propagated everywhere; testing immediately after
    `create-distribution` returns can hit edges still serving the old
    config or none at all.

## Cheat sheet

| Command | Purpose |
|---|---|
| `aws cloudfront create-origin-access-control` | Create OAC so CloudFront can read a private S3 bucket. |
| `aws cloudfront create-distribution --distribution-config file://F` | Create a CDN distribution. |
| `aws s3api put-bucket-policy` | Restrict the S3 origin to only the CloudFront distribution. |
| `aws cloudfront get-distribution --id ID` | Check deployment status and config. |
| `aws cloudfront create-invalidation --paths "/path"` | Force edges to re-fetch specific paths. |
| `aws acm request-certificate --region us-east-1` | Request an HTTPS certificate (must be `us-east-1` for CloudFront). |
| `aws cloudfront update-distribution` | Change config (origins, aliases, cache behaviors). |

## Cheat sheet: `x-cache` values

| Value | Meaning |
|---|---|
| `Miss from cloudfront` | Not cached at this edge; fetched from origin. |
| `Hit from cloudfront` | Served from edge cache. |
| `RefreshHit from cloudfront` | Cached copy was stale; revalidated with origin. |
| `Error from cloudfront` | Origin returned an error, or CloudFront couldn't reach it. |

## Exercise

1. Upload a small static site to a new, private S3 bucket (block all
   public access).
2. Create an OAC and a CloudFront distribution using that bucket as origin
   with the `CachingOptimized` managed policy.
3. Apply a bucket policy allowing only that distribution's ARN to
   `GetObject`, and confirm direct S3 access is denied while the
   CloudFront domain works.
4. Curl the CloudFront domain twice and confirm `x-cache` goes from
   `Miss` to `Hit`.
5. Update `index.html` in S3, invalidate `/index.html`, and confirm (after
   the invalidation completes) the new content is served.
6. Delete the distribution (disable it first, then delete once
   `Status: Deployed` shows it's disabled) and the bucket when finished.
