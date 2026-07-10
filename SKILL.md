---
name: "medplum-self-hosted"
description: "Deploy, configure, and manage a self-hosted Medplum FHIR server with admin and provider apps"
---

# Medplum Self-Hosted Deployment & Management

## Overview

Medplum is an open-source healthcare development platform (FHIR server + apps) that can be self-hosted. It uses a headless architecture: the server exposes a FHIR API, and you build or deploy front-end apps (Admin, Provider) separately.

**Open-core model**: FHIR server + Provider app + Bot framework = free/open-source. Health Gorilla connector, advanced integrations, and managed hosting = paid.

## Architecture

```
┌──────────────────────────────────────────────┐
│  Nginx reverse proxy (port 80/443)           │
│  ├── /        → Medplum App (port 3000)      │
│  └── /api     → Medplum Server (port 8103)   │
├──────────────────────────────────────────────┤
│  Docker containers                            │
│  ├── medplum-postgres    (PostgreSQL 16)     │
│  ├── medplum-redis       (Redis)              │
│  ├── medplum-server      (Node.js API)       │
│  ├── medplum-app         (Vite SPA)          │
│  └── medplum-provider    (Vite SPA, port 3001)│
└──────────────────────────────────────────────┘
```

### Key components:
- **Medplum Server**: FHIR R4 API server (Node.js, port 8103)
- **Medplum App**: Admin/management UI (React+Vite, port 3000)
- **Provider App**: Clinical charting UI (React+Vite, port 3001)
- **PostgreSQL**: All data storage
- **Redis**: Caching, session state, signing keys

## Quick Start: Docker Deployment

### 1. Fresh install with Docker Compose

```bash
# Download the official full-stack compose file
curl https://raw.githubusercontent.com/medplum/medplum/refs/heads/main/docker-compose.full-stack.yml > docker-compose.yml

# Start everything
docker compose up -d

# Wait for healthcheck (takes a few minutes on first start)
curl http://localhost:8103/healthcheck
```

The full-stack Docker Compose includes: redis, postgres, medplum-server, medplum-app — all pre-configured to work together.

### 2. First login (Super Admin)

On first launch, visit `http://localhost:3000/` and register a new account. This creates:
- A **User** (server-scoped, email + password)
- A **Project** (named "Super Admin" by default, with `superAdmin: true`)
- A **ProjectMembership** (linking User + Project + Practitioner profile)
- A **Practitioner** (the FHIR profile resource)

The first user is automatically a **Super Admin** with full access to all projects and system resources.

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

### client_credentials Flow (Machine-to-Machine API Access)

For programmatic API access (scripts, bots, server-side apps), use `client_credentials` grant type.

### ⚠️ Critical: ClientApplication alone is NOT enough

A ClientApplication by itself will return "Invalid client" on token exchange. You **MUST** also create a **ProjectMembership** linking the ClientApplication to a project. This is not documented in Medplum's client-credentials docs.

### Setup steps (all via FHIR API, NEVER direct DB edits):

```bash
# Step 1: Get an admin token via PKCE flow (see Auth Flow section below)
# You need a valid Bearer token to create ClientApplication via API

# Step 2: Create the ClientApplication
CLIENT_ID=$(curl -s -X POST http://SERVER:8103/fhir/R4/ClientApplication \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "resourceType": "ClientApplication",
    "name": "my-api-client",
    "grantType": ["client_credentials"],
    "pkceOptional": true
  }' | jq -r '.id')

# Step 3: Set the secret (use PATCH, not PUT)
SECRET="$(openssl rand -base64 32)"
curl -s -X PATCH http://SERVER:8103/fhir/R4/ClientApplication/$CLIENT_ID \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json-patch+json" \
  -d '[{"op":"add","path":"/secret","value":"'$SECRET'"}]'

# Step 4: Create ProjectMembership (THIS IS THE MISSING PIECE)
curl -s -X POST http://SERVER:8103/fhir/R4/ProjectMembership \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d "{
    \"resourceType\": \"ProjectMembership\",
    \"project\": {\"reference\": \"Project/$PROJECT_ID\"},
    \"user\": {\"reference\": \"ClientApplication/$CLIENT_ID\"},
    \"profile\": {\"reference\": \"ClientApplication/$CLIENT_ID\"},
    \"admin\": true
  }"

# Step 5: Exchange for token
curl -X POST http://SERVER:8103/oauth2/token \
  -H "Content-Type: application/x-www-form-urlencoded" \
  --data-urlencode "grant_type=client_credentials" \
  --data-urlencode "client_id=$CLIENT_ID" \
  --data-urlencode "client_secret=$SECRET" \
  --data-urlencode "scope=openid"
```

