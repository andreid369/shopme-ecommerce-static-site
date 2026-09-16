# Deployment Guide — Shopme Static Website (S3 + CloudFront + OAC + WAF)

This guide walks through deploying the `shopme-s3-website.yaml` CloudFormation
template and publishing `index.html` to the resulting S3 bucket, served
through a CloudFront distribution secured with Origin Access Control (OAC)
and, optionally, an AWS WAF Web ACL for SQL injection protection.

## What this template creates

- An S3 bucket (default name `shopme-website-123456789012`) with static
  website hosting configuration enabled, but with **all public access
  blocked**
- A CloudFront distribution that serves the site over HTTPS
- An Origin Access Control (OAC) that lets only this CloudFront distribution
  read objects from the bucket
- A bucket policy scoped to that one distribution — no other principal can
  read the bucket's contents
- **(Optional, on by default)** an AWS WAF Web ACL with the AWS Managed
  Rule Group `AWSManagedRulesSQLiRuleSet`, attached to the distribution

> **Note:** the S3 website-hosting feature (index/error document config)
> stays enabled on the bucket, but it's no longer the way visitors reach
> the site — CloudFront is. The S3 website endpoint doesn't support the
> SigV4 signing that OAC requires, so CloudFront connects to the bucket's
> regular REST endpoint instead. The direct `http://...s3-website...`
> URL will return Access Denied, which is expected.

## Prerequisites

- An AWS account
- [AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html) installed and configured (`aws configure`) with credentials that have permission to create S3 buckets, bucket policies, CloudFront distributions, WAF Web ACLs, and CloudFormation stacks
- The two files from this project in the same folder:
  - `shopme-s3-website.yaml`
  - `index.html`

## Step 1 — Choose a bucket name

S3 bucket names must be **globally unique** across all AWS accounts, lowercase,
and contain only letters, numbers, dots, and hyphens. This template defaults
to `shopme-website-123456789012`; override it with `--parameter-overrides`
if you need a different name.

## Step 2 — Decide whether to enable WAF

The template includes a parameter, `EnableWAF` (default `"true"`), that
controls whether the WAF Web ACL is created at all.

**WAF is not free.** As of writing, AWS WAF pricing is roughly:
- $5.00/month per Web ACL
- $1.00/month per rule (each AWS Managed Rule Group counts as one)
- $0.60 per million requests inspected

