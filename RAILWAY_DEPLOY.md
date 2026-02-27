# Deploy DocuSeal on Railway for Contract Automation

## Context

Ally Renewal needs a private DocuSeal instance to automate sales contracts. The flow:

1. Custom app triggers DocuSeal API (via webhook or direct call)
2. DocuSeal generates the contract from a template, pre-filling customer details
3. DocuSeal emails the signing link to the customer AND returns the signing URL to the app
4. The app can also send the signing URL via text/embed in its own UI
5. Customer signs -> DocuSeal fires a webhook back to the app
6. Signed PDF stored in Supabase Storage (S3-compatible)

Domain: `sign.allyrenewal.com`

---

## Architecture

```
Custom App (webhook trigger)
    |
    v
+-------------------------------------------+
|  Railway Project                          |
|                                           |
|  +--------------+   +------------------+ |
|  | PostgreSQL   |-->| DocuSeal Service | |
|  | (plugin)     |   | (Docker image)   | |
|  +--------------+   |                  | |
|                      | Puma :3000      | |
|                      | + Sidekiq embed | |
|                      | + Redis embed   | |
|                      +--------+---------+ |
|                               |           |
+-------------------------------+-----------+
                                |
                       +--------v--------+
                       | Supabase Storage|
                       | (S3-compatible) |
                       +-----------------+
```

Single Railway service. No separate Redis or Sidekiq -- both embedded in the Puma process (single-tenant mode).

---

## Step-by-Step Deployment

### 1. Create Supabase Storage Bucket

In your Supabase project dashboard:

1. Go to **Storage** -> **New Bucket**
2. Name: `docuseal-contracts`
3. Set to **Private** (DocuSeal uses presigned URLs)
4. Go to **Settings** -> **API** -> copy:
   - Project URL (e.g., `https://abcdefgh.supabase.co`)
   - `service_role` key (this is the secret key -- treat as a credential)
5. Go to **Settings** -> **General** -> note your project region

### 2. Create Railway Project