### Key facts:
- `client_id` = the **resource ID** (UUID) of the ClientApplication, NOT a separate field
- `client_secret` = the plaintext `secret` field stored on the ClientApplication
- Medplum compares secrets with **timing-safe string comparison** (not bcrypt hash)
- The ProjectMembership must have `admin: true` or an appropriate AccessPolicy
- Tokens expire in 1 hour (3600 seconds)

## Auth Flow (OAuth2 / PKCE)

```
1. Generate code_verifier and code_challenge (S256)
2. POST /auth/login     → {code, login}  (include projectId for project-scoped users)
3. POST /auth/profile   → {code, ...}  (only if multiple profiles; skip if auto-selected)
4. POST /oauth2/token   → {access_token, refresh_token, id_token}
```

For project-scoped users, step 1 requires `projectId` parameter.

⚠️ **`/auth/profile` takes a ProjectMembership ID, NOT a Practitioner ID.** If the user has only one ProjectMembership, the profile is auto-selected and `/auth/profile` returns "Login profile already set" — skip directly to token exchange.

### Full PKCE login script:

```bash
# Generate PKCE verifier and challenge
VERIFIER=$(python3 -c "import base64,hashlib,os; v=base64.urlsafe_b64encode(os.urandom(32)).rstrip(b'=').decode(); print(v)")
CHALLENGE=$(python3 -c "import base64,hashlib; print(base64.urlsafe_b64encode(hashlib.sha256('$VERIFIER'.encode()).digest()).rstrip(b'=').decode())")

# Step 1: Login
LOGIN_RESP=$(curl -s -X POST http://SERVER:8103/auth/login \
  -H "Content-Type: application/json" \
  -d "{\"email\":\"admin@example.com\",\"password\":\"medplum_admin\",\"scope\":\"openid\",\"nonce\":\"nonce123\",\"codeChallenge\":\"$CHALLENGE\",\"codeChallengeMethod\":\"S256\",\"projectId\":\"$PROJECT_ID\"}")

CODE=$(echo $LOGIN_RESP | jq -r '.code')

# Step 2: Skip /auth/profile if single membership (auto-selected)

# Step 3: Exchange code for token
TOKEN_RESP=$(curl -s -X POST http://SERVER:8103/oauth2/token \
  -H "Content-Type: application/x-www-form-urlencoded" \
  --data-urlencode "grant_type=authorization_code" \
  --data-urlencode "code=$CODE" \
  --data-urlencode "code_verifier=$VERIFIER" \
  --data-urlencode "redirect_uri=http://SERVER:3000/signin")

ACCESS_TOKEN=$(echo $TOKEN_RESP | jq -r '.access_token')
```

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
    {"resourceType": " "Organization"},
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
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "id=SECURITY_REQUEST_ID&secret=SECRET&password=NewPassword123!"
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

MedPlum rate-limits auth endpoints (default: 160 requests/minute/IP). If you're testing login flows repeatedly, you may hit the rate limit and get confusing errors. Wait a minute and try again.

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

## Server Configuration (medplum.config.json)

Key settings for self-hosted:

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
| client_credentials returns "Invalid client" | Missing ProjectMembership for ClientApplication | Create a ProjectMembership linking ClientApplication to project via FHIR API |
| "/auth/profile" returns "Profile not found" | Passed Practitioner ID instead of ProjectMembership ID | Use the ProjectMembership resource ID, not the Practitioner ID |

## Fresh Stack Reset

If you've corrupted the database or want a clean start:

```bash
cd /path/to/medplum
docker compose down -v  # Removes volumes (DESTRUCTIVE — deletes all data)
docker compose up -d    # Fresh start
# Wait for healthcheck, then visit http://localhost:3000/ to register
```

⚠️ This deletes ALL data — patients, encounters, users, everything. Only use for development/testing.
