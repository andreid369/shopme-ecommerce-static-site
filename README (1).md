# Shopme Static Website Infrastructure

A secure, resilient, and production-ready static website architecture provisioned on AWS using AWS CloudFormation, Amazon S3, Amazon CloudFront (with Origin Access Control), AWS WAF, Amazon CloudWatch, and optional Cross-Region Disaster Recovery (DR) Replication, fully automated via GitHub Actions CI/CD.

---

## Architecture Overview

- **Storage**:
  - **Website Bucket**: Private, versioned Amazon S3 bucket hosting static site assets (`index.html`, etc.) with all public access strictly blocked.
  - **Images Bucket**: Dedicated private, versioned Amazon S3 bucket for product images (`/images/*`), featuring S3 Lifecycle transitions (Standard-IA at 60 days, Glacier at 180 days).
- **Content Delivery & Security**:
  - **Amazon CloudFront Distribution**: Distributes content globally over HTTPS with multi-origin routing.
  - **Origin Access Control (OAC)**: Restricts bucket access exclusively to CloudFront via SigV4 signing over REST endpoints. Direct S3 website access is disabled.
  - **AWS WAF (Web ACL)**: Enforces application-layer protection (e.g., AWS Managed SQL Injection Ruleset). Provisioned in `us-east-1` to meet CloudFront scope requirements.
- **Disaster Recovery (Optional Cross-Region Replication)**:
  - Asynchronous cross-region replication mirroring both website and images buckets to a secondary AWS region using a dedicated, least-privilege IAM replication role.
- **Observability**:
  - **CloudWatch Dashboard**: Displays side-by-side metrics for CloudFront `Requests` and S3 `AllRequests` (enabled via bucket request metrics configuration).
- **Automation**:
  - **GitHub Actions**: Continuous integration and deployment workflow for code syncing and automated CloudFront cache invalidation.

---

## Multi-Stack CloudFormation Architecture

The infrastructure is decoupled into three CloudFormation templates to adhere to AWS regional constraints:

1. **`shopme-waf.yaml` (`us-east-1`)**:
   - Manages the CloudFront-scoped WAF Web ACL (`Scope: CLOUDFRONT`), which AWS requires to be provisioned exclusively via the `us-east-1` (N. Virginia) endpoint.
2. **`shopme-dr.yaml` (Secondary Region, e.g., `<secondary-region>`)**:
   - *Optional*: Deployed only when cross-region disaster recovery is needed. Provisions versioned destination buckets with `BucketOwnerEnforced` settings.
3. **`shopme-s3-website.yaml` (Primary Region, e.g., `<primary-region>`)**:
   - The core stack provisioning source S3 buckets (versioned), CloudFront distribution, OAC, bucket policies, optional replication roles, and the CloudWatch dashboard.

---

## Repository Structure

```text
.
├── site/                       # Static web application assets
│   ├── index.html              # Frontend application entry point
│   └── images/                 # Product media assets (synced to images bucket)
│       ├── product-1.jpg
│       ├── product-2.jpg
│       └── ...
├── shopme-waf.yaml             # CloudFormation: AWS WAF Web ACL (us-east-1)
├── shopme-dr.yaml              # CloudFormation: Disaster Recovery destination buckets
├── shopme-s3-website.yaml      # CloudFormation: Primary website, OAC, distribution, and monitoring
├── .github/
│   └── workflows/
│       └── deploy.yml          # GitHub Actions deployment workflow
└── README.md
```

---

## Prerequisites

- **AWS Account** with administrative permissions to deploy S3 buckets, CloudFront distributions, IAM roles, WAF Web ACLs, and CloudFormation stacks.
- **AWS CLI** (version 2.x recommended) installed and configured (`aws configure`).
- Locally cloned repository.

---

## Deployment Guide

### 1. Define Unique Bucket Identifiers

Select globally unique bucket names conforming to AWS naming rules (lowercase alphanumeric and hyphens):
- Primary Website Bucket: `<primary-website-bucket>`
- Primary Images Bucket: `<primary-images-bucket>`
- DR Website Bucket (if replication enabled): `<dr-website-bucket>`
- DR Images Bucket (if replication enabled): `<dr-images-bucket>`

---

### 2. Deploy AWS WAF Stack (`us-east-1`)
*(Skip if WAF is not required)*

Deploy the WAF Web ACL stack to `us-east-1`:

