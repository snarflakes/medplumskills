---
name: medplum-pitfalls
description: "What breaks in self-hosted Medplum and why. Use when debugging mysterious auth failures, login errors, or data corruption after DB edits."
---

# Pitfalls — 8 Things That Broke Our Instance

Every pitfall in this file was learned by doing the wrong thing and watching Medplum break. We're documenting them so you don't have to repeat our mistakes.

## 1. Never modify FHIR tables directly in PostgreSQL

**What we did**: Inserted an AccessPolicy directly into the `AccessPolicy` PostgreSQL table via `psql`.

**What broke**: "Profile not found" errors for ALL users (including admin), login returning "Invalid password, must be at least 8 characters" for valid passwords, search returning stale data.

**Why**: Medplum maintains indexed columns (`__version`, `___securitySort`, `___tagSort`, `___compartmentIdentifierSort`, `__sharedTokens`, `__sharedTokensText`) that must be kept in sync with the `content` JSON column. Direct DB inserts bypass all of these. The FHIR server can't find resources it can't index.

**Fix**: Always use the FHIR API (`POST /fhir/R4/{ResourceType}`) or Admin API to create/update resources. If you absolutely must touch the DB:
1. Update ALL indexed columns correctly
2. Run "Rebuild Resources" + "Reindex Resources" from Super Admin (`/admin/super`)
3. Flush Redis: `docker exec medplum-redis-1 redis-cli FLUSHALL`
4. Restart the Medplum server

**Better fix**: Don't touch the DB. Use the API.

## 2. Redis FLUSHALL destroys auth

**What we did**: Ran `redis-cli FLUSHALL` to clear caches after DB edits.

**What broke**: All auth tokens invalidated. Server logs showed "Generating temporary signing key" on every restart. JWT signing keys lost. All users unable to log in.

**Why**: Redis stores JWT signing keys (which exist in PostgreSQL but are cached in Redis for performance). After FLUSHALL, the server generates temporary keys that don't survive restarts. Every restart = new temporary key = all previous tokens invalid.

**Fix**: Never FLUSHALL in production. Use `DEL` for specific keys, or let TTL handle expiration. If you must FLUSHALL (development only), restart the server afterward and re-login all users.

## 3. AccessPolicy circular dependency lockout

**What we did**: Created a restrictive AccessPolicy that blocked writes to `AccessPolicy`, `Project`, `User`, and `Secret`.

**What broke**: Couldn't update the AccessPolicy via API (403 Forbidden). The policy prevented its own modification.

**Fix**: Use the Super Admin account (which bypasses AccessPolicy checks) to modify policies. Or temporarily grant yourself AccessPolicy write access, update, then revoke it.

## 4. Project-scoped user login requires projectId

**What happened**: Invited a practitioner via Admin UI. They got "User not found" at login.

**Why**: The invite created a project-scoped user (default for practitioner invites). Project-scoped users must include `projectId` in the login request. The Provider app's login form may not include this by default.

**Fix**: Include `projectId` in the login request, or ensure the Provider app is configured with the project context. For API testing:
```bash
curl -X POST /auth/login -d "email=user@example.com&password=***&projectId=PROJECT_ID"
```

## 5. Signing key warnings mean auth is broken

**Symptom**: Server logs show "Generating temporary signing key" on startup.

**Why**: Redis was flushed or lost, and the signing key stored there is gone. The server creates a temporary one, but it's lost on the next restart.

**Fix**: Restart the server to generate a fresh signing key in PostgreSQL. Then re-login to get new tokens. All previous tokens are invalid.

## 6. Auth rate limiting produces misleading errors

**Symptom**: "Invalid password, must be at least 8 characters" for a password that's clearly 14+ characters.

**Why**: Medplum rate-limits auth endpoints at 160 requests/minute/IP. When rate-limited, the error message is misleading — it doesn't say "rate limited," it says the password is invalid.

**Fix**: Wait 60 seconds between auth testing bursts. Don't script rapid login attempts.

## 7. Docker Compose restart order matters

**Symptom**: Server crashes on startup after `docker compose restart`.

**Why**: If PostgreSQL or Redis aren't ready when the server starts, it fails. Docker Compose `restart` doesn't guarantee ordering.

**Fix**:
```bash
docker compose down && docker compose up -d  # Clean restart with proper ordering
# Or staggered:
docker compose restart postgres redis && sleep 5 && docker compose restart medplum-server
```

## 8. MEDPLUM_BASE_URL is build-time, not runtime

**Symptom**: Provider app shows blank page, login loops, or CORS errors.

**Why**: `MEDPLUM_BASE_URL` is a Vite build-time environment variable. It's baked into the JavaScript bundle during `npm run build`. Changing the `.env` file after building does nothing.

**Fix**: Rebuild the Provider app (`npm run build`) whenever the server URL changes. Or use nginx reverse proxy on the same origin to avoid CORS entirely.

---

## Nuclear Option: Fresh Stack Reset

If you've corrupted the database and nothing else works:

```bash
cd /path/to/medplum
docker compose down -v  # ⚠️ DESTRUCTIVE — deletes ALL data
docker compose up -d    # Fresh start
```

This wipes all patients, encounters, users, policies — everything. Only use for development/testing.