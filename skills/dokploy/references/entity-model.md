# Dokploy entity model

The most common source of "wrong app" bugs is grabbing an `applicationId` from the wrong project or environment. Read this before any create/update operation.

## Hierarchy

```
Organization
 └── Project           (user-named, e.g. "my-saas")
     └── Environment   (default: "production")
         └── Application
             └── Deployment   (immutable build record with logPath, status)
```

- **Project** — a logical group of services. User-named, unique per organisation.
- **Environment** — a deployment target inside a project. Every project has at least `production`. Multi-env (staging, preview) is opt-in.
- **Application** — the deployable. Owns source (Git repo), build config, env block, and the live container.
- **Deployment** — an immutable record of one build/deploy attempt. Stores `status`, `errorMessage`, timestamps, and a `logPath` (filesystem path on the Dokploy host, not readable via REST).

Services other than Application — Compose, Postgres, MySQL, Redis, Mongo, MariaDB — sit at the same level as Application inside an Environment. **v1 of this skill only covers Application.**

## Uniqueness rules

- **Project names** are unique within an organisation.
- **Environment names** are unique within a project (e.g. only one `production` per project).
- **Application names** are unique within an environment — **not globally**. Two different projects can each have an app called `web`.

→ **Always resolve identifiers through the full hierarchy.** Never pick an app by name alone.

## Resolving a name to an ID

When the user says "redeploy `web`", do not assume. Walk:

```bash
# 1. List projects
curl -sS "$DOKPLOY_URL/api/project.all" \
  -H "Authorization: Bearer $DOKPLOY_API_TOKEN" \
  | jq '.[] | {projectId, name}'
```

```bash
# 2. Get the project (which embeds environments and their applications)
curl -sS "$DOKPLOY_URL/api/project.one?projectId=<id>" \
  -H "Authorization: Bearer $DOKPLOY_API_TOKEN" \
  | jq '.environments[] | {environmentId, name, applications: [.applications[] | {applicationId, name}]}'
```

```bash
# 3. Pick the app by (project, environment, name) triple
```

If the name is ambiguous (multiple matches), **list the candidates and ask the user which one**. Do not guess.

If the user has already supplied an `applicationId` directly, skip the walk.

## Showing the resolved choice

Before any write, show the user the resolved choice in this shape:

```
Resolved: project="my-saas" env="production" app="web" id="app_abc123"
```

This is transparency, not a confirmation gate — the host runtime (Claude Code, Cursor, etc.) decides whether to prompt before the actual write.
