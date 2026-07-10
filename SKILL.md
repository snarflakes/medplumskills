---
name: "medplum-self-hosted"
description: "Deploy, configure, and manage a self-hosted Medplum FHIR server with admin and provider apps"
version: "v1"
date: "2026-07-10"
---

# Medplum Self-Hosted — Getting Started & Onboarding

Skill for deploying, configuring, and operating a self-hosted Medplum FHIR server with Admin and Provider apps.

## Architecture

```
┌─────────────────────────────────────────────────┐
│  Nginx reverse proxy (port 80/443)              │
│  ├── /        → Medplum App (port 3000)         │
│  └── /api      → Medplum Server (port 8103)     │
├─────────────────────────────────────────────────┤
│  Docker containers                               │
│  ├── medplum-postgres    (PostgreSQL 16)         │
│  ├── medplum-redis       (Redis 7)               │
│  ├── medplum-server      (Node.js FHIR API)      │
│  ├── medplum-app         (React+Vite admin SPA)  │
│  └── medplum-provider    (React+Vite clinical SPA)│
└─────────────────────────────────────────────────┘
```

- **Medplum Server** (v5.1.25): FHIR R4 API server + auth + storage + bots
- **Medplum App** (:3000): Admin/management React app
- **Medplum Provider** (:3001): Clinical charting React app (built separately)
- **PostgreSQL 16**: All FHIR data, indexes, auth state
- **Redis 7**: Caching, signing keys, session state
- **Nginx**: Reverse proxy on :80 → App + API on same origin (avoids CORS)

## Quick Start: Docker Deployment

### 1. Fresh install

```bash
# Download the official full-stack compose file
curl https://raw.githubusercontent.com/medplum/medplum/refs/heads/main/docker-compose.full-stack.yml > docker-compose.yml

# Start everything
docker compose up -d

# Wait for healthcheck (takes a few minutes on first start)
curl http://localhost:8103/healthcheck
```

Required environment variables in `docker-compose.yml`:
- `MEDPLUM_BASE_URL=http://your-server/` — public URL of your instance
- `MEDPLUM_ALLOWED_ORIGINS=*` — for development; restrict in production
- `RECAPTCHA_SITE_KEY=""` and `MEDPLUM_RECAPTCHA_SECRET_KEY=""` — disable recaptcha for self-hosted
- `MEDPLUM_ADMIN_PASSWORD=medplum_admin` — default; change immediately

### 2. First login (Super Admin)

1. Open `http://localhost:3000/` in a browser
2. Login: `admin@example.com` / `medplum_admin`
3. **Change the admin password immediately** via Admin UI → Users

This creates:
- A **User** (server-scoped, email + password)
- A **Project** (named "Super Admin" by default, with `superAdmin: true`)
- A **ProjectMembership** (linking User + Project + Practitioner profile)
- A **Practitioner** (the FHIR profile resource)

### 3. Super Admin features

Access at `/admin/super` in the app:
- Rebuild Structure Definitions, Search Parameters, Value Sets
- Reindex Resources
- Rebuild Compartments
- **Force Set Password** — override any user's password (super admin only)
- Purge Resources
- Invite users to any project

## Auth Model

### Users, Projects, and Memberships

```
User (server-scoped or project-scoped)
  └── ProjectMembership (per-project access)
        ├── profile: Practitioner | Patient | RelatedPerson
        ├── accessPolicy: AccessPolicy (what they can do)
        ├── access: [parameterized policies]
        └── admin: boolean
```

**Key concepts**:
- **User**: Authentication identity (email + password). Global, can belong to multiple projects.
- **Project**: Isolated resource container. Each org typically has one project.
- **ProjectMembership**: Links a User to a Project with a profile and access policy.
- **Profile**: A FHIR resource (Practitioner, Patient, or RelatedPerson) that represents the user within the project.
- **AccessPolicy**: Defines what resource types and operations the user can access.

### Server-scoped vs Project-scoped users

