# Trigger redeploy

## When to use

- The user asks to "redeploy", "rebuild", or "restart" an app.
- A deployment failed and the user wants to retry the same source.
- After updating env vars or pulling a fresh commit on the configured branch.

## Inputs you need

- `applicationId` (required).
- Optional `title` (short label, defaults to "manual redeploy").
- Optional `description` (free text — why this redeploy was triggered).

## API call

```bash
curl -sS -X POST "$DOKPLOY_URL/api/application.redeploy" \
  -H "x-api-key: $DOKPLOY_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "applicationId": "<id>",
    "title": "manual redeploy",
    "description": "<reason, optional>"
  }'
```

Response on success: `200` with `{}` body. The redeploy is enqueued — use the [`deployment-status`](./deployment-status.md) recipe to poll for completion.

## Resolving identifiers

If the user gave an app name, walk Project → Environment → Application per [`../entity-model.md`](../entity-model.md). **Show the resolved app to the user before firing the POST.**

Resolved-choice line shape:
```
Resolved: project="<name>" env="<name>" app="<name>" id="<id>"
About to POST /api/application.redeploy
```

This is transparency, not a confirmation gate — the host runtime decides whether to prompt before the write.

## Common errors

| Status | Meaning | Fix |
|---|---|---|
| 400 | `applicationId` missing or wrong shape | Recheck the body. ID must be a string. |
| 401 | Token bad/expired | Stop. Ask user to regenerate. |
| 403 | No deploy permission on this app | Stop. Tell user which app. |
| 404 | App not found | Re-resolve via `project.all`. |
| 500 | Server-side (e.g. build queue down) | Show response body. Do not auto-retry. |

## Follow-ups

- "Did it work?" → run [`deployment-status`](./deployment-status.md) with the same `applicationId`.
- "Why did it fail?" → check `errorMessage` in the deployment record. If insufficient, SSH and `docker logs`.

## See also

- [`../entity-model.md`](../entity-model.md) for name → ID resolution
- [`deployment-status.md`](./deployment-status.md) for polling after redeploy
- [`update-env.md`](./update-env.md) if redeploying because env changed