1. Go to [railway.com/new](https://railway.com/new) -> **Empty Project**
2. Click **"+ New"** -> **"Database"** -> **"PostgreSQL"**
3. Click **"+ New"** -> **"Docker Image"** -> enter: `docuseal/docuseal:latest`

### 3. Link Database

1. Click the DocuSeal service -> **Variables** tab
2. Click **"Add Reference"** -> select `DATABASE_URL` from the PostgreSQL service

### 4. Set Environment Variables

Add all of these in the DocuSeal service **Variables** tab. See `.env.example` for a ready-to-fill template.

```env
# -- Core -------------------------------------------
RAILS_ENV=production
PORT=3000
SECRET_KEY_BASE=<generate with: openssl rand -hex 64>
FORCE_SSL=true
APP_URL=https://sign.allyrenewal.com

# -- Supabase S3-Compatible Storage ----------------
S3_ATTACHMENTS_BUCKET=docuseal-contracts
AWS_ACCESS_KEY_ID=<supabase-project-ref>
AWS_SECRET_ACCESS_KEY=<supabase-service-role-key>
AWS_REGION=<supabase-project-region, e.g. us-east-1>
S3_ENDPOINT=https://<supabase-project-ref>.supabase.co/storage/v1/s3

# -- Email (so DocuSeal can send signing links) -----
SMTP_ADDRESS=<your-smtp-host>
SMTP_PORT=587
SMTP_USERNAME=<your-smtp-username>
SMTP_PASSWORD=<your-smtp-password>
SMTP_DOMAIN=allyrenewal.com

# -- Security ----------------------------------------
SIDEKIQ_BASIC_AUTH_PASSWORD=<generate with: openssl rand -hex 24>
```

### 5. Configure Networking & Domain

1. DocuSeal service -> **Settings** -> **Networking** -> **Custom Domain**
2. Enter: `sign.allyrenewal.com`
3. Railway provides a CNAME target -- add it in your DNS:
   ```
   CNAME  sign  ->  <railway-provided-target>.railway.app
   ```
4. Wait for SSL provisioning (~2-5 minutes)

### 6. Deploy

Railway deploys automatically after adding the Docker image and env vars. Monitor:
1. Click the deployment -> **View Logs**
2. Look for: `Puma starting in single mode...` and `Listening on http://0.0.0.0:3000`

### 7. Initial Setup

1. Visit `https://sign.allyrenewal.com/setup`
2. Create admin account (email + password)
3. Set timezone, locale, organization name -> "Ally Renewal"
4. App auto-generates eSign certificates

### 8. Post-Setup Configuration

In the DocuSeal admin UI:
1. **Settings -> API** -> Create API token -> save it securely
2. **Settings -> Webhooks** -> Add webhook:
   - URL: your app's webhook endpoint
   - Events: `submission.completed`, `form.completed` (at minimum)
3. **Settings -> Email** -> verify SMTP is working (send test)

---

## API Integration Flow

Your custom app will use this flow:

### Create a submission (send contract for signing)

```bash
curl -X POST https://sign.allyrenewal.com/api/submissions \
  -H "X-Auth-Token: YOUR_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "template_id": 1,
    "send_email": true,
    "submitters": [
      {
        "role": "Customer",
        "email": "customer@example.com",
        "name": "Jane Doe",
        "phone": "+15551234567",
        "external_id": "crm-deal-123",
        "fields": [
          { "name": "Contract Value", "default_value": "$25,000" },
          { "name": "Start Date", "default_value": "2026-03-01" }
        ]
      }
    ]
  }'
```

### Response (includes signing URL)

```json
[
  {
    "id": 1,
    "slug": "abc123def456",
    "email": "customer@example.com",
    "status": "sent",
    "embed_src": "https://sign.allyrenewal.com/s/abc123def456",
    "external_id": "crm-deal-123"
  }
]
```

Key fields:
- `embed_src` -- the signing URL. Send this via text, embed in your app, or let DocuSeal email it (controlled by `send_email: true/false`)
- `external_id` -- your CRM's deal/contract ID for correlation
- `slug` -- unique submitter identifier

### Receive webhook on completion

DocuSeal POSTs to your webhook URL when signing completes. Payload includes the signed document URLs and all field values.

---

## Supabase S3 Compatibility

Supabase Storage exposes an S3-compatible API at:
```
https://<project-ref>.supabase.co/storage/v1/s3
```

DocuSeal's `config/storage.yml` sets `force_path_style: true` when `S3_ENDPOINT` is present, which is required for Supabase's S3 gateway. The credentials mapping:

| DocuSeal Env Var | Supabase Value |
|---|---|
| `AWS_ACCESS_KEY_ID` | Your Supabase project ref (e.g., `abcdefgh`) |
| `AWS_SECRET_ACCESS_KEY` | Your `service_role` secret key |
| `AWS_REGION` | Your project's region (e.g., `us-east-1`) |
| `S3_ENDPOINT` | `https://<project-ref>.supabase.co/storage/v1/s3` |
| `S3_ATTACHMENTS_BUCKET` | The bucket name you created (`docuseal-contracts`) |

---

## railway.json

The `railway.json` in this repo configures Railway for Approach A (deploy from GitHub):

```json
{
  "$schema": "https://railway.com/railway.schema.json",
  "build": {
    "dockerfilePath": "Dockerfile"
  },
  "deploy": {
    "startCommand": "/app/bin/bundle exec puma -C /app/config/puma.rb --dir /app",
    "healthcheckPath": "/up",
    "healthcheckTimeout": 30,
    "restartPolicyType": "ON_FAILURE",
    "restartPolicyMaxRetries": 5
  }
}
```

**Recommended approach: Approach B (Docker image `docuseal/docuseal:latest`)**. No repo fork needed, no build time, updates by redeploying. The `railway.json` is only needed if you deploy from this GitHub repo instead.

---

## Smoke Tests

After deployment, run these in order:

```bash
# 1. Health check
curl -s -o /dev/null -w "%{http_code}" https://sign.allyrenewal.com/up
# -> 200

# 2. Setup page
curl -s -o /dev/null -w "%{http_code}" https://sign.allyrenewal.com/setup
# -> 200 (first time) or 302 (already set up)

# 3. API auth (after creating API token)
curl -s -H "X-Auth-Token: YOUR_TOKEN" https://sign.allyrenewal.com/api/templates
# -> JSON response (empty array is fine)

# 4. Upload a test PDF template via the UI
# Go to Templates -> New -> upload a contract PDF
# If it renders, storage (Supabase S3) is working

# 5. Create a test submission via API
# Use the curl command from the API section above with your own email
# Verify you receive the signing email AND get the embed_src URL

# 6. Sign the test contract
# Open the embed_src URL -> complete signing
# Verify your webhook endpoint receives the completion event
```

---

## Ops & Security

- **Postgres backups**: Enable in Railway PostgreSQL service -> Settings -> Backups (daily)
- **Sidekiq dashboard**: `https://sign.allyrenewal.com/jobs` -- protected by `SIDEKIQ_BASIC_AUTH_PASSWORD`
- **SECRET_KEY_BASE**: Generate once, never change (encrypted data depends on it)
- **Pin Docker image version**: Use `docuseal/docuseal:1.x.x` instead of `:latest` for stability, update deliberately
- **FORCE_SSL=true**: Enforces HTTPS redirects and secure cookies

---

## Troubleshooting

| Issue | Fix |
|---|---|
| "No open port detected" | Ensure `PORT=3000` is set |
| Health check fails | Increase `healthcheckTimeout` to 60; check logs for migration errors |
| Storage upload fails | Verify Supabase bucket exists, credentials correct, bucket is private |
| `SECRET_KEY_BASE` errors | Must be hex string, at least 64 chars: `openssl rand -hex 64` |
| DB connection errors | Verify `DATABASE_URL` reference points to the Postgres plugin |
| OOM crashes | DocuSeal needs ~512MB+. Set `RAILS_MAX_THREADS=5` if memory-constrained |
| Emails not sending | Verify SMTP vars, check logs for delivery errors, test in Settings -> Email |
| Webhook not firing | Check Settings -> Webhooks -> Events tab for delivery status and errors |
