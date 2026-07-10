---
name: medplum-auth
description: "Medplum auth model, AccessPolicy, user management. Use when configuring users, roles, permissions, or troubleshooting login failures."
---

# Auth — Users, Projects, Memberships, AccessPolicy

Stock LLMs will tell you to just create a user and set a password. They don't know about project-scoped login requirements, AccessPolicy circular lockouts, or that the invite flow silently drops emails in self-hosted.

## Auth Model

```
User (server-scoped or project-scoped)
  └── ProjectMembership (per-project access)
        ├── profile: Practitioner | Patient | RelatedPerson
        ├── accessPolicy: AccessPolicy (what they can do)
        ├── access: [parameterized policies]
        └── admin: boolean
```

- **User**: Authentication identity (email + password). Global, can belong to multiple projects.
- **Project**: Isolated FHIR data container. Each org typically has one.
- **ProjectMembership**: Links User → Project with a profile and access policy.
- **Profile**: A FHIR resource (Practitioner, Patient, RelatedPerson) representing the user in the project.
- **AccessPolicy**: Defines what resource types and operations the user can access.

## Server-scoped vs Project-scoped

| | Server-scoped | Project-scoped |
|---|---|---|
| **Use for** | Admins, developers | Clinicians, patients |
| **Can access** | Multiple projects | Single project only |
| **Login** | Works everywhere | Requires `projectId` parameter |
| **Default for** | First user (super admin) | Invited practitioners |
| **Set via** | `scope: 'server'` | `scope: 'project'` (default for invites) |

🔴 **Project-scoped users MUST include `projectId` in the login request.** Without it, login returns "User not found." This is the #1 login failure for invited practitioners.

```bash
# Project-scoped login (practitioners invited via Admin UI)
curl -X POST /auth/login -d "email=dr@example.com&password=***&projectId=PROJECT_ID"

# Server-scoped login (admin, first user)
curl -X POST /auth/login -d "email=admin@example.com&password=***"
```

The Provider app handles `projectId` automatically if the user's ProjectMembership is correctly configured. If login fails with "User not found", check the user's scope.

## Auth Flow (OAuth2 / PKCE)

```
1. POST /auth/login     → {code, login}
2. POST /auth/profile   → {code, ...}  (choose profile/project)
3. POST /oauth2/token   → {access_token, refresh_token}
```

Project-scoped users: step 1 requires `projectId`.
Server-scoped users: step 1 works without `projectId`.

## AccessPolicy

AccessPolicy controls which FHIR resource types a user can read/write. Every ProjectMembership must reference one.

### Creating via FHIR API

```bash
curl -X POST http://SERVER:8103/fhir/R4/AccessPolicy \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "resourceType": "AccessPolicy",
    "name": "Clinical Access",
    "resource": [
      {"resourceType": "Patient"},
      {"resourceType": "Encounter"},
      {"resourceType": "Observation"},
      {"resourceType": "Condition"}
    ]
  }'
```

🔴 **Always create AccessPolicy via FHIR API, never via direct DB insert.** Direct DB inserts bypass indexing and cause "Profile not found" errors.

### AccessPolicy features

- **Block access**: Omit a resource type → user cannot access it
- **Read-only**: `"readonly": true` on a resource entry
- **Criteria-based**: `"criteria": "Patient?address-state=CA"`
- **Compartment-based**: `"criteria": "Observation?_compartment=Patient/xyz"`
- **Parameterized**: Use `%profile` or `%variable_name` variables, set on ProjectMembership.access
- **Field-level**: `readonlyFields`, `hiddenFields` arrays
- **Interaction-level**: `"interaction": ["read", "search"]`

### AccessPolicy circular dependency

🔴 **If your AccessPolicy restricts writes to `AccessPolicy`, `Project`, or `User` resource types, you can't update it via the FHIR API (403 Forbidden).** This locks you out of modifying your own policy.

**Fix**: Use the Super Admin account (which bypasses AccessPolicy checks) or update via direct DB (last resort, then rebuild indexes).

### Recommended policies

**Practitioner (56 clinical types)**: Everything needed for day-to-day clinical work, excluding admin infrastructure (AccessPolicy, Project, User, Secret, ClientApplication).

**Admin (64 types)**: Full access including admin infrastructure resources.

**Patient (compartment-scoped)**: Use criteria-based policies scoped to `Patient/{{%patient.id}}`. ⚠️ **HIPAA risk**: misconfiguration could expose other patients' data. Test thoroughly.

## Inviting Users

### Via Admin UI

1. Go to `/admin/invite` (requires project admin or super admin)
2. Choose profile type: Practitioner, Patient, or RelatedPerson
3. Set name, email, AccessPolicy, admin flag
4. Submit → creates User + Profile + ProjectMembership

### Via API

```bash
curl -X POST "http://SERVER/admin/projects/PROJECT_ID/invite" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "resourceType": "Practitioner",
    "firstName": "Jane",
    "lastName": "Smith",
    "email": "jane@example.com",
    "sendEmail": false,
    "membership": {
      "admin": false,
      "accessPolicy": {"reference": "AccessPolicy/POLICY_ID"},
      "access": []
    }
  }'
```

### Setting passwords (no email server)

See [deploy/SKILL.md](../deploy/SKILL.md) for the three methods: Super Admin UI, API, or browser URL.

## Rate Limiting

🔴 **Auth endpoints are rate-limited at 160 requests/minute/IP.** If you're testing login flows repeatedly, you'll hit this and get misleading "Invalid password" errors. Wait 60 seconds and try again.