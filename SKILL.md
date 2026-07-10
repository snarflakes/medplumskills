---
name: medplumskills
description: "Self-hosted Medplum deployment and debugging knowledge for AI agents. Use when deploying, configuring, or troubleshooting a self-hosted Medplum FHIR instance."
---

# MEDPLUMSKILLS — What stock LLMs get wrong about self-hosted Medplum

LLMs will tell you to just `docker compose up` and it works. They don't know about the recaptcha trap, the project-scoped login failure, or that direct DB edits silently corrupt the entire auth system. This skill fixes that.

**Every pitfall here was learned by breaking things.**

---

## Deploy Gotchas

🔴 **Demo recaptcha keys don't work in self-hosted.** The compose file ships demo keys that fail verification. Set both to empty strings to skip recaptcha:
```yaml
RECAPTCHA_SITE_KEY: ""
MEDPLUM_RECAPTCHA_SECRET_KEY: ""
```

🔴 **`command: ["env"]` in docker-compose is intentional.** The server image's entrypoint reads env vars and generates config. Removing it crashes the server trying to read a nonexistent config file.

🔴 **`MEDPLUM_BASE_URL` in the Provider app is baked at build time by Vite.** It's not runtime. If your server URL changes, you must rebuild. Or use nginx reverse proxy on the same origin to avoid CORS entirely.

🔴 **Self-hosted has no email server.** Invite flows silently fail — no welcome email, no password reset. Use `/auth/setpassword` or Super Admin "Force Set Password" to set passwords for invited users.

---

## Auth Gotchas

🔴 **Project-scoped users get "User not found" at login without `projectId`.** This is the #1 login failure. Practitioners invited via Admin UI are project-scoped by default. The Provider app should pass `projectId` automatically, but if login fails, check the user's scope.

```bash
# Project-scoped login (practitioners)
curl -X POST /auth/login -d "email=dr@example.com&password=***&projectId=PROJECT_ID"
# Server-scoped login (admin) — no projectId needed
curl -X POST /auth/login -d "email=admin@example.com&password=***"
```

🔴 **AccessPolicy circular lockout.** If your AccessPolicy restricts writes to `AccessPolicy`, `Project`, or `User`, you can't update it via API (403). Fix: use the Super Admin account (bypasses AccessPolicy), or temporarily grant yourself write access.

🔴 **Auth rate limiting produces "Invalid password" errors.** Medplum rate-limits at 160 req/min/IP. When rate-limited, the error message says "Invalid password, must be at least 8 characters" even for valid passwords. Wait 60 seconds and retry.

---

## The 3 Things That Will Break Your Instance

### 1. Never modify FHIR tables directly in PostgreSQL

Medplum maintains indexed columns (`__version`, `___securitySort`, `___tagSort`, etc.) that must be kept in sync with the `content` JSON column. Direct DB inserts bypass all indexing. Symptoms:
- "Profile not found" for valid users
- "Invalid password" for valid passwords
- Search returning stale data
- Auth completely broken

**Always use the FHIR API.** If you must touch the DB (last resort), run "Rebuild Resources" + "Reindex Resources" from Super Admin, then `redis-cli FLUSHALL`, then restart the server.

### 2. Never FLUSHALL Redis in production

Redis stores JWT signing keys. FLUSHALL destroys them. Symptoms:
- "Generating temporary signing key" in server logs
- All auth tokens invalidated
- Auth fails after every restart

If you must flush (development only), restart the server afterward and re-login all users.

### 3. Docker Compose restart order matters

`docker compose restart` may fail if postgres/redis aren't ready. Use:
```bash
docker compose down && docker compose up -d  # Clean restart with proper ordering
```

---

## AccessPolicy Quick Reference

Create via FHIR API (never via direct DB):

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
      {"resourceType": "Condition"},
      {"resourceType": "MedicationRequest"},
      {"resourceType": "DiagnosticReport"},
      {"resourceType": "Procedure"},
      {"resourceType": "AllergyIntolerance"},
      {"resourceType": "CarePlan"},
      {"resourceType": "DocumentReference"}
    ]
  }'
```

Key features: block access (omit resourceType), read-only (`"readonly": true`), criteria-based filtering, compartment scoping, field-level (`readonlyFields`, `hiddenFields`), interaction-level (`"interaction": ["read", "search"]`).

**Practitioner policy**: 56 clinical resource types, no admin infrastructure.
**Admin policy**: 64 resource types including AccessPolicy, Project, User, Secret.
**Patient policy**: Use compartment-scoped criteria (`Patient/{{%patient.id}}`). ⚠️ HIPAA risk if misconfigured.

---

## SOAP Notes Are Transaction Bundles

A SOAP note is NOT a single FHIR resource. It's a Bundle of 6-8 resources:

| Section | FHIR Resource | Code |
|---|---|---|
| Subjective | Observation | LOINC 10210-3 |
| Objective (vitals) | Observation | LOINC 8310-5, 8480-6, 8462-4 |
| Objective (exam) | Observation | LOINC 10210-3 |
| Assessment | Condition | SNOMED 439401001 |
| Plan | CarePlan + MedicationRequest | SNOMED 736350006 |
| Context | Encounter | — |
| Container | Composition | — |

Use `urn:uuid:` references within transaction Bundles — the server resolves them to actual IDs on create.

Medplum recommends `PlanDefinition + Questionnaire + $apply` for structured visit documentation.

---

## Troubleshooting Table

| Symptom | Cause | Fix |
|---|---|---|
| "User not found" at login | Project-scoped user without `projectId` | Add `projectId` to login request |
| "Profile not found" after login | Corrupted indexed columns from DB edits | Rebuild from Super Admin, or fresh stack reset |
| "Invalid password" for valid password | Auth rate limiting (160 req/min) | Wait 60 seconds |
| "Generating temporary signing key" | Redis flush lost JWT signing key | Restart server, re-login |
| 403 on resource creation | AccessPolicy missing resourceType | Add resourceType to policy via API |
| Provider app blank/won't load | `MEDPLUM_BASE_URL` mismatch | Rebuild Provider app with correct URL |
| CORS errors | App and API on different origins | Use nginx reverse proxy on same origin |
| Recaptcha blocking login | Demo recaptcha keys | Set both recaptcha keys to empty strings |
| DoseSpot "Access Check Failed" | Expected — paid integration | Requires Medplum license |
| Invite email not sent | No SMTP in self-hosted | Use `/auth/setpassword` or Super Admin UI |

---

## Fresh Stack Reset

```bash
cd /path/to/medplum
docker compose down -v  # ⚠️ DESTRUCTIVE — deletes ALL data
docker compose up -d    # Fresh start
```

---

*Built by [snarflakes](https://github.com/snarflakes) and [🧞‍♂️ snarling genie](https://etherscan.io/tx/0x15795ccef7dcde1e88a040c238af782b6c451cc8fc24575a0065d55e73dfaea3) (ERC-8004 Agent #34020)*