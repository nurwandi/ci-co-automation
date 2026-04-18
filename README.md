# Talenta HR Attendance Automation

Automated clock in/out for [Talenta HR](https://hr.talenta.co) using Playwright with stealth browser.

## How It Works

```
cron-job.org (08:00 & 18:00 WIB, Mon-Fri)
  → triggers GitHub Actions via workflow_dispatch
    → launches EC2 spot (Jakarta) as Tailscale VPN exit node
      → runner traffic routed through Indonesian IP
        → Playwright browser logs in & clicks Clock In/Out
          → Discord notification sent
            → EC2 terminated
```

## Quick Start (Local)

```bash
pnpm install
cp .env.example .env    # fill in TALENTA_EMAIL and TALENTA_PASSWORD
pnpm run clock-in       # or: pnpm run clock-out
```

## GitHub Actions Setup

### 1. Secrets

| Secret | Description |
|---|---|
| `TALENTA_EMAIL` | Talenta account email |
| `TALENTA_PASSWORD` | Talenta account password |
| `DISCORD_WEBHOOK_URL` | Discord webhook URL for notifications |
| `AWS_ROLE_ARN` | IAM role ARN (OIDC, for EC2 spot) |
| `TS_AUTHKEY` | Tailscale auth key (reusable, ephemeral, tag:ci) |
| `EC2_AMI_ID` | AMI ID with Tailscale pre-installed (ap-southeast-3) |
| `EC2_SUBNET_ID` | Public subnet in ap-southeast-3 |
| `EC2_SG_ID` | Security group (outbound all, inbound none) |

### 2. Variables

| Variable | Description |
|---|---|
| `CRON_ENABLED` | Set `true` to allow cron triggers |
| `TS_KEY_GENERATED_DATE` | Auth key generation date (YYYY-MM-DD) |
| `REMINDER_EMAIL` | Email for key expiry reminders |

### 3. Cron Trigger (cron-job.org)

Create 2 jobs that POST to GitHub API:

- **Clock In:** `0 8 * * 1-5` → `.../workflows/clock-in.yml/dispatches`
- **Clock Out:** `0 18 * * 1-5` → `.../workflows/clock-out.yml/dispatches`

Headers: `Authorization: Bearer <GITHUB_PAT>`, `Accept: application/vnd.github+json`, `Content-Type: application/json`
Body: `{"ref":"main"}`

### 4. AWS Setup

- OIDC provider for `token.actions.githubusercontent.com`
- IAM role with EC2 permissions (RunInstances, TerminateInstances, DescribeInstances, DescribeInstanceStatus, CreateTags) + `iam:CreateServiceLinkedRole` for Spot
- EC2 AMI: Ubuntu 24.04 + Tailscale installed, init script enables IP forwarding and joins as exit node

### 5. Tailscale Setup

- Auth key: reusable, ephemeral, tag `tag:ci`, 90-day expiry
- ACL: `"autoApprovers": {"exitNode": ["tag:ci"]}`
- Key rotation reminder workflow runs daily, notifies 15 days before expiry
