# Deployment Guide — Shopme Static Website (S3 + CloudFront + OAC + WAF + Dashboard + CI/CD + DR Replication)

This guide walks through deploying three CloudFormation stacks:

1. **`shopme-waf.yaml`** — a standalone WAF Web ACL, which must be deployed
   in **us-east-1**
2. **`shopme-dr.yaml`** — standalone disaster-recovery destination buckets,
   deployed in a **secondary region** (e.g. `eu-west-1`) — only needed if
   you want Cross-Region Replication
3. **`shopme-s3-website.yaml`** — the main stack (S3 buckets + CloudFront +
   OAC), deployed in **your normal region** (e.g. `eu-north-1`), which
   optionally attaches the WAF Web ACL from step 1 and optionally
   replicates to the DR buckets from step 2

## Why multiple stacks?

WAF Web ACLs with `Scope: CLOUDFRONT` can only be **created** via the
us-east-1 API endpoint — this is a hard AWS requirement, unrelated to where
your other resources live. If your main stack normally deploys to a
different region (this project uses `eu-north-1`), CloudFormation can't
create a `Scope: CLOUDFRONT` Web ACL inside that same stack. Splitting it
into its own us-east-1 stack, and passing its ARN into the main stack as a
plain parameter, avoids the region conflict entirely.

The same logic applies to disaster recovery: an S3 bucket always lives in
whichever region its own CloudFormation stack deploys to — there's no way
to create a bucket "in a different region" from within a single stack.
So the DR destination buckets get their own small stack, deployed to
whatever secondary region you choose, and the main stack references their
ARNs as plain parameters rather than creating them directly.

## What each template creates

**`shopme-waf.yaml`** (deploy to us-east-1):
- An AWS WAF Web ACL with the AWS Managed Rule Group `AWSManagedRulesSQLiRuleSet`
  (SQL injection protection)

**`shopme-dr.yaml`** (deploy to a secondary region, e.g. eu-west-1) — only
needed if using replication:
- `DRWebsiteBucket`, `DRImagesBucket` — private, versioned destination
  buckets mirroring the two source buckets, using `BucketOwnerEnforced`
  ownership (simplest model for replication, avoids ACL complications)

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
- A CloudWatch dashboard displaying the CloudFront `Requests` metric and
  the S3 `AllRequests` metric side by side
- An S3 bucket-level metrics configuration (`EntireBucket`), which is
  required to actually enable the `AllRequests` metric — S3 doesn't
  publish it by default
- A **second S3 bucket** (default name `shopme-images-123456789012`),
  fully private, dedicated to product images
- A **second CloudFront origin** pointed at that bucket, with its own OAC,
  routed via a `/images/*` path pattern — so `https://<your-domain>/images/foo.jpg`
  serves `foo.jpg` from the images bucket, not the website bucket
- **S3 Transfer Acceleration** was evaluated but is **not enabled** —
  unsupported in this stack's region (`eu-north-1`); see the dedicated
  section below
- **S3 Lifecycle rules** on the images bucket, transitioning objects to
  cheaper storage tiers as they age
- **Versioning** enabled on both source buckets (required for replication)
- **(Optional, off by default)** S3 Cross-Region Replication from both
  source buckets to the DR buckets from `shopme-dr.yaml`, via a dedicated
  IAM role — see the dedicated section below

> **Note:** the S3 website-hosting feature (index/error document config)
> stays enabled on the bucket, but it's no longer the way visitors reach
> the site — CloudFront is. The S3 website endpoint doesn't support the
> SigV4 signing that OAC requires, so CloudFront connects to the bucket's
> regular REST endpoint instead. The direct `http://...s3-website...`
> URL will return Access Denied, which is expected.