```bash
aws cloudformation deploy \
  --template-file shopme-waf.yaml \
  --stack-name shopme-waf \
  --region us-east-1
```

Extract the Web ACL ARN:

```bash
WAF_ACL_ARN=$(aws cloudformation describe-stacks \
  --stack-name shopme-waf \
  --region us-east-1 \
  --query "Stacks[0].Outputs[?OutputKey=='WebACLArn'].OutputValue" \
  --output text)
```

---

### 3. Deploy Disaster Recovery Stack (Secondary Region)
*(Skip if Cross-Region Replication is not required)*

Deploy the destination buckets to your chosen secondary region (e.g., `eu-west-1`):

```bash
aws cloudformation deploy \
  --template-file shopme-dr.yaml \
  --stack-name shopme-dr \
  --parameter-overrides \
      DRWebsiteBucketName=<dr-website-bucket> \
      DRImagesBucketName=<dr-images-bucket> \
  --region <secondary-region>
```

Retrieve both destination bucket ARNs:

```bash
DR_WEBSITE_ARN=$(aws cloudformation describe-stacks \
  --stack-name shopme-dr \
  --region <secondary-region> \
  --query "Stacks[0].Outputs[?OutputKey=='DRWebsiteBucketArn'].OutputValue" \
  --output text)

DR_IMAGES_ARN=$(aws cloudformation describe-stacks \
  --stack-name shopme-dr \
  --region <secondary-region> \
  --query "Stacks[0].Outputs[?OutputKey=='DRImagesBucketArn'].OutputValue" \
  --output text)
```

---

### 4. Deploy Primary Website Stack (Primary Region)

Deploy to your primary deployment region (e.g., `eu-north-1`):

#### Configuration A: Full Deployment (WAF + DR Replication Enabled)

```bash
aws cloudformation deploy \
  --template-file shopme-s3-website.yaml \
  --stack-name shopme-website \
  --parameter-overrides \
      BucketName=<primary-website-bucket> \
      ImagesBucketName=<primary-images-bucket> \
      WebACLArn="$WAF_ACL_ARN" \
      EnableReplication=true \
      DestinationWebsiteBucketArn="$DR_WEBSITE_ARN" \
      DestinationImagesBucketArn="$DR_IMAGES_ARN" \
  --capabilities CAPABILITY_IAM CAPABILITY_NAMED_IAM \
  --region <primary-region>
```

#### Configuration B: Standard Deployment (WAF Enabled, No Replication)

```bash
aws cloudformation deploy \
  --template-file shopme-s3-website.yaml \
  --stack-name shopme-website \
  --parameter-overrides \
      BucketName=<primary-website-bucket> \
      ImagesBucketName=<primary-images-bucket> \
      WebACLArn="$WAF_ACL_ARN" \
      EnableReplication=false \
  --capabilities CAPABILITY_IAM CAPABILITY_NAMED_IAM \
  --region <primary-region>
```

#### Configuration C: Minimal Deployment (No WAF, No Replication)

```bash
aws cloudformation deploy \
  --template-file shopme-s3-website.yaml \
  --stack-name shopme-website \
  --parameter-overrides \
      BucketName=<primary-website-bucket> \
      ImagesBucketName=<primary-images-bucket> \
      EnableWAF=false \
      EnableReplication=false \
  --capabilities CAPABILITY_IAM CAPABILITY_NAMED_IAM \
  --region <primary-region>
```

> **IAM Capabilities Note**: `CAPABILITY_NAMED_IAM` is required because CloudFormation configures a custom-named IAM replication role (`ReplicationRole`) when replication is enabled.

---

## Asset Upload & Cache Invalidation

### Manual Synchronization

```bash
# Upload web entrypoint
aws s3 cp site/index.html s3://<primary-website-bucket>/index.html --content-type text/html

# Upload images under the images/ prefix
aws s3 sync site/images/ s3://<primary-images-bucket>/images/ --delete
```

Invalidate the CloudFront cache to serve the latest content:

```bash
DISTRIBUTION_ID=$(aws cloudformation describe-stacks \
  --stack-name shopme-website \
  --region <primary-region> \
  --query "Stacks[0].Outputs[?OutputKey=='CloudFrontDistributionId'].OutputValue" \
  --output text)

aws cloudfront create-invalidation \
  --distribution-id "$DISTRIBUTION_ID" \
  --paths "/*"
```

