# Deployment Guide — Shopme Static Website (S3 + CloudFront + OAC)

This guide walks through deploying the `shopme-s3-website.yaml` CloudFormation
template and publishing `index.html` to the resulting S3 bucket, now served
through a CloudFront distribution secured with Origin Access Control (OAC).

## What this template creates

- An S3 bucket (`shopme-website-123456789012`) with static website hosting
  configuration enabled, but with **all public access blocked**
- A CloudFront distribution that serves the site over HTTPS
- An Origin Access Control (OAC) that lets only this CloudFront distribution
  read objects from the bucket
- A bucket policy scoped to that one distribution — no other principal can
  read the bucket's contents

> **Note:** the S3 website-hosting feature (index/error document config)
> stays enabled on the bucket, but it's no longer the way visitors reach
> the site — CloudFront is. The S3 website endpoint doesn't support the
> SigV4 signing that OAC requires, so CloudFront connects to the bucket's
> regular REST endpoint instead. The direct `http://...s3-website...`
> URL will now return Access Denied, which is expected.

## Prerequisites

- An AWS account
- [AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html) installed and configured (`aws configure`) with credentials that have permission to create S3 buckets, bucket policies, CloudFront distributions, and CloudFormation stacks
- The two files from this project in the same folder:
  - `shopme-s3-website.yaml`
  - `index.html`

## Step 1 — Choose a bucket name

S3 bucket names must be **globally unique** across all AWS accounts, lowercase,
and contain only letters, numbers, dots, and hyphens. This template defaults
to `shopme-website-123456789012`; override it with `--parameter-overrides`
if you need a different name.

## Step 2 — Deploy (or update) the CloudFormation stack

From a terminal, in the folder containing `shopme-s3-website_cdn.yaml`, run:

```bash
aws cloudformation deploy \
  --template-file shopme-s3-website_cdn.yaml \
  --stack-name shopme-website \
  --parameter-overrides BucketName=shopme-website-123456789012 \
  --capabilities CAPABILITY_IAM
```

If you already have the `shopme-website` stack from before (S3-only version),
this same command **updates it in place** — CloudFront, the OAC, and the new
bucket policy will be added without needing to delete the old stack.

Deployment (or update) typically takes a few minutes; CloudFront distribution
creation itself can take 5–15 minutes to fully propagate globally.

## Step 3 — Upload the website file to the bucket

```bash
aws s3 cp index.html s3://shopme-website-123456789012/index.html --content-type text/html
```

Re-run this command any time you update `index.html` to push a new version.

## Step 4 — Get the CloudFront URL

Either run:

```bash
aws cloudformation describe-stacks \
  --stack-name shopme-website \
  --query "Stacks[0].Outputs"
```

or open the **CloudFormation console → shopme-website stack → Outputs tab**.

Look for the `CloudFrontURL` output — that's your live site address, e.g.:

```
https://d1234abcd5678.cloudfront.net
```

The `S3WebsiteURL` output is still shown for reference but will no longer
work directly, since the bucket now blocks all public access.

## Step 5 — Open the site

Visit the `CloudFrontURL` from the previous step in your browser. The Shopme
site should load over HTTPS with search and cart functionality working
entirely client-side.

## Updating the site later

1. Re-run the Step 3 upload command to push the new `index.html` to S3.
2. **Invalidate the CloudFront cache** so visitors see the update immediately,
   instead of waiting for the cached version to expire:

   ```bash
   aws cloudformation describe-stacks \
     --stack-name shopme-website \
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
settings, etc.).

## Tearing down

CloudFront distributions must be **disabled** before they can be deleted,
and CloudFormation handles this automatically on stack deletion — but it
can take several minutes.

```bash
aws s3 rm s3://shopme-website-123456789012 --recursive
aws cloudformation delete-stack --stack-name shopme-website
aws cloudformation wait stack-delete-complete --stack-name shopme-website
```

The bucket must be emptied first — CloudFormation won't delete a non-empty
S3 bucket.

## Troubleshooting

| Issue | Likely cause |
|---|---|
| `BucketAlreadyExists` error on deploy | The bucket name is already taken globally — choose a different name |
| 403 Forbidden when visiting the **S3 website URL** | Expected — public access is now blocked. Use the `CloudFrontURL` output instead |
| 403 Forbidden when visiting the **CloudFront URL** | The bucket policy's `AWS:SourceArn` condition may not match the distribution yet — confirm the stack finished updating, or that `index.html` was actually uploaded |
| Changes not showing up after re-upload | CloudFront is serving a cached copy — create an invalidation (Step "Updating the site later") or wait for cache TTL to expire |
| Stack update stuck / slow | CloudFront distribution changes can take 5–15 minutes to propagate — this is normal AWS behavior, not a failure |
| `AccessDenied` calling `create-invalidation` | Your IAM credentials need `cloudfront:CreateInvalidation` permission |