> **Note:** enabling the S3 `AllRequests` metric via `MetricsConfigurations`
> incurs a small additional cost (S3 request metrics pricing — separate
> from normal storage/transfer charges). Check current pricing at
> [aws.amazon.com/s3/pricing](https://aws.amazon.com/s3/pricing) if this
> matters for your budget.

## S3 Transfer Acceleration — not available in this deployment

S3 Transfer Acceleration was initially added to both buckets, but AWS
does **not support this feature for buckets in `eu-north-1` (Stockholm)**
— it's one of several AWS "opt-in" regions excluded from Transfer
Acceleration. Attempting to enable it causes a `405 Method Not Allowed`
error during stack deployment.

As a result:
- `AccelerateConfiguration` is **not** set on either bucket in
  `shopme-s3-website.yaml`
- The GitHub Actions workflow's `aws s3 sync` commands do **not** use
  the `--endpoint-url https://s3-accelerate.amazonaws.com` flag
- Uploads (manual or via CI/CD) go through the bucket's normal regional
  endpoint

If this stack is ever redeployed to a region that supports Transfer
Acceleration, `AccelerateConfiguration: {AccelerationStatus: Enabled}`
can be added back to the bucket resource(s), and the corresponding
`--endpoint-url` flag added back to the workflow's sync steps. AWS
periodically updates which regions support this feature — check
[the current list](https://docs.aws.amazon.com/AmazonS3/latest/userguide/transfer-acceleration.html)
before re-enabling.

## S3 Lifecycle rules (images bucket)

The images bucket has a lifecycle rule that automatically moves objects
to cheaper storage tiers as they age, since older/less-frequently-updated
product images don't need to sit on full-price S3 Standard indefinitely:

| Object age | Transitions to |
|---|---|
| 60 days | S3 Standard-IA (Infrequent Access) |
| 180 days | S3 Glacier |

**Important — Glacier means retrieval delay.** Once an object transitions
to Glacier, it is **not instantly accessible**. Retrieving it requires an
explicit restore request that can take anywhere from minutes to several
hours depending on retrieval speed tier. If CloudFront tries to serve an
image straight from Glacier, it will fail — visitors will see a broken
image (the site's emoji fallback will kick in instead, so the page won't
visually break, but the real photo won't load).

**What this means in practice:**
- Fine for images you genuinely expect to go "cold" — discontinued
  products, seasonal items, anything unlikely to need live serving after
  ~6 months
- **Risky for any image still actively referenced on the live site** —
  if a product stays listed past 180 days without its image being
  re-uploaded (which resets the age), that image will silently stop
  loading
- Periodically check which images are approaching 180 days old, and
  either refresh (re-upload) images for products still active, or accept
  the broken-image fallback for genuinely retired products

Standard-IA (the 60-day step) does **not** have this retrieval delay —
objects there remain instantly accessible, just at a different storage
price point with a per-GB retrieval fee. Only the Glacier step (180 days)
introduces the delay.

## Disaster recovery — S3 Cross-Region Replication (optional)

Both source buckets can replicate every object to a secondary region for
disaster recovery, controlled by the `EnableReplication` parameter on
the main stack (default `"false"`).

**What gets created when enabled:**
- Versioning on both source buckets (`WebsiteBucket`, `ImagesBucket`) —
  this is **always on**, regardless of `EnableReplication`, since it's
  required infrastructure for replication and good practice generally
- A dedicated `ReplicationRole` IAM role, scoped to read from the two
  source buckets and write to the two destination buckets — nothing
  broader
- A `ReplicationConfiguration` on each source bucket, replicating **all**
  objects (no prefix filter) to its corresponding DR bucket

**Storage class handling:** the replication rules don't specify an
override `StorageClass` in the `Destination` block, so replicated objects
land in the DR bucket at whatever storage class they currently have on
the source at the moment of replication (satisfies "replicate objects
including their storage class").

**Important — this does NOT keep replicas in sync with later lifecycle
transitions.** If an object on the source images bucket later transitions
from S3 Standard to Standard-IA or Glacier (via the Lifecycle rule
above), that transition is **not** automatically mirrored to the DR
copy — S3 Cross-Region Replication replicates the object as it exists at
replication time, not ongoing storage-class changes made afterward by a
separate lifecycle rule. The DR buckets in `shopme-dr.yaml` intentionally
have **no lifecycle rules of their own** in this setup, so replicated
objects stay in DR at whatever class they arrived in — they will not
independently transition to IA/Glacier over time in the DR region. If you
want the DR bucket's storage costs and tiering to mirror the source
bucket's long-term behavior, matching lifecycle rules would need to be
added to `shopme-dr.yaml` separately — deliberately left out of this
setup.

**Deploy order:**
1. Deploy `shopme-dr.yaml` to your chosen secondary region first (e.g.
   `eu-west-1`) — see Step 3a below
2. Copy its `DRWebsiteBucketArn` and `DRImagesBucketArn` outputs
3. Deploy/update the main stack with `EnableReplication=true` and those
   two ARNs passed in as `DestinationWebsiteBucketArn` /
   `DestinationImagesBucketArn`

**Cost note:** Cross-Region Replication incurs storage costs in the DR
region (a full second copy of your data) plus inter-region data transfer
charges for every replicated object. For a small static site this is
usually modest, but it's an ongoing cost that scales with how much you
store — check current rates at
[aws.amazon.com/s3/pricing](https://aws.amazon.com/s3/pricing).

## Repository structure

```
.
├── site/                       # Everything here gets deployed to S3 — and only this
│   ├── index.html
│   └── images/                 # Product images — synced to the IMAGES bucket, not the website bucket
│       ├── product-1.jpg
│       ├── product-2.jpg
│       └── ...
├── shopme-waf.yaml
├── shopme-s3-website.yaml
├── deployment.md
├── custom-domain-setup.md
└── .github/
    └── workflows/
        └── deploy.yml           # GitHub Actions CI/CD — see "Continuous deployment" below
```

Keeping site content inside `site/` (rather than the repo root) means the
GitHub Actions workflow can sync that one folder to S3 without needing to
individually exclude every infra/docs file — anything outside `site/`
simply never gets deployed.

## Prerequisites

- An AWS account
- [AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html) installed and configured (`aws configure`) with credentials that have permission to create S3 buckets, bucket policies, CloudFront distributions, WAF Web ACLs, and CloudFormation stacks
- The following from this project, following the repository structure above:
  - `shopme-waf.yaml`
  - `shopme-s3-website.yaml`
  - `site/index.html`

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

## Step 3a — Deploy the DR stack (secondary region) — skip if not using replication

```bash
aws cloudformation deploy \
  --template-file shopme-dr.yaml \
  --stack-name shopme-dr \
  --region eu-west-1
```

(Use whichever secondary region you want for disaster recovery — `eu-west-1`
is just this project's example choice. Avoid picking the same region as
your main stack, and avoid opt-in regions if you might later want features
like Transfer Acceleration there too.)

Then retrieve both destination bucket ARNs:

```bash
aws cloudformation describe-stacks \
  --stack-name shopme-dr \
  --region eu-west-1 \
  --query "Stacks[0].Outputs[?OutputKey=='DRWebsiteBucketArn'].OutputValue" \
  --output text

aws cloudformation describe-stacks \
  --stack-name shopme-dr \
  --region eu-west-1 \
  --query "Stacks[0].Outputs[?OutputKey=='DRImagesBucketArn'].OutputValue" \
  --output text
```

Copy both ARNs — you'll pass them into the main stack next.

## Step 4 — Deploy (or update) the main stack

Replace `<your-region>` with wherever this stack normally lives (this
project uses `eu-north-1`), `<arn-from-step-3>` with the WAF ARN, and
`<website-dr-arn>` / `<images-dr-arn>` with the two ARNs from Step 3a.

**With WAF and replication both enabled:**
```bash
aws cloudformation deploy \
  --template-file shopme-s3-website.yaml \
  --stack-name shopme-website \
  --parameter-overrides \
      BucketName=shopme-website-123456789012 \
      ImagesBucketName=shopme-images-123456789012 \
      WebACLArn=<arn-from-step-3> \
      EnableReplication=true \
      DestinationWebsiteBucketArn=<website-dr-arn> \
      DestinationImagesBucketArn=<images-dr-arn> \
  --capabilities CAPABILITY_IAM CAPABILITY_NAMED_IAM \
  --region <your-region>
```

**With neither WAF nor replication:**
```bash
aws cloudformation deploy \
  --template-file shopme-s3-website.yaml \
  --stack-name shopme-website \
  --parameter-overrides BucketName=shopme-website-123456789012 EnableWAF=false \
  --capabilities CAPABILITY_IAM CAPABILITY_NAMED_IAM \
  --region <your-region>
```

**Why `CAPABILITY_NAMED_IAM`:** the `ReplicationRole` (created only when
`EnableReplication=true`) has an explicit `RoleName` rather than letting
CloudFormation auto-generate one. Any template that names an IAM resource
explicitly requires `CAPABILITY_NAMED_IAM` in addition to the usual
`CAPABILITY_IAM` — otherwise the deploy fails with
`InsufficientCapabilitiesException`. Both example commands above include
it so you can copy-paste either one safely regardless of whether
replication is enabled in that particular deploy.

Mix and match `EnableWAF` / `EnableReplication` independently depending on
which features you want active — each has its own toggle and its own
required parameters (see "What each template creates" above for exactly
which parameters pair with which toggle).

If you already have the `shopme-website` stack from a previous deployment,
this same command **updates it in place** — no need to delete and recreate.
**Remember:** every `deploy` call must include every parameter you want to
keep, not just the one you're changing — CloudFormation replaces the full
parameter set with whatever's in this specific command, falling back to
template defaults for anything omitted.

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

## Step 5 — Upload the website file to the bucket (manual method)

```bash
aws s3 cp site/index.html s3://shopme-website-123456789012/index.html --content-type text/html
```

Re-run this command any time you update `site/index.html` to push a new
version manually. **If you've set up the GitHub Actions workflow (see
"Continuous deployment" below), you don't need to run this manually** —
pushing to `main` does it automatically.

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

## Step 8 — View the CloudWatch dashboard

Get the direct link from the stack outputs:

```bash
aws cloudformation describe-stacks \
  --stack-name shopme-website \
  --region <your-region> \
  --query "Stacks[0].Outputs[?OutputKey=='DashboardURL'].OutputValue" \
  --output text
```

Or open **CloudWatch console → Dashboards → `<bucket-name>-dashboard`**
(e.g. `shopme-website-123456789012-dashboard`) in your stack's region.

The dashboard shows two widgets side by side:
- **CloudFront Requests** — total request count to the distribution,
  queried against us-east-1 (CloudFront metrics always publish there,
  regardless of your stack's own region)
- **S3 AllRequests** — total request count to the bucket, queried against
  your stack's own region

Both metrics can take a little while to show data after first deploying —
S3 request metrics in particular can take up to ~15 minutes for the first
data points to appear. Generate some traffic (visit the site a few times,
run a few `aws s3 cp` uploads) if the graphs look empty at first.

## Custom domain (optional)

If you're pointing a domain like `shop.yourdomain.com` at this distribution,
see the separate `custom-domain-setup.md` guide for the ACM certificate and
DNS record steps.

## Continuous deployment with GitHub Actions

Instead of manually running `aws s3 cp` every time, `.github/workflows/deploy.yml`
automatically syncs `site/` to the website bucket, syncs `site/images/` to
the images bucket, and invalidates the CloudFront cache — all on every
push to `main` (or manually via the Actions tab, using `workflow_dispatch`).

**What it does:**
1. Checks out the repo
2. Configures AWS credentials from GitHub Secrets
3. Runs `aws s3 sync ./site s3://<website-bucket> --delete --exclude "images/*"`
   — uploads site files (HTML, etc.), explicitly skipping the `images/`
   subfolder so images don't get duplicated into the website bucket
4. Runs `aws s3 sync ./site/images s3://<images-bucket>/images --delete`
   — uploads product images to the dedicated images bucket, under the
   `images/` key prefix (required so the `/images/*` CloudFront path
   pattern resolves to the right S3 keys)
5. Runs `aws cloudfront create-invalidation --paths "/*"` — so visitors
   see the new content immediately instead of a cached copy

### One-time setup

1. **Create a dedicated IAM user for CI/CD** (don't reuse your personal
   credentials). Attach a policy scoped to just what's needed:
   - `s3:PutObject`, `s3:DeleteObject`, `s3:ListBucket` on **both** buckets
     (website and images)
   - `cloudfront:CreateInvalidation` on your distribution

   Generate an access key for this user.

2. **Add these as GitHub repository secrets**
   (repo → **Settings → Secrets and variables → Actions → New repository secret**):

   | Secret name | Example value |
   |---|---|
   | `AWS_ACCESS_KEY_ID` | from the CI IAM user |
   | `AWS_SECRET_ACCESS_KEY` | from the CI IAM user |
   | `AWS_REGION` | `eu-north-1` |
   | `S3_BUCKET_NAME` | `shopme-website-123456789012` |
   | `IMAGES_BUCKET_NAME` | `shopme-images-123456789012` |
   | `CLOUDFRONT_DISTRIBUTION_ID` | `E2D646JE7JDDMQ` |

3. **Make sure the repo structure matches** the layout above — product
   images go in `site/images/`, named to match what `index.html`
   references (e.g. `product-1.jpg` through `product-24.jpg`). See
   "Sourcing product images" below for where to get these.

4. **Push to `main`** — either by committing a real change, or triggering
   it manually from the **Actions** tab (`workflow_dispatch` is already
   enabled in `deploy.yml`). Watch the run logs to confirm all three
   steps succeed.

### Sourcing product images

`index.html` references images at `/images/product-<id>.jpg` for products
1 through 24, but no actual image files exist yet — the site falls back
to an emoji icon per product until you add them.

Use a free-license stock photo site — **no copyright risk, no attribution
required**:
- [Unsplash](https://unsplash.com) — Unsplash License
- [Pexels](https://pexels.com) — Pexels License

Search for each product (search terms below), download a photo you like,
and save it into `site/images/` using the exact filename listed — this
must match what `index.html` expects.

| id | Product | Search term | Save as |
|----|---------|-------------|---------|
| 1 | Wireless Headphones | wireless headphones | `product-1.jpg` |
| 2 | Smart Fitness Watch | smart watch | `product-2.jpg` |
| 3 | 4K Streaming Media Player | streaming device tv | `product-3.jpg` |
| 4 | Portable Bluetooth Speaker | bluetooth speaker | `product-4.jpg` |
| 5 | French Press Coffee Maker | french press coffee | `product-5.jpg` |
| 6 | Non-Stick Cookware Set | cookware set pots pans | `product-6.jpg` |
| 7 | Memory Foam Pillow | pillow bedroom | `product-7.jpg` |
| 8 | Robot Vacuum Cleaner | robot vacuum | `product-8.jpg` |
| 9 | Novel (book) | book novel | `product-9.jpg` |
| 10 | Self-help Book | self help book | `product-10.jpg` |
| 11 | Cooking Guide Book | cookbook | `product-11.jpg` |
| 12 | Men's Cotton T-Shirt | mens t-shirt | `product-12.jpg` |
| 13 | Women's Running Shoes | running shoes | `product-13.jpg` |
| 14 | Polarized Sunglasses | sunglasses | `product-14.jpg` |
| 15 | Leather Jacket | leather jacket | `product-15.jpg` |
| 16 | Yoga Mat | yoga mat | `product-16.jpg` |
| 17 | Camping Tent | camping tent | `product-17.jpg` |
| 18 | Adjustable Dumbbell Set | dumbbells | `product-18.jpg` |
| 19 | Insulated Water Bottle | water bottle | `product-19.jpg` |
| 20 | Moisturizing Face Cream | face cream skincare | `product-20.jpg` |
| 21 | Professional Hair Dryer | hair dryer | `product-21.jpg` |
| 22 | Building Blocks Toy Set | building blocks toy | `product-22.jpg` |
| 23 | RC Racing Car | remote control car toy | `product-23.jpg` |
| 24 | Mechanical Gaming Keyboard | gaming keyboard | `product-24.jpg` |

Once added, commit and push — the workflow uploads them to the images
bucket automatically on the next run.

### Security note

This setup uses long-lived AWS access keys stored as GitHub Secrets — fine
for a bootcamp/learning project, but for production use, AWS recommends
**OIDC (OpenID Connect)** instead: GitHub Actions assumes a temporary IAM
role with no stored keys at all. That requires a bit more setup (an IAM
identity provider + trust policy) but removes the risk of a leaked or
stale long-lived key entirely.

## Updating the site later

**With the GitHub Actions workflow set up:** just commit your change to
`site/index.html` and push to `main` — the sync and cache invalidation
happen automatically.

**Manually (without CI/CD):**
1. Re-run the Step 5 upload command to push the new `site/index.html` to S3.
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

**Important — versioning changes bucket emptying.** Now that versioning
is enabled on both source buckets, `aws s3 rm --recursive` only removes
the *current* version of each object — old versions and delete markers
remain, and CloudFormation will refuse to delete a bucket that isn't
**completely** empty (all versions included). Empty each versioned
bucket properly first:

```bash
# Repeat for both shopme-website-123456789012 and shopme-images-123456789012
BUCKET=shopme-website-123456789012

aws s3api delete-objects --bucket $BUCKET --delete "$(aws s3api list-object-versions \
  --bucket $BUCKET --output json \
  --query '{Objects: Versions[].{Key:Key,VersionId:VersionId}}')"

aws s3api delete-objects --bucket $BUCKET --delete "$(aws s3api list-object-versions \
  --bucket $BUCKET --output json \
  --query '{Objects: DeleteMarkers[].{Key:Key,VersionId:VersionId}}')"
```

Then delete the stacks:

```bash
aws cloudformation delete-stack --stack-name shopme-website --region <your-region>
aws cloudformation wait stack-delete-complete --stack-name shopme-website --region <your-region>

# If you also want to remove the WAF Web ACL:
aws cloudformation delete-stack --stack-name shopme-waf --region us-east-1
aws cloudformation wait stack-delete-complete --stack-name shopme-waf --region us-east-1

# If you also deployed the DR stack, empty its two versioned buckets the
# same way as above first, then:
aws cloudformation delete-stack --stack-name shopme-dr --region eu-west-1
aws cloudformation wait stack-delete-complete --stack-name shopme-dr --region eu-west-1
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
| Unexpected AWS bill increase | Check whether the WAF stack is still deployed and attached — WAF has ongoing per-month and per-request charges even with no incidents. Also check whether S3 request metrics (`MetricsConfigurations`) are enabled — these have their own small per-request cost |
| CloudFront Requests widget is empty | Confirm the dashboard widget is querying `us-east-1` — CloudFront metrics always publish there. Also allow a few minutes after deploy/traffic for data to appear |
| S3 AllRequests widget is empty | Confirm `MetricsConfigurations` with `Id: EntireBucket` is present on the bucket (it enables the metric) and that some requests have actually hit the bucket recently — allow up to ~15 minutes for the first data points |
| Dashboard resource fails to create/update | Check `DashboardBody` is valid JSON — a stray comma or unescaped `${...}` reference is the most common cause. CloudFormation will report the exact JSON parse error in the stack events |
| GitHub Actions workflow fails at "Configure AWS credentials" | One or more of `AWS_ACCESS_KEY_ID` / `AWS_SECRET_ACCESS_KEY` / `AWS_REGION` secrets is missing or misspelled — check **Settings → Secrets and variables → Actions** |
| GitHub Actions workflow fails at "Sync website files to S3" with AccessDenied | The CI IAM user's policy doesn't include `s3:PutObject` / `s3:DeleteObject` / `s3:ListBucket` on the bucket, or `S3_BUCKET_NAME` secret doesn't match the real bucket name |
| GitHub Actions workflow fails at "Invalidate CloudFront cache" | The CI IAM user is missing `cloudfront:CreateInvalidation`, or `CLOUDFRONT_DISTRIBUTION_ID` secret is wrong |
| Workflow runs but site doesn't update | Confirm `site/index.html` (not a root-level `index.html`) was actually changed and committed — the workflow only syncs the `site/` folder |
| Workflow doesn't trigger at all | Confirm `deploy.yml` is at exactly `.github/workflows/deploy.yml` and you pushed to the `main` branch specifically |
| Product images show emoji instead of photos | No image file exists yet at that path in the images bucket — see "Sourcing product images" above. Confirm the filename in `site/images/` exactly matches what `index.html` references (case-sensitive) |
| Images upload but still don't load on the site | Confirm they were uploaded under the `images/` key prefix (e.g. `images/product-1.jpg`), not the bucket root — the `/images/*` CloudFront path pattern forwards the full path, so the S3 key must include that prefix too |
| GitHub Actions fails on the images sync step | Check the `IMAGES_BUCKET_NAME` secret is set and matches the real bucket name, and the CI IAM user's policy includes `s3:PutObject` / `s3:DeleteObject` / `s3:ListBucket` on that bucket too, not just the website bucket |
| CloudFormation deploy fails with `405` on `WebsiteBucket` and/or `ImagesBucket` | S3 Transfer Acceleration was likely re-enabled on a bucket in `eu-north-1`, which doesn't support it — remove `AccelerateConfiguration` from the affected bucket(s) |
| A product image that used to load now shows the emoji fallback | The image has likely transitioned to Glacier (180+ days old) and is no longer instantly retrievable — re-upload the image (which resets its age) if it should stay live, or leave it as expected behavior for a retired product |
| Main stack fails with an invalid/missing `DestinationWebsiteBucketArn` or `DestinationImagesBucketArn` | Make sure `EnableReplication=true` is paired with real ARN values from the `shopme-dr` stack's outputs — deploy `shopme-dr.yaml` first (Step 3a) if you haven't |
| `shopme-dr.yaml` fails with `BucketAlreadyExists` | The DR bucket name is already taken globally — choose a different `DRWebsiteBucketName`/`DRImagesBucketName` |
| Objects aren't appearing in the DR buckets | Replication only applies to objects uploaded **after** it was enabled — existing objects at the time you turn on `EnableReplication` are not retroactively replicated. Re-upload (or use S3 Batch Replication separately) to backfill older objects |
| DR bucket images don't show reduced storage costs over time the way the source bucket does | Expected — the DR buckets have no lifecycle rules of their own in this setup, so replicated objects stay at whatever storage class they arrived in and don't independently transition to IA/Glacier. See "Disaster recovery" above |
| `AccessDenied` during replication (visible in the source bucket's replication metrics) | Check the `ReplicationRole`'s policy actually covers both source bucket ARNs and both destination bucket ARNs — a mismatched or stale `DestinationWebsiteBucketArn`/`DestinationImagesBucketArn` parameter value is the most common cause |
| CloudFormation refuses to delete a bucket during teardown | Versioned buckets need every version and delete marker removed, not just current objects — see the updated "Tearing down" section above |
| `InsufficientCapabilitiesException: Requires capabilities: [CAPABILITY_NAMED_IAM]` | The `ReplicationRole` has an explicit name, which requires `CAPABILITY_NAMED_IAM` in addition to `CAPABILITY_IAM` — add both to your `--capabilities` flag |
