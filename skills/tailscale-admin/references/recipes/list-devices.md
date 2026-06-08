# List devices (with filters)

## When to use

- The user asks "what's on my tailnet?" / "list devices" / "show me my Tailscale nodes".
- The user wants to find stale devices (for cleanup) or filter by tag/hostname.
- Pre-step for `device-actions` recipe — resolve a device by name to its ID.

## Inputs you need

- None required. Optional filters (tag, hostname pattern, staleness threshold).

## API call

```bash
curl -sS "https://api.tailscale.com/api/v2/tailnet/-/devices?fields=all" \
  -H "Authorization: Bearer $TAILSCALE_API_TOKEN"
```

`fields=all` returns full device records. Without it, the response is a leaner subset.

### Device record fields (v1-relevant)

| Field | Type | Meaning |
|---|---|---|
| `id` | string | Public device identifier (use in API paths) |
| `name` | string | FQDN-style name (e.g. `dokploy-sidecar-3.tail-something.ts.net`) |
| `hostname` | string | Short hostname |
| `addresses` | array<string> | Tailscale IPs (100.x.x.x and IPv6) |
| `tags` | array<string> | Currently-applied tags |
| `lastSeen` | string | ISO timestamp of last contact |
| `expires` | string | ISO timestamp when key expires |
| `authorized` | bool | Whether device is approved on the tailnet |
| `os` | string | `linux`, `macos`, `windows`, `ios`, `android`, etc. |
| `clientVersion` | string | Tailscale client version |
| `user` | string | Owner email |

## Filter helpers

### Table view (all devices)

```bash
curl -sS "https://api.tailscale.com/api/v2/tailnet/-/devices?fields=all" \
  -H "Authorization: Bearer $TAILSCALE_API_TOKEN" \
  | jq -r '.devices[] | [.id, .hostname, (.tags // [] | join(",")), .lastSeen, .os] | @tsv'
```

### Stale devices (last seen > 30 days ago)

```bash
CUTOFF=$(date -u -v-30d +%s 2>/dev/null || date -u -d '30 days ago' +%s)
curl -sS "https://api.tailscale.com/api/v2/tailnet/-/devices?fields=all" \
  -H "Authorization: Bearer $TAILSCALE_API_TOKEN" \
  | jq --argjson cutoff "$CUTOFF" \
      '.devices[] | select((.lastSeen | fromdateiso8601) < $cutoff) | {hostname, lastSeen, tags, id}'
```

Note the `date` syntax differs between macOS (`-v-30d`) and Linux (`-d '30 days ago'`); the recipe shows both.

### By tag

```bash
curl -sS "https://api.tailscale.com/api/v2/tailnet/-/devices?fields=all" \
  -H "Authorization: Bearer $TAILSCALE_API_TOKEN" \
  | jq '.devices[] | select(.tags // [] | index("tag:sidecar"))'
```

### By hostname pattern

```bash
curl -sS "https://api.tailscale.com/api/v2/tailnet/-/devices?fields=all" \
  -H "Authorization: Bearer $TAILSCALE_API_TOKEN" \
  | jq '.devices[] | select(.hostname | test("dokploy-sidecar"))'
```

### Untagged devices (no tags applied)

```bash
curl -sS "https://api.tailscale.com/api/v2/tailnet/-/devices?fields=all" \
  -H "Authorization: Bearer $TAILSCALE_API_TOKEN" \
  | jq '.devices[] | select((.tags // []) | length == 0) | {hostname, id, lastSeen}'
```

## Resolving identifiers

Devices have a public `id` (use in API paths) and a `name`/`hostname` (use for human reference). When the user says "delete `dokploy-sidecar-3`", list devices, find by hostname, capture the `id`, and show the resolved choice before any write in another recipe.

## Common errors

| Status | Meaning | Fix |
|---|---|---|
| 401 | Token bad/expired | Stop. Ask user to regenerate. |
| 403 | Token valid, no device read scope | Stop. Tell user. |
| 429 | Rate limit | Stop. Wait 60s before retry. |

## Follow-ups

- "Delete this one" → [`device-actions.md`](./device-actions.md) (delete or expire).
- "Retag this one" → [`device-actions.md`](./device-actions.md) (retag).
- "Audit the whole tailnet" → [`audit-tailnet.md`](./audit-tailnet.md) (correlates with keys + ACL).

## See also

- [`../entity-model.md`](../entity-model.md) — device id vs nodeId
- [`device-actions.md`](./device-actions.md) — actions on a single device
- [`audit-tailnet.md`](./audit-tailnet.md) — composite audit using this list
