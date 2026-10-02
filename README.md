# Project 1 — Static website on S3 and CloudFront

**Status:** Concluded lab (built, verified, teardown documented).

**Goal:** Publish a static site over HTTPS without EC2, keep the S3 bucket private, and serve it from CloudFront edge locations.

## Architecture

```
Browser  --HTTPS-->  CloudFront distribution  --OAC-->  private S3 bucket
                         |
                    optional WAF later
```

| Piece | Role |
|-------|------|
| S3 bucket | Origin (objects only; no public website hosting) |
| Origin Access Control (OAC) | CloudFront is allowed to `s3:GetObject`; the public is not |
| CloudFront | TLS, caching, global PoPs |
| `index.html` / `error.html` | Default root and custom 403/404 page |

This is the Practitioner pattern: **object storage + CDN + least privilege**, not “turn on S3 static website hosting and make the bucket public.”

## What you will have when done

- A CloudFront URL like `https://dve6zov3uqhgsdhjfk.cloudfront.net/` that loads this project’s page
- S3 Block Public Access **on**
- No EC2, no load balancer

## Prerequisites

- IAM user who can create S3 buckets and CloudFront distributions
- Billing alarm already in place
- Region for the bucket

---

## Step 1 — Create the bucket

1. Open **Amazon S3** → **Create bucket**.
2. **Bucket name:** globally unique, e.g. `clf-p1-site-<your-initials>`.
3. **AWS Region:** your chosen Region.
4. **Object Ownership:** ACLs disabled (recommended).
5. **Block Public Access:** leave **all four blocks ON**.
6. **Bucket Versioning:** optional (on is safer for labs).
7. **Default encryption:** SSE-S3 is enough.
8. Create the bucket.

## Step 2 — Upload the site files

From this folder, upload:

- `index.html`
- `styles.css`
- `error.html`

Console:

1. Open the bucket → **Upload**.
2. Add the three files.
3. Upload with default settings.

CLI (optional), from this directory:

```bash
BUCKET=your-bucket-name
aws s3 cp index.html "s3://$BUCKET/index.html" --content-type text/html
aws s3 cp styles.css "s3://$BUCKET/styles.css" --content-type text/css
aws s3 cp error.html "s3://$BUCKET/error.html" --content-type text/html
```

Do **not** enable “Static website hosting” on the bucket. CloudFront will be the website.

## Step 3 — Create Origin Access Control

1. Open **CloudFront** → **Origin access** → **Create control setting**.
2. Name: `clf-p1-oac`.
3. **Signing behavior:** Sign requests.
4. **Origin type:** S3.
5. Create.

## Step 4 — Create the distribution

1. CloudFront → **Create distribution**.
2. **Origin domain:** pick your **bucket** (the REST endpoint, e.g. `bucket.s3.eu-west-1.amazonaws.com`), not the static website endpoint.
3. **Origin access:** Origin access control settings → select `clf-p1-oac`.
4. When prompted, choose to **copy the bucket policy** CloudFront suggests (you will paste it in Step 5).
5. **Viewer protocol policy:** Redirect HTTP to HTTPS.
6. **Allowed HTTP methods:** GET, HEAD.
7. **Default root object:** `index.html`.
8. **WAF:** Do not enable for this lab (cost). You can name it on the exam without turning it on.
9. Create distribution and wait until the status is **Enabled** (several minutes).

Note the **Distribution domain name**.

## Step 5 — Bucket policy (OAC)

On the bucket → **Permissions** → **Bucket policy**. Use CloudFront’s generated policy, or this shape (replace Region, bucket, and distribution ID):

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowCloudFrontOAC",
      "Effect": "Allow",
      "Principal": {
        "Service": "cloudfront.amazonaws.com"
      },
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::YOUR-BUCKET-NAME/*",
      "Condition": {
        "StringEquals": {
          "AWS:SourceArn": "arn:aws:cloudfront::YOUR-ACCOUNT-ID:distribution/YOUR-DISTRIBUTION-ID"
        }
      }
    }
  ]
}
```

Confirm **Block Public Access** is still fully on.

## Step 6 — Custom error page (optional but recommended)

Distribution → **Error pages** → Create custom error response:

- HTTP 403 → respond with `/error.html`, status 404 (or 403)
- HTTP 404 → `/error.html`, status 404

This avoids an ugly XML AccessDenied body when someone hits a missing path.

## Step 7 — Verify

1. Open `https://<distribution-domain>/` — you should see the Cloud Practitioner Project 1 page.
2. Open the S3 **object URL** (bucket website or REST URL) in an incognito window — it should **fail** (Access Denied). That proves the bucket is not public.
3. Change `index.html`, re-upload, then **invalidate** CloudFront path `/*` if you still see the old page (cache).

## Teardown (do this unless you intend to keep the site)

1. CloudFront → disable the distribution → wait → **delete** the distribution.
2. Delete the OAC if nothing else uses it.
3. S3 → empty the bucket → delete the bucket.

CloudFront deletes can take several minutes. Do not leave a disabled distribution forever if you are cost-sensitive; delete it.