# Manage auth keys (list, create, delete)

## ⚠️ Critical footgun

The secret key value (`tskey-auth-...`) is in the **create response body, returned ONCE**. After creation, only metadata is retrievable — never the secret. If you lose it, the only fix is delete + recreate.

**Always capture the create response body in full and tell the user to store the secret immediately.**

## When to use

- The user wants to provision a new device (ephemeral container, new laptop, sidecar).
- The user wants to audit existing auth keys for security review.
- The user wants to revoke an old or compromised key.

## Inputs you need depend on operation

- **List:** none.
- **Create:** capabilities (reusable, ephemeral, preauthorized, tags), expiry (max 90 days), description.
- **Delete:** `keyId` from the list operation.

---

## Operation: List

```bash
curl -sS "https://api.tailscale.com/api/v2/tailnet/-/keys" \
  -H "Authorization: Bearer $TAILSCALE_API_TOKEN" \
  | jq '.keys[] | {id, description, expires, revoked}'
```

Returns metadata only — no secrets. Each key has:

| Field | Meaning |
|---|---|
| `id` | Use this for delete |
| `description` | Free-text, set at creation |
| `expires` | ISO timestamp (max 90 days from creation) |
| `revoked` | Revocation timestamp; `null` (or absent) for active keys, set once revoked. Treat "present and non-null" as revoked rather than expecting a strict boolean. |
| `capabilities` | The configured reusable/ephemeral/preauthorized/tags |

### Filter for unused or expired

```bash
NOW=$(date -u +%s)
curl -sS "https://api.tailscale.com/api/v2/tailnet/-/keys" \
  -H "Authorization: Bearer $TAILSCALE_API_TOKEN" \
  | jq --argjson now "$NOW" \
      '.keys[] | select((.expires | fromdateiso8601) < $now) | {id, description, expires}'
```

---

## Operation: Create

```bash
curl -sS -X POST "https://api.tailscale.com/api/v2/tailnet/-/keys" \
  -H "Authorization: Bearer $TAILSCALE_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "capabilities": {
      "devices": {
        "create": {
          "reusable": false,
          "ephemeral": false,
          "preauthorized": true,
          "tags": ["tag:dokploy"]
        }
      }
    },
    "expirySeconds": 7776000,
    "description": "provisioning key for dokploy nodes"
  }'
```

### Field meanings

| Field | Effect |
|---|---|
| `reusable: true` | Key can mint multiple devices. Risky for security — prefer `false` per device unless you're provisioning a fleet. |
| `ephemeral: true` | Devices created from this key auto-delete from the tailnet when offline. Use for sidecars and short-lived containers. |
| `preauthorized: true` | Device is auto-approved without manual admin click. **Required for headless provisioning.** Without it, the device sits unauthorized until someone approves in the UI. |
| `tags` | Tags assigned to devices created by this key. **Must be in ACL `tagOwners`** (see [`../entity-model.md`](../entity-model.md)). |
| `expirySeconds` | Max 90 days (`7776000`). |
| `description` | Free-text for audit purposes. **Always set this** — keys without descriptions are audit-blind. |

### Response

```json
{
  "id": "kBcDeF12CNTRL",
  "key": "tskey-auth-kBcDeF12CNTRL-AbcDef...",
  "created": "2026-06-04T10:00:00Z",
  "expires": "2026-09-02T10:00:00Z",
  "revoked": "",
  "capabilities": { ... }
}
```

The `key` field is the **secret**. Show it to the user immediately and tell them to store it. After this response, only `id` and metadata are retrievable.

---

## Operation: Delete (revoke)

```bash
curl -sS -X DELETE "https://api.tailscale.com/api/v2/tailnet/-/keys/<keyId>" \
  -H "Authorization: Bearer $TAILSCALE_API_TOKEN" \
  -w "\nHTTP %{http_code}\n"
```

Expected: `HTTP 200` with empty body.

Revoking a key:
- **Does not** affect devices already authenticated with it.
- **Does** prevent new device registrations using the key.

## Resolving identifiers

For delete, the `keyId` comes from the list operation. Show the user the resolved key:

```
Resolved: key id="kBcDeF12CNTRL" description="provisioning key for dokploy nodes" expires=2026-09-02T10:00:00Z
About to DELETE /tailnet/-/keys/kBcDeF12CNTRL
```

## Common errors

| Status | Meaning | Fix |
|---|---|---|
| 400 (create) | Tag not in `tagOwners`, expiry too long (>90d), or body shape wrong | Show body. Check tag against `read-acl` recipe. |
| 401 | Token bad/expired | Stop. |
| 403 | Token lacks key management scope | Stop. Tell user. |
| 404 (delete) | Key id wrong | Re-resolve via list. |

## Follow-ups

- After create → user uses the secret to register a new device (`tailscale up --auth-key=<secret>`).
- After delete → check no devices were depending on the key (list devices, look for upcoming expirations).

## See also

- [`../entity-model.md`](../entity-model.md) — secret-returned-once and tag-must-be-declared rules
- [`read-acl.md`](./read-acl.md) — verify tags exist in `tagOwners` before using them in `create`
- [`list-devices.md`](./list-devices.md) — check which devices depend on a key
