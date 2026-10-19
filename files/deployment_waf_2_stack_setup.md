# Deployment Guide — Shopme Static Website (S3 + CloudFront + OAC + WAF)

This guide walks through deploying two CloudFormation stacks:

1. **`shopme-waf.yaml`** — a standalone WAF Web ACL, which must be deployed
   in **us-east-1**
2. **`shopme-s3-website.yaml`** — the main stack (S3 bucket + CloudFront +
   OAC), deployed in **your normal region** (e.g. `eu-north-1`), which
   optionally attaches the WAF Web ACL from step 1

## Why two stacks?

WAF Web ACLs with `Scope: CLOUDFRONT` can only be **created** via the
us-east-1 API endpoint — this is a hard AWS requirement, unrelated to where
your other resources live. If your main stack normally deploys to a
different region (this project uses `eu-north-1`), CloudFormation can't
create a `Scope: CLOUDFRONT` Web ACL inside that same stack. Splitting it
into its own us-east-1 stack, and passing its ARN into the main stack as a
plain parameter, avoids the region conflict entirely.

## What each template creates

**`shopme-waf.yaml`** (deploy to us-east-1):
- An AWS WAF Web ACL with the AWS Managed Rule Group `AWSManagedRulesSQLiRuleSet`
  (SQL injection protection)

**`shopme-s3-website.yaml`** (deploy to your normal region):
- An S3 bucket (default name `shopme-website-123456789012`) with static
  website hosting configuration enabled, but with **all public access
  blocked**
- A CloudFront distribution that serves the site over HTTPS
- An Origin Access Control (OAC) that lets only this CloudFront distribution
  read objects from the bucket
- A bucket policy scoped to that one distribution — no other principal can
  read the bucket's contents
- **(Optional, on by default)** attaches the WAF Web ACL from the first
  stack, via its ARN passed in as a parameter

> **Note:** the S3 website-hosting feature (index/error document config)
> stays enabled on the bucket, but it's no longer the way visitors reach
> the site — CloudFront is. The S3 website endpoint doesn't support the
> SigV4 signing that OAC requires, so CloudFront connects to the bucket's
> regular REST endpoint instead. The direct `http://...s3-website...`
> URL will return Access Denied, which is expected.

## Prerequisites

- An AWS account
- [AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html) installed and configured (`aws configure`) with credentials that have permission to create S3 buckets, bucket policies, CloudFront distributions, WAF Web ACLs, and CloudFormation stacks
- The three files from this project in the same folder:
  - `shopme-waf.yaml`
  - `shopme-s3-website.yaml`
  - `index.html`

## Step 1 — Choose a bucket name

S3 bucket names must be **globally unique** across all AWS accounts, lowercase,
and contain only letters, numbers, dots, and hyphens. The template defaults
to `shopme-website-123456789012`; override it with `--parameter-overrides`
if you need a different name.

## Step 2 — Decide whether to enable WAF

WAF is controlled by the `EnableWAF` parameter on the main stack (default
`"true"`). If enabled, it requires a `WebACLArn` value from the separately
deployed WAF stack (Step 3 below).

**WAF is not free.** As of writing, AWS WAF pricing is roughly:
- $5.00/month per Web ACL
- $1.00/month per rule (each AWS Managed Rule Group counts as one)
- $0.60 per million requests inspected

