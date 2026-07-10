# medplumskills 🏥

AI agent skills and operational guides for self-hosting [Medplum](https://www.medplum.com/) — the open-source healthcare development platform.

Built from real-world experience deploying, configuring, and debugging Medplum on a Raspberry Pi. Every pitfall documented here was learned the hard way.

## What's Inside

| File | Description |
|------|-------------|
| [`SKILL.md`](./SKILL.md) | Complete self-hosting guide — architecture, auth, AccessPolicy, Provider app deployment, pitfalls, troubleshooting |

## Quick Links

- **[Full Self-Hosting Guide →](./SKILL.md)** — Start here
- **[Hard-Learned Pitfalls →](./SKILL.md#⚠️-hard-learned-pitfalls)** — Don't skip this section
- **[AccessPolicy Reference →](./SKILL.md#accesspolicy)** — Role-based access control for clinicians, admins, and patients
- **[Troubleshooting Table →](./SKILL.md#troubleshooting)** — Symptom → Cause → Fix

## Key Takeaways

1. **Never modify FHIR tables directly in PostgreSQL** — Medplum maintains indexed columns that break silently when bypassed
2. **Never `FLUSHALL` Redis in production** — It destroys JWT signing keys and breaks auth for all users
3. **Project-scoped users need `projectId` at login** — Without it, they get "User not found"
4. **`MEDPLUM_BASE_URL` is baked into the Provider app build** — Rebuild when the URL changes
5. **Self-hosted has no email service** — Use `/auth/setpassword` or Super Admin UI for passwords

## Use as an OpenClaw Skill

If you're running [OpenClaw](https://github.com/openclaw/openclaw), you can install this as a skill:

```bash
openclaw skill add snarflakes/medplumskills
```

Or reference `SKILL.md` directly in your agent's workspace.

## License

MIT — see [LICENSE](./LICENSE)

---

*Built by [snarflakes](https://github.com/snarflakes) and [🧞‍♂️ snarling genie](https://etherscan.io/tx/0x15795ccef7dcde1e88a040c238af782b6c451cc8fc24575a0065d55e73dfaea3) (ERC-8004 Agent #34020)*