| | Server-scoped | Project-scoped |
|---|---|---|
| **Use for** | Admins, developers | Clinicians, patients |
| **Can access** | Multiple projects | Single project only |
| **Login** | Works everywhere | Requires `projectId` parameter |
| **Default for** | Practitioners | Patients |
| **Set via** | `scope: 'server'` in invite | `scope: 'project'` in invite |

⚠️ **Project-scoped users need `projectId` at login time** — this is critical for the Provider app. If a project-scoped user logs in without specifying the project, they get "User not found."

### Auth Flow (OAuth2 / PKCE)

```
1. POST /auth/login     → {code, login}
2. POST /auth/profile   → {code, ...}  (choose which profile/project to use)
3. POST /oauth2/token   → {access_token, refresh_token}
```

For project-scoped users, step 1 requires `projectId` parameter.

### Self-hosted email limitation

Self-hosted Medplum has **no email server configured by default**. This means:
- **Invite flows that send emails will silently fail** — no welcome email, no password reset
- **Workaround**: Use "Force Set Password" in Super Admin UI (`/admin/super`) or the `/auth/setpassword` API endpoint
- **To enable email**: Configure SMTP in Project settings (see Project SMTP docs) or in server config

## AccessPolicy

AccessPolicies control what resource types a user can access and what operations they can perform.

### Creating AccessPolicies

```json
{
  "resourceType": "AccessPolicy",
  "name": "Clinical Access",
  "resource": [
    { "resourceType": "Patient" },
    { "resourceType": "Encounter" },
    { "resourceType": "Observation" },
    { "resourceType": "Condition" },
    { "resourceType": "MedicationRequest" },
    { "resourceType": "DiagnosticReport" },
    { "resourceType": "Procedure" },
    { "resourceType": "AllergyIntolerance" },
    { "resourceType": "CarePlan" },
    { "resourceType": "DocumentReference" }
  ]
}
```

### AccessPolicy features

- **Block access**: Omit a resource type entirely → user cannot access it
- **Read-only**: `"readonly": true` on a resource entry
- **Criteria-based**: `"criteria": "Patient?address-state=CA"` — filter by FHIR search
- **Compartment-based**: `"criteria": "Observation?_compartment=Patient/xyz"`
- **Parameterized**: Use `%profile` or custom `%variable_name` variables, set on ProjectMembership.access
- **Field-level**: `readonlyFields`, `hiddenFields` arrays
- **Interaction-level**: `"interaction": ["read", "search"]` — fine-grained CRUD control
- **Write constraints**: FHIRPath expressions in `writeConstraint`

### Recommended AccessPolicy for clinicians (56 types)

```json
{
  "resourceType": "AccessPolicy",
  "name": "Practitioner - Clinical Access",
  "resource": [
    {"resourceType": "Patient"},
    {"resourceType": "Practitioner"},
    {"resourceType": "PractitionerRole"},
    {"resourceType": "Organization"},
    {"resourceType": "Location"},
    {"resourceType": "Encounter"},
    {"resourceType": "Observation"},
    {"resourceType": "Condition"},
    {"resourceType": "MedicationRequest"},
    {"resourceType": "Medication"},
    {"resourceType": "MedicationStatement"},
    {"resourceType": "DiagnosticReport"},
    {"resourceType": "Procedure"},
    {"resourceType": "AllergyIntolerance"},
    {"resourceType": "CarePlan"},
    {"resourceType": "CareTeam"},
    {"resourceType": "DocumentReference"},
    {"resourceType": "Binary"},
    {"resourceType": "Task"},
    {"resourceType": "Questionnaire"},
    {"resourceType": "QuestionnaireResponse"},
    {"resourceType": "PlanDefinition"},
    {"resourceType": "ActivityDefinition"},
    {"resourceType": "ServiceRequest"},
    {"resourceType": "Appointment"},
    {"resourceType": "Schedule"},
    {"resourceType": "Slot"},
    {"resourceType": "Communication"},
    {"resourceType": "Flag"},
    {"resourceType": "List"},
    {"resourceType": "Bundle"},
    {"resourceType": "Composition"},
    {"resourceType": "Immunization"},
    {"resourceType": "Specimen"},
    {"resourceType": "BodyStructure"},
    {"resourceType": "Device"},
    {"resourceType": "Media"},
    {"resourceType": "NutritionOrder"},
    {"resourceType": "Goal"},
    {"resourceType": "RiskAssessment"},
    {"resourceType": "ClinicalImpression"},
    {"resourceType": "EpisodeOfCare"},
    {"resourceType": "ReferralRequest"},
    {"resourceType": "SupplyRequest"},
    {"resourceType": "GuidanceResponse"},
    {"resourceType": "Mapping"},
    {"resourceType": "Bot"},
    {"resourceType": "Subscription"},
    {"resourceType": "StructureDefinition"},
    {"resourceType": "ValueSet"},
    {"resourceType": "CodeSystem"},
    {"resourceType": "ConceptMap"},
    {"resourceType": "SearchParameter"},
    {"resourceType": "OperationDefinition"},
    {"resourceType": "ChargeItemDefinition"},
    {"resourceType": "ExplanationOfBenefit"}
  ]
}
```

