# Tailscale Admin API overview

Shared primer for every recipe in this skill. Load this once when you start working with Tailscale in a session.

## Base URL

```
https://api.tailscale.com/api/v2
```

Recipe paths append the rest, e.g. `https://api.tailscale.com/api/v2/tailnet/-/devices`.

## Authentication

Every request needs:

```
Authorization: Bearer $TAILSCALE_API_TOKEN
```

The token is generated in the Tailscale admin console under **Settings → Keys → API access tokens**. Tokens are long-lived and rotated manually.

## Tailnet identifier

This skill hardcodes the `-` shorthand for the tailnet name in every path:

```
/tailnet/-/devices
/tailnet/-/keys
/tailnet/-/acl
```

`-` resolves to "the tailnet this token belongs to". Users with multi-tailnet access who need explicit naming can override `-` with the literal tailnet name (e.g. `example.com` or `tail-something.ts.net`) in any recipe.

## Endpoint style

REST-ish. Methods used in v1:

| Method | Path | Purpose |
|---|---|---|
| GET | `/tailnet/-/devices?fields=all` | List devices |
| GET | `/tailnet/-/keys` | List auth keys |
| POST | `/tailnet/-/keys` | Create auth key |
| DELETE | `/tailnet/-/keys/{keyId}` | Revoke auth key |
| GET | `/tailnet/-/acl?details=1` | Read ACL with edit metadata |
| DELETE | `/device/{deviceId}` | Delete device |
| POST | `/device/{deviceId}/expire` | Expire device key |
| POST | `/device/{deviceId}/tags` | Replace device tags |

Always send `Content-Type: application/json` on POST writes.

## Error envelope

HTTP status carries the meaning. Response bodies on error:

```json
{"message": "tagOwners block missing tag:foo"}
```

| Status | Meaning | What the skill should do |
|---|---|---|
| 400 | Bad request — body shape wrong, or invalid tag, or expired key | Show the response body. Re-check required fields. |
| 401 | Token missing or invalid | Stop. Ask the user to regenerate. |
| 403 | Token valid, no permission | Stop. Tell the user which operation was rejected. |
| 404 | Resource not found | Re-resolve the device or key ID. |
| 429 | Rate limit hit | Stop and back off. Skill should not auto-retry. |

## Rate limits

Tailscale rate-limits the Admin API at approximately **60 requests per minute per token**. v1 recipes make 1–5 calls per operation (audit makes 3). Well under limit for human-paced use.

If the skill is being used in a loop or batch, surface 429 responses immediately and tell the user to slow down or split into chunks.

## Pagination

**None on v1 endpoints.** Device list, key list, and ACL all return full payloads in one response.

## Discovering endpoints not in this skill

Tailscale does not publish an OpenAPI spec for the Admin API. The authoritative reference is:

```
https://tailscale.com/api
```

If a user asks for something outside the v1 recipes (DNS, exit nodes, webhooks, ACL write), point at that URL and compose the call from the docs. The same auth header applies.

## OAuth client upgrade path

For shared use (CI, multi-user automation), swap the API token for an OAuth client:

1. Create an OAuth client in the admin console with scopes matching your operations.
2. At runtime, exchange `clientId` + `clientSecret` for a short-lived access token via `POST /api/v2/oauth/token`.
3. Use the resulting access token in the `Authorization: Bearer` header — recipes do not change.

Documentation: https://tailscale.com/kb/1215/oauth-clients.

This skill's recipes work with both auth modes because they only assume the `Authorization: Bearer` header.

## Quoting and jq

All recipes use `curl -sS` (silent + show errors) and pipe through `jq` for shaping. Tips:

- Single-quote JSON bodies in `-d` to avoid shell expansion.
- Use `jq -n --arg` for safe multi-line string encoding (relevant for ACL bodies if you ever go beyond v1).
- Always show the raw response body on non-2xx — Tailscale's error messages are usually actionable.
