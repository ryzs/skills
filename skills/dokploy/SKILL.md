---
name: dokploy
description: Use when the user wants to interact with a Dokploy self-hosted PaaS — deploying apps, updating env vars, redeploying services, checking deployment status, or scaffolding Dokploy-ready Dockerfiles. Calls Dokploy's REST API directly using DOKPLOY_URL and DOKPLOY_API_TOKEN env vars. Performs live writes against the user's Dokploy server.
---

# Dokploy

## Prerequisites

- `DOKPLOY_URL` — base URL of the user's Dokploy server, e.g. `https://dokploy.example.com`
- `DOKPLOY_API_TOKEN` — API key (sent as the `x-api-key` header) from the Dokploy UI → Settings → API
- `curl` and `jq` on PATH

If either env var is missing, **stop and ask the user to set them**. Do not invent or guess values.

## When to use this skill

- The user mentions Dokploy, a Dokploy server, or a Dokploy app/project.
- The user wants to deploy, redeploy, update env vars, or check deployment status on a self-hosted PaaS.
- The user wants to generate a Dockerfile or `.dockerignore` that plays well with Dokploy.

## API over CLI

Dokploy ships an official CLI at [`Dokploy/cli`](https://github.com/Dokploy/cli) but it lags the server. This skill calls the REST API directly via `curl`. **Do not install or invoke the CLI** — use the API recipes below.

## Entity model

Dokploy organises resources as: **Project → Environment → Application**.

Read [`references/entity-model.md`](./references/entity-model.md) before any create or update operation. App names are unique within an environment, not globally — always resolve through the hierarchy.

## Recipes (load on demand)

| Task | Reference |
|---|---|
| Check deployment status | [`references/recipes/deployment-status.md`](./references/recipes/deployment-status.md) |
| Trigger redeploy | [`references/recipes/redeploy.md`](./references/recipes/redeploy.md) |
| Update env vars (safe splice) | [`references/recipes/update-env.md`](./references/recipes/update-env.md) |
| Create app from a Git repo | [`references/recipes/create-app-from-git.md`](./references/recipes/create-app-from-git.md) |
| Scaffold a Dockerfile | [`references/recipes/scaffold-dockerfile.md`](./references/recipes/scaffold-dockerfile.md) |

For anything not covered, consult [`references/api-overview.md`](./references/api-overview.md) and the user's Dokploy server's `/swagger` endpoint.

## Safety

Writes go straight through. The host runtime (Claude Code, Cursor, Codex, etc.) gates network calls — **do not add a second confirmation layer in this skill**. For irreversible operations (delete service, drop database), summarise the call before executing so the user can interrupt.

## Logs (current limitation)

Runtime log tailing is **not** available in this skill — Dokploy serves logs over WebSocket only. Use the [`deployment-status`](./references/recipes/deployment-status.md) recipe for status + `errorMessage` triage. For full log inspection, SSH to the Dokploy host and use `docker logs`.
