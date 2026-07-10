---
name: medplum-deploy
description: "Deploying self-hosted Medplum with Docker Compose. Use when setting up, configuring, or troubleshooting a fresh Medplum instance."
---

# Deploy — Docker Compose, First Login, Provider App

Stock LLMs will tell you to just `docker compose up` and it works. They're wrong about the config specifics, the recaptcha trap, the Provider app build requirements, and what happens without an email server.

## Docker Compose Setup

```bash
curl https://raw.githubusercontent.com/medplum/medplum/refs/heads/main/docker-compose.full-stack.yml > docker-compose.yml
docker compose up -d
```

### Required environment variables

The official compose file needs these overrides for self-hosted:

```yaml
# In docker-compose.yml, medplum-server environment:
MEDPLUM_BASE_URL: "http://YOUR_HOST/"  # Public URL — NOT localhost
MEDPLUM_ALLOWED_ORIGINS: "*"           # Dev only — restrict in production
RECAPTCHA_SITE_KEY: ""                 # Empty = disabled (demo keys DON'T work)
MEDPLUM_RECAPTCHA_SECRET_KEY: ""       # Empty = disabled
```

🔴 **Demo recaptcha keys don't work in self-hosted.** The compose file ships demo keys that fail verification. Set both to empty strings to skip recaptcha entirely.

🔴 **`command: ["env"]` is intentional.** The medplum-server image's entrypoint reads env vars and generates `medplum.config.json`. If you remove `command: ["env"]`, the server crashes trying to read a config file that doesn't exist.

## First Login

1. Open `http://YOUR_HOST:3000/`
2. Login: `admin@example.com` / `medplum_admin`
3. **Change the admin password immediately.** The default password is public knowledge.

This creates a User, Project ("Super Admin" — just a display name, not a role), ProjectMembership, and Practitioner.

## Provider App Deployment

🔴 **`MEDPLUM_BASE_URL` is baked at build time by Vite.** It's not read at runtime. If your server URL changes, you must rebuild the entire Provider app.

```bash
git clone https://github.com/medplum/medplum-provider.git
cd medplum-provider
echo "MEDPLUM_BASE_URL=http://YOUR_HOST/" > .env
npm ci
npm run build
```

For Docker:

```dockerfile
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
ARG MEDPLUM_BASE_URL=http://localhost:8103/
ENV MEDPLUM_BASE_URL=$MEDPLUM_BASE_URL
RUN npm run build

FROM nginx:alpine
COPY --from=builder /app/dist /usr/share/nginx/html
# nginx.conf must proxy /auth/, /fhir/, /storage/ back to Medplum server
```

### Provider App Auth

Uses PKCE OAuth2 flow — no ClientApplication or client secret needed. The app:
1. Redirects to `/oauth2/authorize` with PKCE challenge
2. User logs in at Medplum auth page
3. Redirected back with auth code
4. Exchanges code for access token

### CORS and same-origin

🔴 **Don't run the Provider app on a different origin than the server.** CORS issues are constant. Use nginx reverse proxy on the same origin (port 80/443) and proxy `/auth/`, `/fhir/`, `/storage/`, `/admin/`, `/healthcheck` to the Medplum server.

## Self-Hosted Has No Email

🔴 **Invite flows that send emails silently fail.** No welcome email, no password reset email. To set passwords:

**Option A: Super Admin UI** — Go to `/admin/super`, "Force Set Password", enter User ID + new password.

**Option B: API** — Create a `UserSecurityRequest`, then call `/auth/setpassword` with the id and secret:

```bash
# Create security request
curl -X POST "http://SERVER/fhir/R4/UserSecurityRequest" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"resourceType": "UserSecurityRequest", "user": {"reference": "User/USER_ID"}, "type": "password-reset"}'

# Set password using response id + secret
curl -X POST "http://SERVER/auth/setpassword" \
  -H "Content-Type: application/json" \
  -d '{"id": "SECURITY_REQUEST_ID", "secret": "***", "password": "***"}'
```

**Option C: Browser URL** — Navigate to `http://SERVER/setpassword/{id}/{secret}`.

## Project Feature Flags

Enable via Super Admin → Project settings:

| Flag | Purpose |
|------|---------|
| `bots` | Bot execution |
| `ai` | AI/Spaces features |
| `cron` | Bot CRON timers |
| `email` | Bot email sending |
| `terminology` | Full ValueSet expansion |
| `websocket-subscriptions` | WebSocket subscriptions |
| `transaction-bundles` | Atomic transaction processing |

## Health Check

```bash
curl http://YOUR_HOST:8103/healthcheck
# Returns: {"status": "ok", "version": "5.1.25"}
```

If this returns anything other than `{"status": "ok"}`, the server isn't ready.