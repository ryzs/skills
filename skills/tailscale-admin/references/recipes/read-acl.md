# Read current ACL (read-only)

## When to use

- The user asks "what does my ACL allow?" / "show me the ACL".
- The user wants to audit which tags are defined and who can apply them.
- Before any tag operation in another recipe — verify the tag exists in `tagOwners`.

This recipe is **read-only**. ACL editing is deferred to v2 (needs `If-Match` ETag handling to avoid race conditions and lockout risk).

## Inputs you need

- None beyond `TAILSCALE_API_TOKEN`. The tailnet is auto-resolved via `-`.

## API call

Two response modes, controlled by the `Accept` header. **Do not add `?details=1`** — that wraps the policy as `{acl, errors, warnings}` where `acl` is an unparseable HuJSON *string*, which breaks every jq path below.

**Raw HuJSON** (preserves comments and structure — present this to the user first):

```bash
curl -sS "https://api.tailscale.com/api/v2/tailnet/-/acl" \
  -H "Authorization: Bearer $TAILSCALE_API_TOKEN" \
  -H "Accept: application/hujson"
```

This returns HuJSON text (comments, trailing commas). Readable, but **not** valid JSON — do not pipe it to `jq`.

**Parsed JSON** (HuJSON normalized to a real JSON object — use this for any jq query):

```bash
curl -sS "https://api.tailscale.com/api/v2/tailnet/-/acl" \
  -H "Authorization: Bearer $TAILSCALE_API_TOKEN" \
  -H "Accept: application/json" \
  | jq
```

The JSON object exposes `tagOwners`, `acls`, `grants`, `hosts`, `nodeAttrs`, `ssh`, `autoApprovers` at the **top level** — query them directly (e.g. `.tagOwners`). There is no `acl` wrapper key in this mode.

## What to show the user

1. **Raw ACL** (HuJSON; preserves comments). Always present this first.
2. **Tag summary** — extract `tagOwners` and list each declared tag with its owners.
3. **Rule summary** — group ACL rules by what they grant (e.g. "tag:dokploy can SSH to tag:sidecar").

Example tag summary jq (against the parsed-JSON output):

```bash
curl -sS "https://api.tailscale.com/api/v2/tailnet/-/acl" \
  -H "Authorization: Bearer $TAILSCALE_API_TOKEN" \
  -H "Accept: application/json" \
  | jq '.tagOwners | to_entries | map({tag: .key, owners: .value})'
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
