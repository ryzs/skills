# Tailscale entity model

The most common source of "wrong device" or "tag rejected" bugs is misunderstanding the relationships between tailnets, devices, auth keys, and ACL tags. Read this before any create/update.

## Hierarchy

```
Tailnet (≈ organisation)
 ├── Users (people who can log into the admin console)
 ├── Devices (each has id, hostname, addresses, tags, lastSeen, expires, os, authorized)
 ├── Auth Keys (capabilities-based; mint new devices on use)
 │    └── secret value returned ONCE at creation; only metadata thereafter
 ├── ACL (HuJSON; defines tags, rules, groups, ssh, hosts)
 │    └── tagOwners block declares which tags exist and who can apply them
 └── DNS / MagicDNS / Exit-nodes / Subnet routers   (out of v1 scope)
```

## Identifiers

- **Device `id`** — the public identifier used in API paths (e.g. `n8MqXKjP12CNTRL`). Use this everywhere.
- **Device `nodeId`** — internal identifier visible in some responses. **Do not use in API paths.**
- **Auth key `id`** — opaque string returned by `POST /tailnet/-/keys` and listed by `GET /tailnet/-/keys`.
- **Tailnet** — the skill hardcodes `-` for "this token's tailnet".

## Critical constraints (footguns)

### Tags must be declared in `tagOwners` before use

Tags use `tag:foo` format. To apply `tag:foo` to a device or an auth key, `tag:foo` must already be defined in the ACL's `tagOwners` block:

```hujson
{
  "tagOwners": {
    "tag:dokploy":   ["autogroup:admin"],
    "tag:sidecar":   ["autogroup:admin"],
    "tag:laptop":    ["autogroup:admin"]
  },
  // ...
}
```

Applying an undeclared tag returns `400 Bad Request` with `{"message": "tagOwners block missing tag:foo"}`. Recipes that touch tags must either verify the tag is declared (via `read-acl` recipe) or surface the 400 with a clear "fix the ACL first" message.

### Auth key secrets are returned ONCE

The `tskey-auth-...` secret value is in the `POST /tailnet/-/keys` response body. After that, you can only retrieve key metadata (id, capabilities, expiry) — never the secret again.

If the secret is lost, the only fix is `DELETE /tailnet/-/keys/{id}` + create a new one.

### Tag replacement is wholesale

`POST /device/{deviceId}/tags` with body `{"tags": ["tag:foo"]}` **replaces** the device's tags. Sending one tag drops all others. Always read current tags first (via `list-devices` or `device.one`), splice the changes, and POST the full set.

### Ephemeral devices auto-delete

Devices created from auth keys with `ephemeral: true` are deleted from the tailnet when they go offline. Useful for sidecar containers but surprising if you forget — a "lost device" might just be a stopped ephemeral.

## Showing the resolved choice

Before any destructive write, show the user the resolved choice:

```
Resolved: device hostname="dokploy-sidecar-3" id="n8MqXKjP12CNTRL" current_tags=["tag:sidecar"]
About to POST /device/n8MqXKjP12CNTRL/tags with new tags ["tag:dokploy","tag:sidecar"]
```

This is transparency, not a confirmation gate — the host runtime (Claude Code, Cursor, etc.) decides whether to prompt before the write.
