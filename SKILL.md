---
name: medplumskills
description: "Self-hosted Medplum deployment, auth, and clinical FHIR knowledge for AI agents. Use when deploying, configuring, debugging, or building on a self-hosted Medplum instance — FHIR server setup, AccessPolicy, Provider app, user management, Docker, and clinical documentation patterns."
---

# MEDPLUMSKILLS — Self-hosted EHR infrastructure knowledge for AI agents

LLMs think Medplum is just a FHIR server with a nice API. They don't know about AccessPolicy circular dependency lockouts, project-scoped login failures, Redis flush auth destruction, or that direct DB edits silently corrupt the indexing layer. This repo fixes that.

Each skill is a markdown file. Give any URL to your AI agent — it reads it and instantly corrects its Medplum self-hosting knowledge.

https://snarflakes.github.io/medplumskills/SKILL.md ← table of contents  
https://snarflakes.github.io/medplumskills/auth/SKILL.md ← just auth & AccessPolicy  
https://snarflakes.github.io/medplumskills/pitfalls/SKILL.md ← what breaks and why

The agent will look up the specific skills when needed.

---

## Start Here

**Deploying Medplum?** Start with [deploy/SKILL.md](deploy/SKILL.md) — Docker Compose setup, first login, Provider app.

**Configuring access?** Fetch [auth/SKILL.md](auth/SKILL.md) — AccessPolicy, project-scoped users, password setup.

**Something broke?** Fetch [pitfalls/SKILL.md](pitfalls/SKILL.md) — the 8 things we learned the hard way.

---

## Skills

### [Deploy](deploy/SKILL.md) — Start here
Docker Compose, first login, Provider app deployment, nginx config.
- `command: ["env"]` in docker-compose is intentional — the entrypoint generates config from env vars. Removing it crashes the server.
- `MEDPLUM_BASE_URL` in the Provider app is a build-time Vite variable, not runtime. Rebuild when the URL changes.
- Self-hosted has no email server. Invite flows silently fail. Use `/auth/setpassword` or Super Admin "Force Set Password."

### [Auth](auth/SKILL.md)
Users, Projects, Memberships, AccessPolicy, OAuth2/PKCE flow.
- Project-scoped users get "User not found" at login unless `projectId` is included. This is the #1 login failure.
- `FLUSHALL` on Redis destroys JWT signing keys. Auth breaks for all users until server restart + re-login.
- AccessPolicy circular dependency: if your policy restricts `AccessPolicy` writes, you can't update it via API. Only Super Admin can bypass this.

### [Pitfalls](pitfalls/SKILL.md)
8 things that broke our instance, what caused them, and how to fix (or avoid) each one.
- Direct PostgreSQL edits corrupt indexed columns silently. Profile lookups fail. Search returns stale data. Auth breaks. **Always use the FHIR API.**
- Redis `FLUSHALL` in production = all sessions invalidated, signing keys destroyed, auth down.
- Rate limiting on `/auth/login` (160 req/min/IP) produces misleading "Invalid password" errors during testing.

### [Clinical](clinical/SKILL.md)
SOAP notes, FHIR resource mapping, visit templates, PlanDefinition + $apply.
- A SOAP note is NOT a single FHIR resource. It's a transaction Bundle of 6-8 resources (Encounter, Observation, Condition, CarePlan, MedicationRequest, Composition).
- Medplum recommends PlanDefinition + Questionnaire + $apply for structured visit documentation, not raw Bundle creation.

### [Troubleshooting](troubleshooting/SKILL.md)
Symptom → Cause → Fix table for every error we've hit.
- "User not found" = project-scoped user without `projectId`
- "Profile not found" = corrupted indexed columns from DB edits
- 403 on writes = AccessPolicy missing that resourceType
- Provider app blank = `MEDPLUM_BASE_URL` mismatch

---

Skills teach what stock LLMs get wrong about self-hosted Medplum. Content is verified against real deployment experience on Raspberry Pi hardware. If a stock LLM already knows something, we don't include it.

PRs welcome from humans and agents. Read [CONTRIBUTING.md](CONTRIBUTING.md) first — the bar is "would a stock LLM get this wrong?"

MIT