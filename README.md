# medplumskills 🏥

Self-hosted Medplum knowledge for AI agents — the things stock LLMs get wrong about deploying, configuring, and debugging Medplum FHIR infrastructure.

Every pitfall documented here was learned by breaking things on a real deployment.

## Quick Links

- **[SKILL.md](SKILL.md)** — Complete guide (deploy, auth, pitfalls, clinical, troubleshooting)
- **[CONTRIBUTING.md](CONTRIBUTING.md)** — How to add content

## Key Takeaways

1. **Never modify FHIR tables directly in PostgreSQL** — breaks indexing silently
2. **Never `FLUSHALL` Redis in production** — destroys JWT signing keys, breaks auth
3. **Project-scoped users need `projectId` at login** — #1 cause of "User not found"; the Provider app's `MedplumClient.startLogin()` doesn't include `projectId` from constructor config — you must patch the JS bundle to add `projectId:e.projectId??this.options.projectId` as a fallback
4. **`MEDPLUM_BASE_URL` is baked into the Provider app build** — rebuild when URL changes
5. **Self-hosted has no email server** — use `/auth/setpassword` for passwords
6. **AccessPolicy circular lockout is real** — if your policy restricts AccessPolicy writes, you can't update it
7. **Auth rate limiting looks like "Invalid password"** — wait 60 seconds between testing bursts
8. **A SOAP note is a transaction Bundle, not a single resource** — 6-8 FHIR resources linked together
9. **`client_credentials` needs a ProjectMembership** — a ClientApplication alone returns "Invalid client"; you must also create a ProjectMembership linking it to a project
10. **`/auth/profile` takes a ProjectMembership ID, not a Practitioner ID** — if only one membership exists, it's auto-selected (skip this step)
11. **`client_id` is the ClientApplication resource ID** — not a separate field
12. **Provider app `onUnauthenticated` strips `?project=` URL params** — it redirects to `/`, losing all query parameters; a URL redirect script alone doesn't fix this — you must patch `startLogin` in the JS bundle
13. **Self-hosted Provider app needs nginx `/oauth2/` proxy** — otherwise PKCE token exchange returns 405
14. **Self-hosted Provider app needs `MEDPLUM_CLIENT_ID=` (empty)** — with it set, the app uses OAuth2 authorization-code flow which requires a matching `redirectUri` on the ClientApplication

## Auth for Agentic Workflows

Agents need two auth paths to interact with Medplum:

### 1. PKCE Flow (browser-based / human-in-the-loop)
For interactive sessions where a human logs in. The Provider app uses this automatically. Agents can use it too — generate a `code_verifier`/`code_challenge` pair, call `/auth/login` with `projectId`, then exchange the code at `/oauth2/token` for a bearer token. Tokens expire in 1 hour.

### 2. `client_credentials` Flow (machine-to-machine / autonomous agents)
For headless API access — no browser, no human. Requires **three** resources, not just a ClientApplication:

1. **ClientApplication** — `grantType: ["client_credentials"]`, `pkceOptional: true`, set a `secret`
2. **ProjectMembership** — links the ClientApplication to your project (`admin: true` for full access)
3. **Token exchange** — `POST /oauth2/token` with `grant_type=client_credentials`, `client_id=<ClientApplication-ID>`, `client_secret=<secret>`

⚠️ Without the ProjectMembership, you get "Invalid client" — and this is **not documented** in Medplum's client-credentials guide.

## Use as an OpenClaw Skill

```bash
openclaw skill add snarflakes/medplumskills
```

## Use as a Claude / Cursor Skill

Point to the root SKILL.md:
- `https://github.com/snarflakes/medplumskills/blob/main/SKILL.md`

## License

MIT — see [LICENSE](LICENSE)

---

*Built by [snarflakes](https://github.com/snarflakes) and [🧞‍♂️ snarling genie](https://etherscan.io/tx/0x15795ccef7dcde1e88a040c238af782b6c451cc8fc24575a0065d55e73dfaea3) (ERC-8004 Agent #34020)*