---

## Continuous Deployment with GitHub Actions

The workflow `.github/workflows/deploy.yml` triggers on every push to `main`:
1. Synchronizes non-image files in `./site` to the primary website bucket (excluding `images/*`).
2. Synchronizes `./site/images` to the images bucket under the `images/` prefix.
3. Automatically triggers a CloudFront invalidation for `/*`.

### Required GitHub Secrets

Configure these parameters under **Repository Settings > Secrets and variables > Actions**:

| Secret Name | Description |
|---|---|
| `AWS_ACCESS_KEY_ID` | CI/CD IAM user access key |
| `AWS_SECRET_ACCESS_KEY` | CI/CD IAM user secret access key |
| `AWS_REGION` | Primary stack target region (e.g., `<primary-region>`) |
| `S3_BUCKET_NAME` | Primary website S3 bucket name |
| `IMAGES_BUCKET_NAME` | Primary images S3 bucket name |
| `CLOUDFRONT_DISTRIBUTION_ID` | CloudFront Distribution ID |

*Production Recommendation: Use OpenID Connect (OIDC) with temporary IAM role credentials instead of long-lived access keys.*

---

## Lifecycle, Replication & Regional Notes

- **S3 Lifecycle Rules (Images Bucket)**:
  - **60 Days**: Automatic transition to `S3 Standard-IA`.
  - **180 Days**: Automatic transition to `S3 Glacier Flexible Retrieval`.
  - *Operational Notice*: Glacier objects require an explicit asynchronous restoration request before they can be retrieved. Serving cold Glacier items directly via CloudFront without restoring them will result in an error.
- **Cross-Region Replication Behavior**:
  - Replicates newly created or updated objects to the DR bucket retaining the storage class at time of upload.
  - S3 does not replicate subsequent lifecycle transitions applied to source objects to destination replicas.
  - Replication applies to objects uploaded after replication is enabled; existing objects require manual backfill or S3 Batch Replication.
- **S3 Transfer Acceleration**:
  - Transfer Acceleration is unsupported in specific AWS opt-in regions (such as `eu-north-1`). Ensure `AccelerateConfiguration` remains disabled on these buckets to avoid HTTP `405 Method Not Allowed` errors.

---

## Infrastructure Verification

Retrieve stack outputs and endpoints:

```bash
aws cloudformation describe-stacks \
  --stack-name shopme-website \
  --region <primary-region> \
  --query "Stacks[0].Outputs"
```

Key outputs:
- **`CloudFrontURL`**: Live public HTTPS endpoint (`https://<distribution-id>.cloudfront.net`).
- **`DashboardURL`**: CloudWatch metrics dashboard URL.

---

## Teardown & Deletion

Because versioning is enabled on the S3 buckets, all versions and delete markers must be deleted prior to stack teardown:

```bash
# Helper function to empty versioned buckets
empty_versioned_bucket() {
  local BUCKET=$1
  echo "Purging versioned bucket: $BUCKET"
  aws s3api delete-objects --bucket "$BUCKET" --delete "$(aws s3api list-object-versions --bucket "$BUCKET" --output json --query '{Objects: Versions[].{Key:Key,VersionId:VersionId}}')" 2>/dev/null || true
  aws s3api delete-objects --bucket "$BUCKET" --delete "$(aws s3api list-object-versions --bucket "$BUCKET" --output json --query '{Objects: DeleteMarkers[].{Key:Key,VersionId:VersionId}}')" 2>/dev/null || true
}

# Empty primary buckets
empty_versioned_bucket "<primary-website-bucket>"
empty_versioned_bucket "<primary-images-bucket>"

# Delete primary website stack
aws cloudformation delete-stack --stack-name shopme-website --region <primary-region>
aws cloudformation wait stack-delete-complete --stack-name shopme-website --region <primary-region>

# Delete WAF stack (if provisioned)
aws cloudformation delete-stack --stack-name shopme-waf --region us-east-1
aws cloudformation wait stack-delete-complete --stack-name shopme-waf --region us-east-1

# Empty and delete DR stack (if provisioned)
empty_versioned_bucket "<dr-website-bucket>"
empty_versioned_bucket "<dr-images-bucket>"
aws cloudformation delete-stack --stack-name shopme-dr --region <secondary-region>
aws cloudformation wait stack-delete-complete --stack-name shopme-dr --region <secondary-region>
```