## Inviting Users

### Via Admin UI

1. Go to `/admin/invite` as a project admin
2. Select the target project (super admins can invite to any project)
3. Choose profile type: Practitioner, Patient, or RelatedPerson
4. Set name, email, AccessPolicy, admin flag
5. Submit — creates User + Profile + ProjectMembership

### Via API

```bash
curl -X POST "http://SERVER/admin/projects/PROJECT_ID/invite" \
  -H "Authorization: Bearer ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "resourceType": "Practitioner",
    "firstName": "Jane",
    "lastName": "Smith",
    "email": "jane@example.com",
    "sendEmail": false,
    "membership": {
      "admin": false,
      "accessPolicy": { "reference": "AccessPolicy/POLICY_ID" },
      "access": []
    }
  }'
```

### Setting passwords without email

Since self-hosted instances don't have email configured:

**Option A: Super Admin UI** — Go to `/admin/super`, find "Force Set Password", enter User ID and new password.

**Option B: API endpoint**:
```bash
# 1. Create a UserSecurityRequest
curl -X POST "http://SERVER/fhir/R4/UserSecurityRequest" \
  -H "Authorization: Bearer TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"resourceType": "UserSecurityRequest", "user": {"reference": "User/USER_ID"}, "type": "password-reset"}'

# 2. Use the setpassword endpoint with id and secret from the response
curl -X POST "http://SERVER/auth/setpassword" \
  -H "Content-Type: application/json" \
  -d '{"id": "SECURITY_REQUEST_ID", "secret": "SECRET", "password": "NewPassword123!"}'
```

## Provider App Deployment

### Build and Deploy

```bash
git clone https://github.com/medplum/medplum-provider.git
cd medplum-provider

# Create .env with your server URL
echo "MEDPLUM_BASE_URL=http://YOUR_SERVER:8103/" > .env

# Install and build
npm install
npm run build

# Serve with any static server (Docker, nginx, etc.)
```

