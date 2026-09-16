# Custom Domain Setup — shop.mydomain.com → CloudFront

This guide covers pointing `shop.mydomain.com` at your CloudFront
distribution: requesting an ACM certificate, attaching it to CloudFront,
and adding the DNS record.

Replace `mydomain.com` / `shop.mydomain.com` below with your actual domain.

## Overview of what's needed

1. An ACM (AWS Certificate Manager) certificate for `shop.mydomain.com`,
   issued in the **us-east-1** region (required by CloudFront, regardless
   of which region your stack/bucket lives in)
2. DNS validation of that certificate (adding a CNAME record your CA checks for)
3. Adding `shop.mydomain.com` as an alternate domain name (CNAME/alias) on
   the CloudFront distribution, with the certificate attached
4. A DNS record at your domain's DNS provider pointing `shop.mydomain.com`
   to the CloudFront distribution's domain name

## Step 1 — Request the ACM certificate (in us-east-1)

Console:
1. Go to **AWS Certificate Manager**
2. **Switch your region to US East (N. Virginia) / us-east-1** in the top-right region selector — this is required even if your bucket/CloudFront stack is elsewhere
3. Click **Request a certificate → Request a public certificate**
4. Enter the domain name: `shop.mydomain.com`
5. Validation method: **DNS validation** (recommended)
6. Click **Request**

CLI equivalent:
```bash
aws acm request-certificate \
  --domain-name shop.mydomain.com \
  --validation-method DNS \
  --region us-east-1
```

## Step 2 — Add the DNS validation record

1. Open the certificate in the ACM console (still in us-east-1) — status will show **Pending validation**
2. Expand the domain and copy the **CNAME name** and **CNAME value** ACM gives you
3. Add that as a CNAME record at your DNS provider:

```
Name:  _xxxxxxxxxxxxxxxx.shop.mydomain.com
Type:  CNAME
Value: _yyyyyyyyyyyyyyyy.xyzcompany.acm-validations.aws.
```

(Exact values are unique to your request — always copy them from the ACM console, don't reuse these placeholders.)

4. Wait for validation — usually a few minutes, occasionally up to ~30 minutes. The certificate status changes to **Issued** once done. You can check with:

```bash
aws acm describe-certificate \
  --certificate-arn arn:aws:acm:us-east-1:682563173107:certificate/944000e2-6c1a-4431-a3dd-4c6138eaf589 \
  --region us-east-1 \
  --query "Certificate.Status"
```

> If your domain is already on Route 53, ACM can add this validation
> record for you automatically — the console offers a **"Create records
> in Route 53"** button on the pending certificate.

nslookup -type=CNAME _d2accc4e238197e2d55186f3e9c908b0.shop.opencloud.ro 

## Step 3 — Attach the domain + certificate to CloudFront

Console:
1. Go to **CloudFront → your distribution → Edit** (General settings)
2. Under **Alternate domain name (CNAME)**, add `shop.mydomain.com`
3. Under **Custom SSL certificate**, select the certificate you just issued (it will only appear here once status is "Issued" and it's in us-east-1)
4. Save changes — this triggers a distribution update, which can take 5–15 minutes to deploy

CLI equivalent (update the distribution config):
```bash
# 1. Get the current config + ETag
aws cloudfront get-distribution-config \
  --id <your-distribution-id> > dist-config.json

# 2. Edit dist-config.json:
#    - Add "shop.mydomain.com" to DistributionConfig.Aliases.Items
#      (and set Aliases.Quantity accordingly)
#    - Set DistributionConfig.ViewerCertificate.ACMCertificateArn to your cert ARN
#    - Set ViewerCertificate.SSLSupportMethod to "sni-only"
#    - Remove/replace any existing default CloudFront certificate settings

# 3. Apply the update (requires the ETag from step 1)
aws cloudfront update-distribution \
  --id <your-distribution-id> \
  --distribution-config file://updated-config.json \
  --if-match <ETag-from-step-1>
```

> If you're managing this distribution via the CloudFormation template
> instead of manual CLI edits, add `Aliases: [shop.mydomain.com]` and a
> `ViewerCertificate` block (`AcmCertificateArn`, `SslSupportMethod:
> sni-only`, `MinimumProtocolVersion`) to the `DistributionConfig` in
> the template, then redeploy the stack.

## Step 4 — Point your domain's DNS at CloudFront

At your DNS provider (or Route 53), add a record for `shop.mydomain.com`:

**If hosted on Route 53:**
- Record type: **A — Alias**
- Alias target: your CloudFront distribution (select it from the dropdown)

**If hosted elsewhere (GoDaddy, Namecheap, Cloudflare, etc.):**
```
Name:  shop
Type:  CNAME
Value: d1234abcd5678.cloudfront.net
```
(Use your actual `CloudFrontDomainName` output value, not this placeholder.)

## Step 5 — Verify

Wait for DNS propagation (usually minutes, up to ~48 hours worst case), then:

```bash
nslookup shop.mydomain.com
```

should resolve to CloudFront's IPs, and:

```
https://shop.mydomain.com
```

should load your site with a valid HTTPS certificate (check the padlock
in your browser — it should show it's issued for `shop.mydomain.com`).

## Troubleshooting

| Issue | Likely cause |
|---|---|
| Certificate stuck on "Pending validation" | DNS validation CNAME record wasn't added correctly, or hasn't propagated yet — double-check the exact name/value from ACM |
| Certificate not selectable in CloudFront | Certificate isn't in **us-east-1**, or isn't yet in "Issued" status |
| `CNAMEAlreadyExists` error on CloudFront update | Another distribution already has this alias attached — check you're not double-assigning it |
| Browser shows certificate warning | Wrong/self-signed cert selected, or CloudFront hasn't finished deploying the update yet |
| `shop.mydomain.com` doesn't resolve | DNS record missing, still propagating, or added at the wrong DNS provider (make sure you're editing the zone that's actually authoritative for the domain) |
| Site loads but shows Access Denied | Distribution update deployed but S3 bucket policy / OAC scoping might need rechecking — unrelated to the domain/cert setup itself |





=======================================================

Parameter	Value to use	Purpose
BucketName	shopme-website-123456789012	Your S3 bucket name
EnableWAF	true	Turns WAF on
WebACLArn	arn:aws:wafv2:us-east-1:682563173107:global/webacl/shopme-website-waf/f8dee533-b41d-4a7a-84c6-f8504c30ed38
DomainName	shop.opencloud.ro	Your custom domain
AcmCertificateArn	arn:aws:acm:us-east-1:682563173107:certificate/944000e2-6c1a-4431-a3dd-4c6138eaf589	Your HTTPS certificate