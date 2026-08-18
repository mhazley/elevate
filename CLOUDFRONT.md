# CloudFront + HTTPS + custom domain (elevatecoach.ing)

Puts CloudFront in front of a **private** S3 bucket (locked down with Origin
Access Control), serving `elevatecoach.ing` over HTTPS with an ACM certificate.
DNS stays at Gandi.

Run everything against your personal account:
```bash
export AWS_PROFILE=personal
aws sts get-caller-identity   # confirm Account = 826902266210
```

Placeholders: `BUCKET` (your existing site bucket), `ACCOUNT_ID` = `826902266210`,
`DIST_ID` (CloudFront distribution id, created in step 3).

---

## 1. Request the TLS certificate (must be us-east-1)

CloudFront only reads certs from **us-east-1**, regardless of your bucket region.

```bash
aws acm request-certificate \
  --region us-east-1 \
  --domain-name elevatecoach.ing \
  --subject-alternative-names www.elevatecoach.ing \
  --validation-method DNS \
  --query CertificateArn --output text
```

Note the printed **certificate ARN**. Then fetch the DNS validation records:

```bash
aws acm describe-certificate --region us-east-1 \
  --certificate-arn <CERT_ARN> \
  --query 'Certificate.DomainValidationOptions[].ResourceRecord'
```

You'll get one or two CNAME records like
`_abc123.elevatecoach.ing → _xyz.acm-validations.aws`.

**At Gandi** (Domain → DNS records), add each as a **CNAME**. Use just the host
label (strip the trailing `.elevatecoach.ing`), e.g. name `_abc123`, value
`_xyz.acm-validations.aws.` (keep the trailing dot). Validation flips to
`ISSUED` within minutes to an hour:

```bash
aws acm describe-certificate --region us-east-1 \
  --certificate-arn <CERT_ARN> --query 'Certificate.Status'
```
Wait for `"ISSUED"` before creating the distribution.

---

## 2. Origin Access Control (lets CloudFront read the private bucket)

```bash
aws cloudfront create-origin-access-control \
  --origin-access-control-config \
  Name=elevate-oac,SigningProtocol=sigv4,SigningBehavior=always,OriginAccessControlOriginType=s3 \
  --query 'OriginAccessControl.Id' --output text
```
Note the **OAC id**.

---

## 3. Create the distribution (console)

The 2024+ console splits this into "create minimal, then configure in tabs".

**Create:** CloudFront → **Create distribution** → **Origin domain** = your
bucket's **REST** endpoint (`BUCKET.s3.eu-west-1.amazonaws.com`), *not* the
`.s3-website` one. If an **Origin access** option appears, choose **Origin
access control settings** and create an OAC named `elevate-oac`. Click
**Create distribution**. Note the **Distribution domain** (`dXXXX.cloudfront.net`)
and **Distribution ID** (`DIST_ID`).

**Then open the distribution and edit these tabs:**

- **Origins** → edit the origin → Origin access = **Origin access control
  settings** → `elevate-oac`. It shows a *"update the S3 bucket policy"* banner
  with **Copy policy** — copy that; it's the step-4 policy, already scoped to this
  distribution (use it instead of hand-writing the JSON if you like).
- **Settings** (General) → Edit:
  - **Alternate domain names (CNAME)**: `elevatecoach.ing` and `www.elevatecoach.ing`.
  - **Custom SSL certificate**: the ACM cert from step 1 (only listed once
    **ISSUED** and in **us-east-1**).
  - **Default root object**: `index.html`.
- **Behaviors** → edit the `Default (*)` behavior:
  - **Viewer protocol policy**: **Redirect HTTP to HTTPS**.
  - **Cache policy**: **CachingOptimized** (managed).

Each save triggers a short "Deploying" cycle — normal.

> Optional niceties you can set later: a custom error response mapping 403/404 →
> `/index.html` (or a real `404.html`).

---

## 4. Lock the bucket to CloudFront only

Replace the old public-read policy with one that allows **only this
distribution** (via OAC). Remove the public policy first if present:

```bash
aws s3api put-bucket-policy --bucket BUCKET --policy '{
  "Version": "2012-10-17",
  "Statement": [{
    "Sid": "AllowCloudFrontOAC",
    "Effect": "Allow",
    "Principal": { "Service": "cloudfront.amazonaws.com" },
    "Action": "s3:GetObject",
    "Resource": "arn:aws:s3:::BUCKET/*",
    "Condition": {
      "StringEquals": {
        "AWS:SourceArn": "arn:aws:cloudfront::ACCOUNT_ID:distribution/DIST_ID"
      }
    }
  }]
}'
```

**Test before touching DNS**: open `https://dXXXX.cloudfront.net` — the site
should load over HTTPS. If it 403s, re-check the bucket policy `DIST_ID` and that
the origin uses the REST endpoint + OAC.

---

## 5. Point Gandi DNS at CloudFront

Gandi → your domain → **DNS records**:

| Type    | Name  | Value                    |
|---------|-------|--------------------------|
| `ALIAS` | `@`   | `dXXXX.cloudfront.net.`  |
| `CNAME` | `www` | `dXXXX.cloudfront.net.`  |

Gandi LiveDNS supports **`ALIAS`** at the apex (`@`) — that's how the bare
`elevatecoach.ing` can point at CloudFront (a plain CNAME isn't allowed at the
apex). Keep the trailing dot on the values.

> If Gandi won't accept an `ALIAS` record, the fallback is: make `www` the CNAME
> above, and set a **web redirect** from the apex to `https://www.elevatecoach.ing`
> — then also flip the `canonical` tags in the HTML to the `www` host.

Propagation is usually minutes. Verify:
```bash
curl -sI https://elevatecoach.ing | head -n 1        # expect HTTP/2 200
curl -sI https://www.elevatecoach.ing | head -n 1
```

---

## 6. Fully close public access (after DNS works)

Once the site loads via the domain over HTTPS, re-enable Block Public Access so
nothing is served straight from S3:

```bash
aws s3api put-public-access-block --bucket BUCKET \
  --public-access-block-configuration \
  BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true
```
(The OAC policy still works — it's a bucket policy for a service principal, not
"public".) You can also turn off S3 static website hosting now; CloudFront's
default root object handles `/`.

---

## 7. Wire cache invalidation into deploys

The workflow already has an invalidation step that activates once you set the
distribution id. Add these so pushes go live instantly:

1. **GitHub repo variable**: `CLOUDFRONT_DISTRIBUTION_ID` = `DIST_ID`.
2. **Extend the deploy role's permissions** (`perms.json`) with invalidation
   rights, then re-put it:

   ```json
   { "Effect": "Allow",
     "Action": ["cloudfront:CreateInvalidation"],
     "Resource": "arn:aws:cloudfront::ACCOUNT_ID:distribution/DIST_ID" }
   ```
   ```bash
   aws iam put-role-policy --role-name elevate-deploy \
     --policy-name elevate-s3-sync --policy-document file://perms.json
   ```

Push to `main` → sync to S3 → CloudFront cache invalidated → live.

---

## Done

- `https://elevatecoach.ing` and `https://www.elevatecoach.ing` serve over HTTPS.
- Bucket is private; only CloudFront can read it.
- HTTP auto-redirects to HTTPS — WhatsApp links work now.
- `canonical` tags already point at the apex, so `www` / the CloudFront domain
  won't cause duplicate-content issues.
