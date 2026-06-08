# Read current ACL (read-only)

## When to use

- The user asks "what does my ACL allow?" / "show me the ACL".
- The user wants to audit which tags are defined and who can apply them.
- Before any tag operation in another recipe — verify the tag exists in `tagOwners`.

This recipe is **read-only**. ACL editing is deferred to v2 (needs `If-Match` ETag handling to avoid race conditions and lockout risk).

## Inputs you need

- None beyond `TAILSCALE_API_TOKEN`. The tailnet is auto-resolved via `-`.

## API call

```bash
curl -sS "https://api.tailscale.com/api/v2/tailnet/-/acl?details=1" \
  -H "Authorization: Bearer $TAILSCALE_API_TOKEN" \
  -H "Accept: application/hujson"
```

`details=1` includes edit metadata (`tailnet`, `acl`, edited-by user, edit timestamp).

For JSON-normalized output (loses HuJSON comments and structure but is easier to query with jq):

```bash
curl -sS "https://api.tailscale.com/api/v2/tailnet/-/acl?details=1" \
  -H "Authorization: Bearer $TAILSCALE_API_TOKEN" \
  -H "Accept: application/json" \
  | jq
```

## What to show the user

1. **Raw ACL** (default HuJSON; preserves comments). Always present this first.
2. **Tag summary** — extract `tagOwners` and list each declared tag with its owners.
3. **Rule summary** — group ACL rules by what they grant (e.g. "tag:dokploy can SSH to tag:sidecar").
4. **Edit metadata** — show who last edited and when (from the `?details=1` response).

Example tag summary jq (against JSON output):

```bash
... | jq '.tagOwners | to_entries | map({tag: .key, owners: .value})'
```

## Resolving identifiers

Not applicable — there is only one ACL per tailnet.

## Common errors

| Status | Meaning | Fix |
|---|---|---|
| 401 | Token bad/expired | Stop. Ask user to regenerate. |
| 403 | Token valid, no ACL read permission | Stop. Tell user which token was rejected. |
| 500 | Server-side | Show body. Do not retry. |

## Follow-ups

- "Edit the ACL" → not supported in v1. Tell the user to edit in the web UI or paste the current ACL into a local editor, modify, paste back to admin UI.
- "Why is my tag rejected?" → check if it's in `tagOwners`. If not, that's the fix.
- "Who can apply `tag:dokploy`?" → look at `tagOwners["tag:dokploy"]`.

## See also

- [`../entity-model.md`](../entity-model.md) — the `tagOwners` declaration rule
- [`../api-overview.md`](../api-overview.md) — error envelope shape
- [`device-actions.md`](./device-actions.md) — retag operations that depend on ACL declarations
