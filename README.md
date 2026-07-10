# medplumskills 🏥

Self-hosted Medplum knowledge for AI agents — the things stock LLMs get wrong about deploying, configuring, and debugging Medplum FHIR infrastructure.

Built from real-world experience deploying Medplum on a Raspberry Pi. Every pitfall documented here was learned by breaking things.

## What's Here

Each skill is a standalone markdown file. Give any URL to your AI agent — it reads it and instantly corrects its Medplum knowledge.

**https://github.com/snarflakes/medplumskills/blob/main/SKILL.md** ← table of contents

| Skill | What LLMs Get Wrong |
|-------|---------------------|
| [Deploy](deploy/SKILL.md) | `command: ["env"]` is intentional, `MEDPLUM_BASE_URL` is build-time not runtime, demo recaptcha keys don't work |
| [Auth](auth/SKILL.md) | Project-scoped users get "User not found" without `projectId`, AccessPolicy circular lockouts, no email in self-hosted |
| [Pitfalls](pitfalls/SKILL.md) | Direct DB edits corrupt indexes, Redis FLUSHALL destroys auth, rate limiting looks like "Invalid password" |
| [Clinical](clinical/SKILL.md) | SOAP notes are transaction Bundles not single resources, DoseSpot is paid, `urn:uuid:` for Bundle references |
| [Troubleshooting](troubleshooting/SKILL.md) | Symptom → Cause → Fix table for every error we've hit |

## Use as an OpenClaw Skill

```bash
openclaw skill add snarflakes/medplumskills
```

Or reference `SKILL.md` directly in your agent's workspace.

## Use as a Claude / Cursor Skill

Add to your project's `CLAUDE.md` or `AGENTS.md`:
```
SKILL.md
```

Point to the root or any sub-skill:
- `https://github.com/snarflakes/medplumskills/blob/main/SKILL.md`
- `https://github.com/snarflakes/medplumskills/blob/main/pitfalls/SKILL.md`

## Key Takeaways

1. **Never modify FHIR tables directly in PostgreSQL** — breaks indexing silently
2. **Never `FLUSHALL` Redis in production** — destroys JWT signing keys, breaks auth
3. **Project-scoped users need `projectId` at login** — #1 cause of "User not found"
4. **`MEDPLUM_BASE_URL` is baked into the Provider app build** — rebuild when URL changes
5. **Self-hosted has no email server** — use `/auth/setpassword` for passwords
6. **AccessPolicy circular lockout is real** — if your policy restricts AccessPolicy writes, you can't update it
7. **Auth rate limiting looks like "Invalid password"** — wait 60 seconds between testing bursts
8. **A SOAP note is a transaction Bundle, not a single resource** — 6-8 FHIR resources linked together

## License

MIT — see [LICENSE](LICENSE)

---

*Built by [snarflakes](https://github.com/snarflakes) and [🧞‍♂️ snarling genie](https://etherscan.io/tx/0x15795ccef7dcde1e88a040c238af782b6c451cc8fc24575a0065d55e73dfaea3) (ERC-8004 Agent #34020)*