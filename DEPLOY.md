# Deploying Elevate Coaching to S3

Static site → S3 bucket with website hosting, deployed by GitHub Actions using
OIDC (no long-lived AWS keys). No CloudFront/DNS yet — we'll serve from the S3
website endpoint for feedback, and add HTTPS + a custom domain later.

Fill in these values as you go:

| Placeholder   | Example                          |
|---------------|----------------------------------|
| `BUCKET`      | `elevate-coaching-site`          |
| `REGION`      | `eu-west-2` (London)             |
| `ACCOUNT_ID`  | `123456789012`                   |
| `OWNER/REPO`  | `matt/elevate`                   |

---

## 1. Create the bucket + enable website hosting

```bash
aws s3api create-bucket \
  --bucket BUCKET \
  --region REGION \
  --create-bucket-configuration LocationConstraint=REGION

aws s3 website s3://BUCKET \
  --index-document index.html \
  --error-document index.html
```

## 2. Make it publicly readable

The website endpoint needs public read access (there's no CloudFront in front yet).

```bash
# Allow public bucket policies
aws s3api put-public-access-block \
  --bucket BUCKET \
  --public-access-block-configuration \
  BlockPublicAcls=false,IgnorePublicAcls=false,BlockPublicPolicy=false,RestrictPublicBuckets=false

# Public read of objects only
aws s3api put-bucket-policy --bucket BUCKET --policy '{
  "Version": "2012-10-17",
  "Statement": [{
    "Sid": "PublicReadGetObject",
    "Effect": "Allow",
    "Principal": "*",
    "Action": "s3:GetObject",
    "Resource": "arn:aws:s3:::BUCKET/*"
  }]
}'
```

## 3. Create the GitHub OIDC provider (once per AWS account)

Skip if you've already added it for another repo.

```bash
aws iam create-open-id-connect-provider \
  --url https://token.actions.githubusercontent.com \
  --client-id-list sts.amazonaws.com
```
> If your CLI complains that a thumbprint is required, add:
> `--thumbprint-list 6938fd4d98bab03faadb97b34396831e3780aea1`

## 4. Create the deploy role

**Trust policy** — only *your* repo may assume this role (`trust.json`):

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": { "Federated": "arn:aws:iam::ACCOUNT_ID:oidc-provider/token.actions.githubusercontent.com" },
    "Action": "sts:AssumeRoleWithWebIdentity",
    "Condition": {
      "StringEquals": { "token.actions.githubusercontent.com:aud": "sts.amazonaws.com" },
      "StringLike":   { "token.actions.githubusercontent.com:sub": "repo:OWNER/REPO:*" }
    }
  }]
}
```

**Permissions policy** — just enough to sync (`perms.json`):

```json
{
  "Version": "2012-10-17",
  "Statement": [
    { "Effect": "Allow", "Action": ["s3:ListBucket"], "Resource": "arn:aws:s3:::BUCKET" },
    { "Effect": "Allow", "Action": ["s3:PutObject","s3:GetObject","s3:DeleteObject"], "Resource": "arn:aws:s3:::BUCKET/*" }
  ]
}
```

Create the role and attach the policy:

```bash
aws iam create-role \
  --role-name elevate-deploy \
  --assume-role-policy-document file://trust.json

aws iam put-role-policy \
  --role-name elevate-deploy \
  --policy-name elevate-s3-sync \
  --policy-document file://perms.json
```

The role ARN is `arn:aws:iam::ACCOUNT_ID:role/elevate-deploy`.

## 5. Add the GitHub repo variables

Repo → **Settings → Secrets and variables → Actions → Variables** tab → New
repository variable (these are **variables**, not secrets — none are sensitive):

| Name            | Value                                          |
|-----------------|------------------------------------------------|
| `AWS_ROLE_ARN`  | `arn:aws:iam::ACCOUNT_ID:role/elevate-deploy`  |
| `AWS_REGION`    | `REGION`                                        |
| `S3_BUCKET`     | `BUCKET`                                         |

## 6. Deploy

Push to `main` (or run the workflow manually from the **Actions** tab). Then
visit the website endpoint:

```
http://BUCKET.s3-website.REGION.amazonaws.com
```

> Note: the S3 website endpoint is **HTTP only**. That's fine for gathering
> feedback. HTTPS + a custom domain (e.g. `elevatecoaching.co.uk`) comes with
> the CloudFront + DNS step later.

---

## Troubleshooting: `Not authorized to perform sts:AssumeRoleWithWebIdentity`

The provider, role, and trust policy can all look perfect and still fail. The
usual cause is that the token's **actual `sub` claim** isn't what you assume.
This account has **OIDC subject-claim customization** enabled, so the subject is
not the default `repo:OWNER/REPO:...` — it embeds immutable numeric IDs:

```
repo:mhazley@6282802/elevate@1306919298:ref:refs/heads/main
```

So the trust policy's `sub` condition must match **that**, not `repo:mhazley/elevate:*`:

```json
"StringLike": { "token.actions.githubusercontent.com:sub": "repo:mhazley@6282802/elevate@1306919298:*" }
```

To see the real claims your workflow is sending, temporarily add a step before
the AWS-credentials step:

```yaml
      - name: Decode real OIDC claims
        run: |
          IDTOKEN=$(curl -sS -H "Authorization: Bearer $ACTIONS_ID_TOKEN_REQUEST_TOKEN" \
            "$ACTIONS_ID_TOKEN_REQUEST_URL&audience=sts.amazonaws.com" | jq -r '.value')
          python3 - "$IDTOKEN" <<'PY'
          import sys, base64, json
          p = sys.argv[1].split('.')[1]; p += '=' * (-len(p) % 4)
          c = json.loads(base64.urlsafe_b64decode(p))
          for k in ("sub","aud","repository","job_workflow_ref"): print(f"{k} = {c.get(k)!r}")
          PY
```

Match the trust policy's `sub` to whatever `sub` prints, then remove the step.

---

### Later, before the real launch
- Put CloudFront in front for HTTPS, a custom domain, and proper caching.
- Swap the blanket `--cache-control "no-cache"` in the workflow for long-cache
  on `assets/` + `css/` and short-cache on the HTML.
- Add a dedicated `404.html` and point the bucket's error document at it.