### Docker deployment

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
# Configure nginx for SPA routing
```

### Provider App Auth

The Provider app uses **PKCE OAuth2 flow** — no ClientApplication or client secret needed. The app:
1. Redirects to `/oauth2/authorize` with `response_type=code` and PKCE challenge
2. User logs in at the Medplum auth page
3. Redirected back to Provider app with auth code
4. Exchanges code for access token at `/oauth2/token`

⚠️ **`MEDPLUM_BASE_URL` is baked into the build** — it's a Vite build-time env variable, not read at runtime. You must rebuild the Provider app when the server URL changes.

### Project-scoped users in Provider App

Project-scoped practitioners need their project ID included in the login request. The Provider app should handle this automatically if the user's ProjectMembership is properly configured, but if login fails with "User not found", check that:
1. The user has a ProjectMembership with a valid profile
2. The AccessPolicy attached to the membership is correct
3. The user is project-scoped (`project` field is set on the User resource)

## ⚠️ Hard-Learned Pitfalls

### 1. Never modify FHIR resource tables directly in PostgreSQL

Medplum maintains **indexed columns** (`__version`, `___securitySort`, `___tagSort`, `___compartmentIdentifierSort`, `__sharedTokens`, `__sharedTokensText`, etc.) that must be kept in sync with the `content` JSON column. Direct DB updates bypass the FHIR server's indexing logic, causing:
- Profile lookups to fail ("Profile not found" errors)
- Search to return stale data
- Auth to break

**Always use the FHIR API** for creating/updating resources. If you must touch the DB, you must:
1. Update ALL indexed columns correctly
2. Run "Rebuild Resources" + "Reindex Resources" from Super Admin (`/admin/super`)
3. Flush Redis: `docker exec medplum-redis-1 redis-cli -a YOUR_PASSWORD FLUSHALL`
4. Restart the Medplum server

### 2. Redis flushes can break auth

The Medplum server stores JWT signing keys in PostgreSQL but caches them in Redis. After a Redis FLUSHALL:
- The server regenerates **temporary signing keys** (you'll see "Generating temporary signing key" in logs)
- Existing auth tokens become invalid
- You must re-login to get new tokens
- The temporary key is lost on every restart, so tokens don't survive server restarts after a Redis flush

### 3. Never FLUSHALL Redis in production

Redis holds session state, signing keys, and cached resources. FLUSHALL destroys all of this. Use `DEL` for specific keys if needed, or let TTL-based expiration handle cleanup.

### 4. AccessPolicy circular dependency

If an AccessPolicy restricts writes to AccessPolicy resources (or Project, User, etc.), you can get locked out:
- The admin's own AccessPolicy may block them from editing AccessPolicies
- Workaround: Super Admin can bypass AccessPolicy checks
- Fix: Use direct DB updates (see pitfall #1) + reindex, or use the Super Admin context

### 5. Project-scoped user login requires projectId

If you invite a practitioner with `scope: 'project'`, they must include `projectId` in their login request. The Provider app should handle this, but raw API calls need:
```bash
curl -X POST /auth/login -d "email=user@example.com&password=pass&projectId=PROJECT_ID"
```

### 6. Signing key warnings are critical

If you see "Generating temporary signing key" in server logs, it means:
- Redis was flushed and the signing key was lost
- Tokens issued before the flush are now invalid
- Storage URLs (for binary resources) won't work correctly
- The key will be regenerated on every restart until the server can persist it properly

**Fix**: Restart the server after Redis is clean, then re-login to get fresh tokens.

### 7. Rate limiting on auth endpoints

Medplum rate-limits auth endpoints (default: 160 requests/minute/IP). If you're testing login flows repeatedly, you may hit the rate limit and get confusing errors. Wait a minute and try again.

### 8. Docker Compose restart order matters

When restarting the full stack:
```bash
docker compose restart  # May fail if postgres/redis aren't ready
```
Better approach:
```bash
docker compose down && docker compose up -d  # Clean restart
# Or: docker compose restart postgres redis && sleep 5 && docker compose restart medplum-server medplum-app
```

## FHIR Resources for Clinical Documentation

### SOAP Note as FHIR Bundle

A SOAP note maps to a FHIR transaction Bundle of multiple resources:

| SOAP Section | FHIR Resource | Purpose |
|---|---|---|
| Subjective | Observation | Patient-reported symptoms |
| Objective | Observation | Vital signs, physical exam findings |
| Assessment | Condition | Diagnoses, clinical assessments |
| Plan | CarePlan + MedicationRequest | Treatment plan, prescriptions |
| Overall | Composition | Container linking all sections |
| Context | Encounter | The visit itself |

### Visit Template Resources

- **PlanDefinition**: Defines the overall visit template
- **ActivityDefinition**: Defines individual activities within the visit
- **Questionnaire**: Structured data collection forms
- **ChargeItemDefinition**: Billing rules for the visit

### Creating Resources via FHIR API

```bash
# Always use the FHIR API — NEVER modify PostgreSQL directly
curl -X POST http://SERVER:8103/fhir/R4/Encounter \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d @encounter.json

