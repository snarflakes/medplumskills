---
name: medplum-troubleshooting
description: "Symptom → Cause → Fix table for self-hosted Medplum. Use when debugging errors, login failures, 403s, or weird behavior."
---

# Troubleshooting — Symptom, Cause, Fix

Every entry in this table was hit during real deployment. No theoretical fixes.

## Auth Errors

| Symptom | Cause | Fix |
|---|---|---|
| "User not found" at login | Project-scoped user without `projectId` | Add `projectId=PROJECT_ID` to login request, or ensure Provider app passes it |
| "Profile not found" after login | Corrupted indexed columns from direct DB edits | Rebuild resources from Super Admin UI, or fresh stack reset |
| "Invalid password, must be at least 8 characters" for valid password | Auth rate limiting (160 req/min/IP) | Wait 60 seconds, retry. Don't script rapid login attempts. |
| "Invalid password" after DB password update | Direct DB bcrypt hash insert bypassed indexing | Use `/auth/setpassword` API instead. Never update passwords in PostgreSQL directly. |
| "Generating temporary signing key" in logs | Redis FLUSHALL destroyed JWT signing key | Restart server. All previous tokens are invalid; re-login required. |
| All users locked out after Redis flush | Signing keys lost, tokens invalidated | Restart server + re-login all users. Don't FLUSHALL in production. |

## Access / Permission Errors

| Symptom | Cause | Fix |
|---|---|---|
| 403 Forbidden on FHIR resource creation | AccessPolicy missing that resourceType | Add the resourceType to the user's AccessPolicy via FHIR API |
| 403 Forbidden updating AccessPolicy | AccessPolicy restricts its own writes | Use Super Admin account (bypasses AccessPolicy), or temporarily grant AccessPolicy write access |
| "Error loading patient summary: Forbidden" in Provider app | AccessPolicy missing Encounter, Condition, or Observation | Add all clinical resource types (see [auth/SKILL.md](../auth/SKILL.md)) |
| DoseSpot "Access Check Failed: Forbidden" | Expected — paid integration | Self-hosted doesn't include DoseSpot. Requires Medplum license. |

## Provider App Errors

| Symptom | Cause | Fix |
|---|---|---|
| Provider app shows blank page | `MEDPLUM_BASE_URL` mismatch | Rebuild Provider app with correct URL (it's a Vite build-time variable) |
| CORS errors when Provider calls API | Different origin than server | Use nginx reverse proxy on same origin, or set `MEDPLUM_ALLOWED_ORIGINS=*` |
| Login works in Admin app, fails in Provider | Project-scoped user needs `projectId` | Ensure Provider app passes project context, or make user server-scoped |
| Provider app auth redirect loops | PKCE flow misconfiguration | Check `MEDPLUM_BASE_URL` includes trailing slash, matches server URL exactly |

## Server Errors

| Symptom | Cause | Fix |
|---|---|---|
| Server crashes on startup | `command: ["env"]` removed from docker-compose | Restore `command: ["env"]` — it triggers the entrypoint that generates config |
| Server crashes on startup | PostgreSQL or Redis not ready | Use `docker compose down && docker compose up -d` for clean restart |
| Recaptcha blocking login | Demo recaptcha keys in self-hosted | Set `RECAPTCHA_SITE_KEY=""` and `MEDPLUM_RECAPTCHA_SECRET_KEY=""` in docker-compose |
| Can't create Secret resource (400 Bad Request) | Secret handling requires specific server config | Check server config, may need `SECRETS_ENCRYPTION_KEY` |
| Invite email not sent | No SMTP configured in self-hosted | Use `/auth/setpassword` API or Super Admin "Force Set Password" |

## Database / Index Errors

| Symptom | Cause | Fix |
|---|---|---|
| Search returns stale data after DB update | Direct DB edit bypassed search indexes | Use FHIR API for all resource creation/updates |
| Resource exists in DB but API returns 404 | Indexed columns not updated | Rebuild resources from Super Admin UI |
| "Profile not found" for valid Practitioner | Direct DB insert didn't update `__sharedTokens` etc. | Use FHIR API. If stuck, Super Admin → Rebuild Resources + Reindex |

## Fresh Stack Reset

When all else fails:

```bash
cd /path/to/medplum
docker compose down -v  # ⚠️ DESTRUCTIVE — deletes ALL data
docker compose up -d    # Fresh start
# Visit http://localhost:3000/ to register
```

This wipes everything — patients, encounters, users, policies. Development/testing only.