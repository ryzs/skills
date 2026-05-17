# Check deployment status

## When to use

- The user asks "did my deploy go through?" / "what's the status of the last deploy?"
- The user wants to triage a recent deployment failure (returns `status` + `errorMessage`, **not** log content).

## Inputs you need

- `applicationId` (required). If the user gave a name, resolve via the walk in [`../entity-model.md`](../entity-model.md).
- Optional: how many recent deployments to show (default: 5).

## API call

```bash
curl -sS "$DOKPLOY_URL/api/deployment.all?applicationId=<id>" \
  -H "Authorization: Bearer $DOKPLOY_API_TOKEN" \
  | jq '[.[] | {title, status, errorMessage, startedAt, finishedAt, deploymentId}] | .[:5]'
```

Returns an array of deployment records, newest first. Each record contains:

| Field | Type | Meaning |
|---|---|---|
| `deploymentId` | string | Unique ID |
| `title` | string | Set when the deploy was triggered (e.g. "manual redeploy") |
| `description` | string | Optional reason |
| `status` | `running` \| `done` \| `error` \| `cancelled` | Current state |
| `errorMessage` | string \| null | Set when `status == "error"`. May or may not be informative. |
| `startedAt`, `finishedAt`, `createdAt` | ISO timestamps | |
| `logPath` | string | Filesystem path on the Dokploy host — **not readable via REST**. SSH to read. |

## Resolving identifiers

If the user said "check status of `web`", walk Project → Environment → Application per [`../entity-model.md`](../entity-model.md). Show the resolved app before firing the GET.

## Common errors

| Status | Meaning | Fix |
|---|---|---|
| 401 | Token bad/expired | Stop. Ask user to regenerate in Dokploy UI. |
| 403 | Token valid, no read access to this app | Stop. Tell user which app was rejected. |
| 404 | `applicationId` doesn't exist | Re-resolve via `project.all`. Probably the wrong app. |

## Follow-ups

- "Why did it fail?" → show `errorMessage`. If null or uninformative, tell the user to SSH and `docker logs` (see logs limitation in `SKILL.md`).
- "Tail logs" → not supported by this recipe. Logs are WebSocket-only in Dokploy. Suggest SSH `docker logs <container>`.
- "Redeploy it" → see [`redeploy.md`](./redeploy.md).

## See also

- [`../entity-model.md`](../entity-model.md) for name → ID resolution
- [`../api-overview.md`](../api-overview.md) for the error envelope
- [`redeploy.md`](./redeploy.md) when status is `error` and the user wants to retry
