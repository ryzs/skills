# Create app from a Git repo

Multi-step orchestration. Each step is its own API call. **Show each resolved choice before firing the next call** so the user can interrupt if a step picked the wrong target.

## When to use

- The user wants to deploy a new app from a Git repo to Dokploy.
- "Set up `<repo>` on Dokploy under `<project>`."

## Inputs you need

- Project name (existing or new).
- Application name.
- Git provider: `github` / `gitlab` / `gitea` / `bitbucket` / `git` (generic SSH/HTTPS).
- Repo URL or `owner/repo` slug (depending on provider).
- Branch (default: `main`).
- Build type: `dockerfile` (default), `nixpacks`, `heroku_buildpacks`, `paketo_buildpacks`, `static`.

If anything is missing, **ask the user before continuing**. Do not invent.

## Step 1 — Find or create the project

```bash
# List existing projects
curl -sS "$DOKPLOY_URL/api/project.all" \
  -H "Authorization: Bearer $DOKPLOY_API_TOKEN" \
  | jq '.[] | {projectId, name}'
```

If a project with the requested name exists, capture its `projectId`. Otherwise:

```bash
curl -sS -X POST "$DOKPLOY_URL/api/project.create" \
  -H "Authorization: Bearer $DOKPLOY_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"name": "<project name>", "description": "<optional>"}'
```

The response contains the new `projectId` and an initial `production` environment with an `environmentId`.

## Step 2 — Pick the environment

Use the project's `production` environment unless the user specified otherwise:

```bash
curl -sS "$DOKPLOY_URL/api/project.one?projectId=<projectId>" \
  -H "Authorization: Bearer $DOKPLOY_API_TOKEN" \
  | jq '.environments[] | {environmentId, name}'
```

Capture the `environmentId` matching `production` (or the user's chosen env).

## Step 3 — Create the application shell

```bash
curl -sS -X POST "$DOKPLOY_URL/api/application.create" \
  -H "Authorization: Bearer $DOKPLOY_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "<app name>",
    "description": "<optional>",
    "projectId": "<projectId>",
    "environmentId": "<environmentId>"
  }'
```

Response contains the new `applicationId`. Capture it.

## Step 4 — Wire the Git provider

Pick the endpoint matching the user's provider:

| Provider | Endpoint |
|---|---|
| GitHub | `POST /api/application.saveGithubProvider` |
| GitLab | `POST /api/application.saveGitlabProvider` |
| Gitea | `POST /api/application.saveGiteaProvider` |
| Bitbucket | `POST /api/application.saveBitbucketProvider` |
| Generic Git (SSH/HTTPS) | `POST /api/application.saveGitProvider` |

Body shape (GitHub example — others are similar; use the user's `/swagger` to confirm fields per provider):

```bash
curl -sS -X POST "$DOKPLOY_URL/api/application.saveGithubProvider" \
  -H "Authorization: Bearer $DOKPLOY_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "applicationId": "<applicationId>",
    "repository": {
      "owner": "<owner>",
      "repo": "<repo>",
      "branch": "main"
    },
    "buildPath": "/"
  }'
```

`buildPath` is the path inside the repo to use as the build context (default `/` = repo root).

The provider must already be authorised in the Dokploy UI under **Settings → Git**. If the user hasn't connected the provider, this call will 400. **Tell the user to connect it in the UI first** — there is no API path for OAuth.

## Step 5 — (Optional) Set the build type

Default is Dockerfile. To override:

```bash
curl -sS -X POST "$DOKPLOY_URL/api/application.saveBuildType" \
  -H "Authorization: Bearer $DOKPLOY_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "applicationId": "<applicationId>",
    "buildType": "nixpacks"
  }'
```

Build types: `dockerfile` (default), `nixpacks`, `heroku_buildpacks`, `paketo_buildpacks`, `static`.

## Step 6 — Trigger the initial deploy

```bash
curl -sS -X POST "$DOKPLOY_URL/api/application.deploy" \
  -H "Authorization: Bearer $DOKPLOY_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "applicationId": "<applicationId>",
    "title": "initial deploy"
  }'
```

Then poll with [`deployment-status.md`](./deployment-status.md).

## Common errors

| Step | Status | Meaning | Fix |
|---|---|---|---|
| Any | 401 / 403 | Token wrong / insufficient perms | Stop. |
| Project create | 400 | Name already exists | Use the existing `projectId` from step 1. |
| Application create | 400 | App name duplicate in env | Pick a different name. |
| Save provider | 400 | Provider not connected in UI | Stop. Tell user to connect in Dokploy → Settings → Git. |
| Save provider | 404 | Bad `applicationId` | Did step 3 succeed? |
| Deploy | 500 | Build queue or Dockerfile missing | Show body. Check the repo has a Dockerfile at `buildPath`. |

## Follow-ups

- "Did it deploy?" → [`deployment-status.md`](./deployment-status.md).
- "I need env vars set" → [`update-env.md`](./update-env.md) **before** the initial deploy, then redeploy.
- "I want a custom domain" → not in v1; tell user to set in UI for now.

## See also

- [`../entity-model.md`](../entity-model.md) for the hierarchy you're creating
- [`update-env.md`](./update-env.md) for setting env before first deploy
- [`deployment-status.md`](./deployment-status.md) for polling the deploy
