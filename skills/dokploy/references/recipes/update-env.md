# Update env vars (safe splice)

## ⚠️ Critical footgun

Dokploy stores environment variables as **a single raw `.env`-format string**, not a key-value map. The `saveEnvironment` endpoint replaces the entire string. **Sending only the new var will clobber every other env var.**

Always: **read → splice → write**. Never write a partial env block.

## When to use

- The user wants to add, change, or remove a single env var on an existing app.
- The user wants to bulk-replace the env block (rare — only when they've staged a full file).

## Inputs you need

- `applicationId` (required).
- The env vars to set, add, or delete (key + value pairs from the user).

## Step 1 — Read the current env block

```bash
curl -sS "$DOKPLOY_URL/api/application.one?applicationId=<id>" \
  -H "x-api-key: $DOKPLOY_API_TOKEN" \
  | jq -r '.env // ""'
```

Output is the raw `.env` file as text, e.g.:

```
NODE_ENV=production
DATABASE_URL=postgres://...
LOG_LEVEL=info
```

If `.env` is `null`, treat it as the empty string.

## Step 2 — Splice the user's changes locally

Parse the existing block into `KEY=VALUE` lines. For each user change:

- **Add new key** → append a new line.
- **Update existing key** → replace the matching line (match on `^KEY=`).
- **Delete key** → drop the matching line.

Preserve every other line untouched, including blank lines and `# comments`.

Show the user the diff before writing:

```
--- current
+++ proposed
- LOG_LEVEL=info
+ LOG_LEVEL=debug
+ FEATURE_X_ENABLED=true
```

This is transparency, not a confirmation gate — the host runtime decides whether to prompt before the POST.

## Step 3 — Write the full block back

```bash
curl -sS -X POST "$DOKPLOY_URL/api/application.saveEnvironment" \
  -H "x-api-key: $DOKPLOY_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d "$(jq -n \
    --arg id '<id>' \
    --arg env '<full env file as string, including newlines>' \
    '{
      applicationId: $id,
      env: $env,
      buildArgs: null,
      buildSecrets: null,
      createEnvFile: true
    }')"
```

**All five fields are required by the API** — `applicationId`, `env`, `buildArgs`, `buildSecrets`, `createEnvFile`. Pass `null` for `buildArgs`/`buildSecrets` if you're not changing them. `createEnvFile: true` writes the env to a file in the container at runtime (Dokploy's default behaviour).

Use `jq -n --arg` to safely encode multi-line strings into JSON — do not hand-escape newlines.

Response on success: `200` with `{}` body.

## Step 4 — Redeploy to apply

`saveEnvironment` updates the stored block but does not restart the app. The new env takes effect on the **next** deploy. Run [`redeploy.md`](./redeploy.md) immediately after.

## Resolving identifiers

If the user gave an app name, walk per [`../entity-model.md`](../entity-model.md). Show the resolved app before firing the read OR the write.

## Common errors

| Status | Meaning | Fix |
|---|---|---|
| 400 | Missing one of the 5 required fields | Recheck body shape. All 5 must be present. |
| 401 / 403 | Token wrong or no edit permission | Stop. Tell user. |
| 404 | `applicationId` wrong | Re-resolve. |
| 500 | Server-side | Show body, don't retry. |

## Follow-ups

- After write → redeploy ([`redeploy.md`](./redeploy.md)) so the new env takes effect.
- "Did the app come up?" → [`deployment-status.md`](./deployment-status.md).

## See also

- [`../entity-model.md`](../entity-model.md) for name → ID resolution
- [`redeploy.md`](./redeploy.md) — required after every env change
- [`../api-overview.md`](../api-overview.md) for the error envelope
