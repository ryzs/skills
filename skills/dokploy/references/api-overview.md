# Dokploy API overview

Shared primer for every recipe in this skill. Load this once when you start working with Dokploy in a session.

## Base URL

```
<DOKPLOY_URL>/api
```

Example: `https://dokploy.example.com/api/project.all`.

`DOKPLOY_URL` is the bare origin — no trailing `/api`. The recipes append `/api/...` themselves.

## Authentication

Every request needs:

```
Authorization: Bearer $DOKPLOY_API_TOKEN
```

The token is a static string generated in the Dokploy UI under **Settings → API**. There is no refresh flow — when it expires, the user regenerates it in the UI and updates the env var.

## Endpoint style

tRPC-style RPC over HTTP, exposed as plain JSON-over-REST. Endpoints look like:

```
POST /api/application.redeploy
POST /api/application.saveEnvironment
GET  /api/project.all
GET  /api/application.one?applicationId=<id>
GET  /api/deployment.all?applicationId=<id>
```

Convention: **GET for reads** (query params), **POST for writes** (JSON body). Always send `Content-Type: application/json` on writes.

## Error envelope

HTTP status codes carry the meaning. Response bodies on error are shaped like:

```json
{"code": "BAD_REQUEST", "message": "applicationId is required"}
```

| Status | Meaning | What the skill should do |
|---|---|---|
| 400 | Bad request — body schema wrong | Show the response body. Re-check required fields. |
| 401 | Token missing or invalid | Stop. Ask the user to regenerate the token. |
| 403 | Token valid, no permission for this resource | Stop. Tell the user which resource was rejected. |
| 404 | Resource not found | Re-resolve via `/project.all` (the name probably didn't match). |
| 500 | Server-side error | Show the response body. Do not blindly retry. |

## Pagination

**There is none.** List endpoints (`project.all`, `deployment.all`) return the full array. Dokploy is single-server scale and small enough that this is not a problem in practice.

## Discovering endpoints not in this skill

Every Dokploy server exposes the full OpenAPI spec at:

```
<DOKPLOY_URL>/swagger      # interactive UI
<DOKPLOY_URL>/openapi.json # raw spec, if exposed
```

If a user asks for something not in the recipes, point at `/swagger` and compose the call from the spec. The same auth header applies.

## Quoting and jq

All recipes use `curl -sS` (silent + show-errors) and pipe through `jq` for pretty output. Make sure to:

- Single-quote the `-d` body to avoid shell variable expansion inside JSON.
- Use `--data-raw` if the body contains `@` (Dokploy doesn't currently, but be defensive).
- Always show the raw response body on non-2xx — Dokploy's error messages are usually actionable.
