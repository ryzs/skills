# Device actions (delete, expire, retag)

Three operations on a single device — combined here because the user-intent is shared ("do something to this device"). Each is its own subsection below.

## When to use

- The user wants to remove an old or compromised device from the tailnet.
- The user wants to force a device offline immediately (compromise response).
- The user wants to change the tags on a device (e.g. label a sidecar).

## Pre-step for all three: resolve the device id

If the user gave a hostname, run [`list-devices.md`](./list-devices.md) first and capture the `id`. Show the resolved choice before firing any of the actions below:

```
Resolved: device hostname="dokploy-sidecar-3" id="n8MqXKjP12CNTRL" current_tags=["tag:sidecar"]
About to <action>
```

---

## Action 1: Delete a device

Permanent removal from the tailnet. The device cannot rejoin without a new auth key. Use for genuinely abandoned nodes.

```bash
curl -sS -X DELETE "https://api.tailscale.com/api/v2/device/<deviceId>" \
  -H "Authorization: Bearer $TAILSCALE_API_TOKEN" \
  -w "\nHTTP %{http_code}\n"
```

Expected: `HTTP 200` with empty body.

⚠️ **Permanent.** The device record is gone. If you want to keep history but kick the device off, use **expire** instead.

---

## Action 2: Expire device key

Kicks the device offline immediately. The device record stays in the tailnet; the device needs to re-authenticate (new login or auth key) to reconnect.

```bash
curl -sS -X POST "https://api.tailscale.com/api/v2/device/<deviceId>/expire" \
  -H "Authorization: Bearer $TAILSCALE_API_TOKEN" \
  -w "\nHTTP %{http_code}\n"
```

Expected: `HTTP 200` with empty body.

Use for compromise response or to force a refresh without losing device history.

---

## Action 3: Retag a device

⚠️ **Wholesale replace.** The `tags` array in the request body **replaces** the device's current tags. Sending one tag drops all others.

### Step 3a — Read current tags first

```bash
curl -sS "https://api.tailscale.com/api/v2/tailnet/-/devices?fields=all" \
  -H "Authorization: Bearer $TAILSCALE_API_TOKEN" \
  | jq --arg id '<deviceId>' '.devices[] | select(.id == $id) | .tags'
```

### Step 3b — Compute the new tag set locally

Show the user the diff:

```
--- current
+++ proposed
- tag:sidecar
+ tag:sidecar
+ tag:dokploy
```

### Step 3c — Verify all proposed tags are declared in the ACL

If any proposed tag is not in `tagOwners`, the API returns `400 Bad Request`. Pre-check by running [`read-acl.md`](./read-acl.md) and confirming each tag is declared.

### Step 3d — Write the full tag set

```bash
curl -sS -X POST "https://api.tailscale.com/api/v2/device/<deviceId>/tags" \
  -H "Authorization: Bearer $TAILSCALE_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"tags":["tag:sidecar","tag:dokploy"]}' \
  -w "\nHTTP %{http_code}\n"
```

Expected: `HTTP 200` with empty body.

---

## Common errors (all three actions)

| Status | Operation | Meaning | Fix |
|---|---|---|---|
| 401 | any | Token bad/expired | Stop. |
| 403 | any | Token lacks device management scope | Stop. Tell user which op. |
| 404 | any | `deviceId` wrong | Re-resolve via `list-devices`. |
| 400 | retag | Tag not in `tagOwners` | Run `read-acl`, declare tag first via UI. |
| 500 | any | Server-side | Show body. Do not retry. |

## Follow-ups

- After delete → check if any auth keys depended on this device pattern.
- After expire → if the device is supposed to be alive, the operator needs to re-authenticate it (`tailscale up`).
- After retag → run [`read-acl.md`](./read-acl.md) to verify the new tags grant the intended access.

## See also

- [`list-devices.md`](./list-devices.md) — find the device id from a hostname
- [`read-acl.md`](./read-acl.md) — verify tags before retag, audit grants after
- [`../entity-model.md`](../entity-model.md) — tag-wholesale-replace rule
