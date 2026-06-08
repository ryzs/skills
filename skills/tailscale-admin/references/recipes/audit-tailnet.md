# Audit tailnet (composite)

Runs three reads, correlates the data, renders a single dashboard. The killer recipe for finding sidecar sprawl, unused tags, expired keys, and stale devices without manually clicking through the admin UI.

## When to use

- The user asks "audit my tailnet" / "find sidecar sprawl" / "what should I clean up?"
- Periodically (monthly?) to keep the tailnet tidy.
- Before a clean-up project to baseline what's there.

## Inputs you need

- Optional: staleness threshold in days (default: 30).

## Process

1. `GET /tailnet/-/devices?fields=all` — full device list.
2. `GET /tailnet/-/keys` — all auth keys.
3. `GET /tailnet/-/acl` with `Accept: application/json` — current ACL as a parsed object. Do **not** add `?details=1` (it returns `acl` as an unparseable HuJSON string).

Three API calls total. Well under rate limit.

## Computation

**Devices:**
- Total count.
- Stale count: `lastSeen` older than the threshold.
- Untagged count: `tags` array empty or missing.
- Group by tag: `{tag: count}` map.

**Auth keys:**
- Total count.
- Expired count: `expires` in the past.
- Reusable count: `capabilities.devices.create.reusable == true`. Flag as security-review-worthy.
- Description-less count: `description` empty. Flag as audit-blind.

**ACL:**
- Tags defined: keys of `tagOwners`.
- Tags in use on devices: union of all `.tags[]` across the device list.
- Tags defined but unused: declared in `tagOwners` but not on any device. Candidates for ACL cleanup.

## Example: composed jq pipeline

```bash
DEVICES=$(curl -sS "https://api.tailscale.com/api/v2/tailnet/-/devices?fields=all" \
  -H "Authorization: Bearer $TAILSCALE_API_TOKEN")

KEYS=$(curl -sS "https://api.tailscale.com/api/v2/tailnet/-/keys" \
  -H "Authorization: Bearer $TAILSCALE_API_TOKEN")

ACL=$(curl -sS "https://api.tailscale.com/api/v2/tailnet/-/acl" \
  -H "Authorization: Bearer $TAILSCALE_API_TOKEN" \
  -H "Accept: application/json")

THRESHOLD=$(date -u -v-30d +%s 2>/dev/null || date -u -d '30 days ago' +%s)
NOW=$(date -u +%s)

# Devices summary
echo "=== Devices ==="
echo "$DEVICES" | jq --argjson cutoff "$THRESHOLD" '
  .devices |
  {
    total: length,
    stale: map(select((.lastSeen | fromdateiso8601) < $cutoff)) | length,
    untagged: map(select((.tags // []) | length == 0)) | length,
    by_tag: [.[] | (.tags // [])[]] | group_by(.) | map({tag: .[0], count: length}) | sort_by(-.count)
  }'

# Keys summary
echo "=== Auth keys ==="
echo "$KEYS" | jq --argjson now "$NOW" '
  .keys |
  {
    total: length,
    expired: map(select((.expires | fromdateiso8601) < $now)) | length,
    reusable: map(select(.capabilities.devices.create.reusable == true)) | length,
    no_description: map(select((.description // "") == "")) | length
  }'

# ACL summary — tags defined vs used
echo "=== ACL tag usage ==="
DEFINED=$(echo "$ACL" | jq '(.tagOwners // {}) | keys')
USED=$(echo "$DEVICES" | jq '[.devices[] | (.tags // [])[]] | unique')
echo "$DEFINED" | jq --argjson used "$USED" '
  . as $defined |
  {
    defined: $defined,
    in_use: $used,
    unused: ($defined - $used)
  }'
```

## What to show the user

Format the output as a single dashboard:

```
=== Tailnet Audit ===
Devices: 27 total
  - 4 stale (lastSeen > 30 days): foo-1, foo-2, bar-3, baz-9
  - 8 untagged
  - By tag: tag:dokploy (5), tag:sidecar (12), tag:laptop (2)

Auth keys: 12 total
  - 3 expired (delete-safe)
  - 5 reusable (security review recommended)
  - 2 without description (audit-blind)

ACL tags:
  - Defined in tagOwners: tag:dokploy, tag:sidecar, tag:laptop, tag:builder, tag:ci, tag:scratch
  - In use on devices: tag:dokploy, tag:sidecar, tag:laptop
  - Defined but unused: tag:builder, tag:ci, tag:scratch (consider removing from ACL)
```

For each "stale device" or "expired key" in the output, also show its id so the user can pipe it into `device-actions` or `auth-keys` delete recipes.

## Common errors

| Status | On which call | Meaning | Fix |
|---|---|---|---|
| 401 | any | Token bad/expired | Stop the whole audit. |
| 403 | one of the three | Token lacks a scope | Tell user which scope is missing. Continue audit with partial data and warn. |
| 429 | any | Rate limit | Wait 60s, retry. |

## Follow-ups

- For each stale device → [`device-actions.md`](./device-actions.md) (delete or expire).
- For each expired key → [`auth-keys.md`](./auth-keys.md) (delete to clean up listings).
- For unused ACL tags → tell user to remove from `tagOwners` in the web UI (ACL write not in v1).
- For reusable keys flagged → tell user to review and decide whether to delete.

## See also

- [`list-devices.md`](./list-devices.md) — the same device endpoint, with single-purpose filters
- [`auth-keys.md`](./auth-keys.md) — for cleaning up keys flagged here
- [`read-acl.md`](./read-acl.md) — for the ACL data used in tag correlation
- [`device-actions.md`](./device-actions.md) — for acting on the stale/untagged devices flagged here
