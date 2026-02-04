# Slack → OpenAI → Approval → LinkedIn (n8n Workflow)

This repository contains everything needed to run an **end-to-end automation** that:

1. Listens to messages in Slack
2. Sends them to OpenAI for drafting a LinkedIn post
3. Sends the draft to Slack for approval
4. Posts the approved content to LinkedIn (personal or company page)

The workflow is built in **n8n**, runs locally via **Docker**, and uses **ngrok** for Slack webhooks.

---

## Architecture Overview

```
Slack (Events API)
   ↓
n8n Webhook
   ↓
OpenAI (Draft LinkedIn Post)
   ↓
Slack (Approval Message)
   ↓
Wait for Approval (Webhook)
   ↓
Decision Logic
   ↓
LinkedIn Post (Personal or Company)
```

---

## Prerequisites

Each teammate must install the following **locally**:

### Required Software
- **Docker Desktop**  
  https://www.docker.com/products/docker-desktop
- **ngrok**  
  https://ngrok.com/download
- **Git**  
  https://git-scm.com/
- A modern browser (Chrome recommended)

### Required Accounts
- Slack workspace where you can create apps
- LinkedIn account (admin access required for company posting)
- OpenAI account with API access

---

## Repository Structure

```
slack-to-linkedin-n8n/
│
├── docker-compose.yml
├── env.example
├── .gitignore
│
├── workflows/
│   └── slack_to_linkedin_approval.json
│
├── scripts/
│   ├── start.ps1
│   └── start.sh
│
└── README.md
```

---

## Step 1 — Clone the Repository

```bash
git clone https://github.com/YOUR_ORG/slack-to-linkedin-n8n.git
cd slack-to-linkedin-n8n
```

---

## Step 2 — Create `.env` File

Copy the example file and fill in placeholders.

```bash
cp env.example .env
```

### `env.example` explained

```env
# n8n runtime
N8N_HOST=localhost
N8N_PORT=5678
N8N_PROTOCOL=http

# This must match your ngrok https URL
WEBHOOK_URL=https://YOUR_NGROK_DOMAIN

# Required for credential encryption (set once)
N8N_ENCRYPTION_KEY=REPLACE_WITH_RANDOM_32_CHAR_STRING

# Slack channels (IDs, not names)
LOG_CHANNEL_ID=C012ABCDEF
APPROVAL_CHANNEL_ID=C045XYZ123

# OpenAI
OPENAI_MODEL=gpt-4.1-mini

# LinkedIn
# Personal posting:
LINKEDIN_AUTHOR_URN=urn:li:person:XXXXXXXX

# Company posting (optional):
# LINKEDIN_ORG_URN=urn:li:organization:XXXXXXXX
```

⚠️ **Never commit `.env` to GitHub**

---

## Step 3 — Start n8n (Docker)

```bash
docker compose up -d
```

Verify:
- n8n UI opens at: http://localhost:5678

---

## Step 4 — Start ngrok

In a **separate terminal**:

```bash
ngrok http 5678
```

You will see output like:

```
Forwarding https://revisional-xxxxx.ngrok-free.dev -> http://localhost:5678
```

👉 **Copy the HTTPS ngrok URL** — you will need it multiple times.

---

## Step 5 — Import the Workflow into n8n

1. Open http://localhost:5678
2. Click **Import Workflow**
3. Import:
   ```
   workflows/slack_to_linkedin_approval.json
   ```
4. Save the workflow

---

## Step 6 — Create Credentials in n8n

### OpenAI
- Credentials → New → **HTTP Header Auth** (or OpenAI if using native node)
- Header:
  ```
  Authorization: Bearer sk-xxxx
  ```

### Slack
- Credentials → New → **Slack**
- Use **Bot User OAuth Token**
- Token starts with `xoxb-...`

### LinkedIn
- Credentials → New → **LinkedIn OAuth2 API**
- Requires LinkedIn App (see below)

---

## Step 7 — Slack App Setup (Critical)

### Create Slack App
1. https://api.slack.com/apps → **Create New App**
2. Choose **From scratch**
3. Select your workspace

### OAuth & Permissions
Add bot scopes:
- `chat:write`
- `channels:read`
- `channels:history`
- `groups:read` (if using private channels)

Install app to workspace.

### Event Subscriptions
1. Enable **Event Subscriptions**
2. Request URL:
   ```
   https://YOUR_NGROK_DOMAIN/webhook/slack/events
   ```
3. Wait for **Verified**
4. Subscribe to bot events:
   - `message.channels`
   - `message.groups` (if private)

### Invite bot to channels
In Slack:
```
/invite @YourBotName
```

---

## Step 8 — LinkedIn App Setup

### Create LinkedIn App
https://www.linkedin.com/developers/apps

### OAuth Settings
- Redirect URL:
  ```
  http://localhost:5678/rest/oauth2-credential/callback
  ```

### Required Scopes
Personal posting:
- `w_member_social`
- `openid`
- `profile`
- `email`

Company posting:
- `w_organization_social`
- Must be approved by LinkedIn
- You must be an admin of the Page

⚠️ After changing scopes, **re-authenticate** in n8n.

---

## Step 9 — Update Webhook URLs After Restart

Every time ngrok restarts:

1. Copy new ngrok URL
2. Update:
   - Slack Event Subscriptions → Request URL
   - `.env` → `WEBHOOK_URL`
3. Restart n8n if `.env` changed

---

## Step 10 — Activate the Workflow

In n8n:
- Open the workflow
- Toggle **Active** → ON

⚠️ Slack production webhooks **only work when Active**

---

## Step 11 — Test End-to-End

1. Send a message in Slack source channel:
   ```
   hello world
   ```
2. n8n drafts LinkedIn post via OpenAI
3. Approval message appears in approval channel
4. Click **Approve**
5. Post appears on LinkedIn

---

## Common Issues & Fixes

### “Unknown webhook”
- Workflow not Active
- Wrong URL (`/webhook-test` instead of `/webhook`)
- ngrok URL changed

### Slack `channel_not_found`
- Use **channel ID**, not name
- Invite bot to channel

### LinkedIn 422 error
- Missing `visibility`, `lifecycleState`, or `specificContent`
- Body must be valid JSON or Expression mode

### Approval loses draft text
- Ensure **Store Draft → Merge → Approval Decision** pattern is used

---

## Security Notes

- Never commit `.env`
- Never commit `n8n_data/`
- Credentials are encrypted locally using `N8N_ENCRYPTION_KEY`

---

## Optional Improvements

- Replace ngrok with Cloudflare Tunnel
- Add Docker health checks
- Add retry logic for LinkedIn/OpenAI
- Convert to multi-tenant SaaS

---

## Support

If something breaks:
1. Check ngrok is running
2. Check workflow is Active
3. Check Slack Event Subscriptions show **Verified**
4. Check n8n execution logs

---

## Final Notes

This setup is intentionally **explicit and reproducible**.  
Anyone following this README should be able to stand up the full pipeline on their own machine.

---

If you want, I can also provide:
- A hardened production `docker-compose.yml`
- A troubleshooting decision tree
- A video walkthrough outline
- A GitHub template repo

