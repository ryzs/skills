# ryzs/skills

Daily-driver AI skills by [@ryzs](https://github.com/ryzs). Compatible with Claude Code, Cursor, Codex, Copilot, Windsurf, Gemini, Cline, and other agents that read the [vercel-labs/skills](https://github.com/vercel-labs/skills) format. Listed on [skills.sh](https://www.skills.sh).

## Install

```bash
npx skills add ryzs/skills
```

This installs every skill below into your AI host's skill directory.

## Skills

| Skill | What it does |
|---|---|
| [`dokploy`](./skills/dokploy) | Deploy, redeploy, manage env vars, check deployment status, and scaffold Dockerfiles against a self-hosted [Dokploy](https://dokploy.com) PaaS via its REST API. |
| [`tailscale-admin`](./skills/tailscale-admin) | Manage a Tailscale tailnet via the Admin API — devices, auth keys, ACL read, audit. For local-node CLI ops, see [majiayu000's tailscale skill](https://github.com/majiayu000/claude-skill-registry). |

## License

MIT — see [LICENSE](./LICENSE).