So enabling WAF costs roughly **$6/month baseline** plus request-based
charges. Set `EnableWAF=false` on the main stack (and skip Step 3 entirely)
if you want to avoid this cost. Always check current pricing at
[aws.amazon.com/waf/pricing](https://aws.amazon.com/waf/pricing) before
deploying, since AWS updates pricing periodically.

## Step 3 — Deploy the WAF stack (us-east-1) — skip if not using WAF

```bash
aws cloudformation deploy \
  --template-file shopme-waf.yaml \
  --stack-name shopme-waf \
  --region us-east-1
```

Then retrieve its ARN:

```bash
aws cloudformation describe-stacks \
  --stack-name shopme-waf \
  --region us-east-1 \
  --query "Stacks[0].Outputs[?OutputKey=='WebACLArn'].OutputValue" \
  --output text
```

Copy that ARN — you'll pass it into the main stack next.

## Step 4 — Deploy (or update) the main stack

Replace `<your-region>` with wherever this stack normally lives (this
project uses `eu-north-1`), and `<arn-from-step-3>` with the value you just
copied.

**With WAF enabled:**
```bash
aws cloudformation deploy \
  --template-file shopme-s3-website.yaml \
  --stack-name shopme-website \
  --parameter-overrides BucketName=shopme-website-123456789012 WebACLArn=<arn-from-step-3> \
  --capabilities CAPABILITY_IAM \
  --region <your-region>
```
aws cloudformation deploy \
  --template-file shopme-s3-website.yaml \
  --stack-name shopme-website \
  --parameter-overrides BucketName=shopme-website-123456789012 WebACLArn=arn:aws:wafv2:us-east-1:<ACCOUNT_ID>:global/webacl/shopme-website-waf/f8dee533-b41d-4a7a-84c6-f8504c30ed38 \
  --capabilities CAPABILITY_IAM \
  --region eu-north-1
  
**Without WAF:**
```bash
aws cloudformation deploy \
  --template-file shopme-s3-website.yaml \
  --stack-name shopme-website \
  --parameter-overrides BucketName=shopme-website-123456789012 EnableWAF=false \
  --capabilities CAPABILITY_IAM \
  --region <your-region>
```

If you already have the `shopme-website` stack from a previous deployment,
this same command **updates it in place** — no need to delete and recreate.

Deployment (or update) typically takes a few minutes; CloudFront distribution
changes themselves can take 5–15 minutes to fully propagate globally.

**Important:** always deploy the main stack to the same region it was
originally created in. Passing a different `--region` (or omitting it and
relying on a changed CLI default) makes CloudFormation treat it as a
different stack, and it will try to create resources — like the S3 bucket —
that already exist elsewhere under your account, causing a conflict. If
you're not sure which region your stack is in, check with:

```bash
aws configure get region
```

(assuming you haven't been overriding it with `--region` on prior deploys).

## Step 5 — Upload the website file to the bucket

```bash
aws s3 cp index.html s3://shopme-website-123456789012/index.html --content-type text/html
```

Re-run this command any time you update `index.html` to push a new version.

## Step 6 — Get the CloudFront URL

Either run (using your main stack's actual region):

```bash
aws cloudformation describe-stacks \
  --stack-name shopme-website \
  --region <your-region> \
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

## Step 7 — Open the site

Visit the `CloudFrontURL` from the previous step in your browser. The Shopme
site should load over HTTPS with search and cart functionality working
entirely client-side.

## Custom domain (optional)

If you're pointing a domain like `shop.yourdomain.com` at this distribution,
see the separate `custom-domain-setup.md` guide for the ACM certificate and
DNS record steps.

## Updating the site later

1. Re-run the Step 5 upload command to push the new `index.html` to S3.
2. **Invalidate the CloudFront cache** so visitors see the update immediately,
   instead of waiting for the cached version to expire:

   ```bash
   aws cloudformation describe-stacks \
     --stack-name shopme-website \
     --region <your-region> \
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

- **Turning WAF on**: deploy `shopme-waf.yaml` (Step 3) if you haven't
  already, then re-run the main stack deploy (Step 4) with
  `EnableWAF=true` and the correct `WebACLArn`.
- **Turning WAF off**: re-run the main stack deploy with `EnableWAF=false`
  — the `WebACLArn` parameter is ignored in this case. You can also delete
  the `shopme-waf` stack separately in us-east-1 if you no longer need it:

  ```bash
  aws cloudformation delete-stack --stack-name shopme-waf --region us-east-1
  ```

  (Only do this after confirming no other stack still references its ARN.)

## Tearing down

CloudFront distributions must be **disabled** before they can be deleted,
and CloudFormation handles this automatically on stack deletion — but it
can take several minutes.

```bash
aws s3 rm s3://shopme-website-123456789012 --recursive
aws cloudformation delete-stack --stack-name shopme-website --region <your-region>
aws cloudformation wait stack-delete-complete --stack-name shopme-website --region <your-region>

# If you also want to remove the WAF Web ACL:
aws cloudformation delete-stack --stack-name shopme-waf --region us-east-1
aws cloudformation wait stack-delete-complete --stack-name shopme-waf --region us-east-1
```

The bucket must be emptied first — CloudFormation won't delete a non-empty
S3 bucket.

## Troubleshooting

| Issue | Likely cause |
|---|---|
| `BucketAlreadyExists` error on deploy | The bucket name is already taken globally — choose a different name |
| `AWS::EarlyValidation::ResourceExistenceCheck` failure | You deployed the main stack with a different `--region` than it was originally created in — CloudFormation treated it as a new stack and tried to create a bucket that already exists elsewhere. Redeploy using the stack's actual original region (check with `aws configure get region`) |
| 403 Forbidden when visiting the **S3 website URL** | Expected — public access is now blocked. Use the `CloudFrontURL` output instead |
| 403 Forbidden when visiting the **CloudFront URL** | The bucket policy's `AWS:SourceArn` condition may not match the distribution yet — confirm the stack finished updating, or that `index.html` was actually uploaded |
| Changes not showing up after re-upload | CloudFront is serving a cached copy — create an invalidation (see "Updating the site later") or wait for cache TTL to expire |
| Stack update stuck / slow | CloudFront distribution changes can take 5–15 minutes to propagate — this is normal AWS behavior, not a failure |
| `AccessDenied` calling `create-invalidation` | Your IAM credentials need `cloudfront:CreateInvalidation` permission |
| WAF stack fails to create | Confirm you're deploying `shopme-waf.yaml` with `--region us-east-1` — `Scope: CLOUDFRONT` Web ACLs can only be created there |
| Main stack fails with a missing/invalid `WebACLArn` | Make sure `EnableWAF=true` is paired with a real `WebACLArn` value from the `shopme-waf` stack's output — an empty or malformed ARN will fail validation |
| Legitimate requests being blocked (403 from WAF) | The managed SQLi rule set may be flagging a false positive — check the Web ACL's sampled requests in the WAF console (us-east-1), or temporarily set `EnableWAF=false` on the main stack to confirm WAF is the cause |
| Unexpected AWS bill increase | Check whether the WAF stack is still deployed and attached — WAF has ongoing per-month and per-request charges even with no incidents |