# Transaction Bundle for creating multiple related resources
curl -X POST http://SERVER:8103/fhir/R4 \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d @soap_bundle.json
```

## Server Configuration

Key settings for self-hosted (in `docker-compose.yml` environment or `medplum.config.json`):

```json
{
  "port": 8103,
  "baseUrl": "http://your-server:8103/",
  "appBaseUrl": "http://your-server:3000/",
  "database": {
    "host": "postgres",
    "port": 5432,
    "dbname": "medplum",
    "user": "medplum",
    "password": "medplum"
  },
  "redis": {
    "host": "redis",
    "port": 6379,
    "password": "medplum"
  },
  "logLevel": "INFO"
}
```

### Project feature flags

Set in Project settings (requires Super Admin):
- `bots` — Enable Bot execution
- `ai` — Enable AI/Spaces features
- `cron` — Enable Bot CRON timers
- `email` — Enable Bot email sending
- `terminology` — Enable full ValueSet expansion
- `websocket-subscriptions` — Enable WebSocket subscriptions
- `transaction-bundles` — Enable atomic transaction processing

## Useful API Endpoints

```
GET  /healthcheck                    → Server health + version
POST /auth/login                     → Authenticate (returns code + login)
POST /auth/profile                   → Select profile (returns code for token exchange)
POST /oauth2/token                   → Exchange code for access token
POST /admin/projects/:id/invite      → Invite user to project
GET  /fhir/R4/{ResourceType}         → Search resources
GET  /fhir/R4/{ResourceType}/{id}    → Read resource
POST /fhir/R4/{ResourceType}         → Create resource
PUT  /fhir/R4/{ResourceType}/{id}    → Update resource
POST /fhir/R4                        → Transaction bundle
GET  /admin/super                    → Super Admin UI
POST /auth/setpassword               → Set password (needs id + secret from UserSecurityRequest)
```

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| "User not found" at login | Project-scoped user without projectId | Add `projectId` to login request |
| "Profile not found" at auth/profile | Corrupted indexed columns | Rebuild resources from Super Admin UI |
| "Invalid password" for valid passwords | Rate limiting or corrupted session | Wait 60s; if persistent, check server logs |
| "Generating temporary signing key" | Redis flush lost signing key | Restart server; re-login |
| 403 Forbidden on resource writes | AccessPolicy too restrictive | Update AccessPolicy via API (not DB!) |
| Provider app won't load | Wrong MEDPLUM_BASE_URL | Rebuild Provider app with correct URL |
| Can't create Secret resource | 400 Bad Request | Secrets may need special handling — check server config |
| DoseSpot "Access Check Failed" | Expected — paid integration | Not available in self-hosted without license |
| Invite email not sent | No SMTP configured | Use Force Set Password or /auth/setpassword |
| CORS errors | App and API on different origins | Use nginx reverse proxy on same origin |
| Recaptcha blocking login | Demo recaptcha keys in self-hosted | Set empty site key and secret key |

## Fresh Stack Reset

If you've corrupted the database or want a clean start:

```bash
cd /path/to/medplum
docker compose down -v  # Removes volumes (DESTRUCTIVE — deletes all data)
docker compose up -d    # Fresh start
# Wait for healthcheck, then visit http://localhost:3000/ to register
```

⚠️ This deletes ALL data — patients, encounters, users, everything. Only use for development/testing.

## Key References

- **Medplum Docs**: https://www.medplum.com/docs
- **User Management**: https://www.medplum.com/docs/user-management
- **Access Policies**: https://www.medplum.com/docs/access/access-policies
- **Self-Hosting Guide**: https://www.medplum.com/docs/self-hosting
- **FHIR R4 Spec**: https://hl7.org/fhir/R4/
- **Provider App Source**: https://github.com/medplum/medplum/tree/main/packages/provider