So leaving `EnableWAF=true` costs roughly **$6/month baseline** plus
request-based charges. Set `EnableWAF=false` in the deploy command if you
want to skip this cost. Always check current pricing at
[aws.amazon.com/waf/pricing](https://aws.amazon.com/waf/pricing) before
deploying, since AWS updates pricing periodically.

**Important:** WAF Web ACLs with `Scope: CLOUDFRONT` can only be created in
the **us-east-1** region, regardless of where your other AWS resources
normally live. The deploy command below already specifies `--region us-east-1`
for this reason — don't omit it, or the stack will fail if WAF is enabled.

## Step 3 — Deploy (or update) the CloudFormation stack

From a terminal, in the folder containing `shopme-s3-website.yaml`, run one
of the following.

**With WAF enabled (default):**
```bash
aws cloudformation deploy \
  --template-file shopme-s3-website.yaml \
  --stack-name shopme-website \
  --parameter-overrides BucketName=shopme-website-123456789012 \
  --capabilities CAPABILITY_IAM \
  --region us-east-1
```

**Without WAF (skip the added cost):**
```bash
aws cloudformation deploy \
  --template-file shopme-s3-website.yaml \
  --stack-name shopme-website \
  --parameter-overrides BucketName=shopme-website-123456789012 EnableWAF=false \
  --capabilities CAPABILITY_IAM \
  --region us-east-1
```

If you already have the `shopme-website` stack from a previous deployment,
this same command **updates it in place** — no need to delete and recreate.

Deployment (or update) typically takes a few minutes; CloudFront distribution
changes themselves can take 5–15 minutes to fully propagate globally.

## Step 4 — Upload the website file to the bucket

```bash
aws s3 cp index.html s3://shopme-website-123456789012/index.html --content-type text/html
```

Re-run this command any time you update `index.html` to push a new version.

## Step 5 — Get the CloudFront URL

Either run:

```bash
aws cloudformation describe-stacks \
  --stack-name shopme-website \
  --region us-east-1 \
  --query "Stacks[0].Outputs"
```

or open the **CloudFormation console → shopme-website stack → Outputs tab**.

Look for the `CloudFrontURL` output — that's your live site address, e.g.:

```
https://d21m7g1cjhobyi.cloudfront.net
```

The `S3WebsiteURL` output is still shown for reference but will no longer
work directly, since the bucket now blocks all public access. If WAF is
enabled, a `WebACLArn` output will also be present (omitted if `EnableWAF=false`).

## Step 6 — Open the site

Visit the `CloudFrontURL` from the previous step in your browser. The Shopme
site should load over HTTPS with search and cart functionality working
entirely client-side.

## Custom domain (optional)

If you're pointing a domain like `shop.yourdomain.com` at this distribution,
see the separate `custom-domain-setup.md` guide for the ACM certificate and
DNS record steps.

## Updating the site later

1. Re-run the Step 4 upload command to push the new `index.html` to S3.
2. **Invalidate the CloudFront cache** so visitors see the update immediately,
   instead of waiting for the cached version to expire:

   ```bash
   aws cloudformation describe-stacks \
     --stack-name shopme-website \
     --region us-east-1 \
     --query "Stacks[0].Outputs[?OutputKey=='CloudFrontDistributionId'].OutputValue" \
     --output text
   ```

   Then:

   ```bash
   aws cloudfront create-invalidation \
     --distribution-id <the-id-from-above> \
     --paths "/*"
   ```

No need to redeploy the CloudFormation stack unless you're changing the
infrastructure itself (bucket settings, cache behavior, distribution
settings, WAF toggle, etc.).

## Toggling WAF on/off later

Re-run the Step 3 deploy command with `EnableWAF=true` or `EnableWAF=false`
as needed — CloudFormation will create or remove the Web ACL and detach/attach
it from the distribution accordingly.

## Tearing down

CloudFront distributions must be **disabled** before they can be deleted,
and CloudFormation handles this automatically on stack deletion — but it
can take several minutes.

```bash
aws s3 rm s3://shopme-website-123456789012 --recursive
aws cloudformation delete-stack --stack-name shopme-website --region us-east-1
aws cloudformation wait stack-delete-complete --stack-name shopme-website --region us-east-1
```

The bucket must be emptied first — CloudFormation won't delete a non-empty
S3 bucket.

## Troubleshooting

| Issue | Likely cause |
|---|---|
| `BucketAlreadyExists` error on deploy | The bucket name is already taken globally — choose a different name |
| 403 Forbidden when visiting the **S3 website URL** | Expected — public access is now blocked. Use the `CloudFrontURL` output instead |
| 403 Forbidden when visiting the **CloudFront URL** | The bucket policy's `AWS:SourceArn` condition may not match the distribution yet — confirm the stack finished updating, or that `index.html` was actually uploaded |
| Changes not showing up after re-upload | CloudFront is serving a cached copy — create an invalidation (see "Updating the site later") or wait for cache TTL to expire |
| Stack update stuck / slow | CloudFront distribution changes can take 5–15 minutes to propagate — this is normal AWS behavior, not a failure |
| `AccessDenied` calling `create-invalidation` | Your IAM credentials need `cloudfront:CreateInvalidation` permission |
| WAF resource fails to create | Confirm you're deploying with `--region us-east-1` — `Scope: CLOUDFRONT` Web ACLs can only be created there |
| Legitimate requests being blocked (403 from WAF) | The managed SQLi rule set may be flagging a false positive — check the Web ACL's sampled requests in the WAF console, or temporarily set `EnableWAF=false` to confirm WAF is the cause |
| Unexpected AWS bill increase | Check whether `EnableWAF=true` is set — WAF has ongoing per-month and per-request charges even with no incidents |
