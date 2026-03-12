---
name: codecapsules
description: "Manage CodeCapsules spaces, capsules, and deployments via the CodeCapsules REST API. Use when the user asks to list, create, or delete capsules, check build status, deploy apps, bind databases, or work with any CodeCapsules resource. Also use when the user mentions deploying to CodeCapsules, checking their CC spaces, or managing cloud hosting on codecapsules.io, even if they don't say 'capsule' explicitly."
---

# CodeCapsules Skill

Manage CodeCapsules (codecapsules.io) cloud hosting via its REST API — spaces, capsules, deployments, databases, and builds.

Use `curl` (or write a short script if the task requires it) to call the API directly. All the endpoints, auth flow, and payload structures you need are documented here and in `references/api.md`.

## ⚠️ Safety: Destructive Actions

**Before deleting any capsule, you MUST confirm with the user.** Deletion is irreversible. Even if the user says "delete X", treat it as a request — list the capsule, show its name and type, ask for explicit confirmation, and wait for a "yes" before running the DELETE. See the full protocol under "Delete a capsule" below. Never batch-delete without confirming each capsule individually.

## Authentication

Every API call needs a Bearer token. To get one, log in with the user's codecapsules.io credentials.

The user needs two environment variables set:
- `CC_EMAIL` — their codecapsules.io login email
- `CC_PASSWORD` — their codecapsules.io password

If either is not set, ask the user to provide them.

### Retrieve the platform API key

The login endpoint requires an `x-api-key` header. This key is a public frontend key embedded in the Code Capsules web app. Retrieve it automatically before authenticating:

```bash
# 1. Fetch the app shell and find the environment chunk
CC_CHUNK=$(curl -s https://app.codecapsules.io | grep -oE 'chunk-[A-Z0-9]+\.js' | while read chunk; do
  curl -s "https://app.codecapsules.io/$chunk" | grep -q 'appstraxServicesApiKey' && echo "$chunk" && break
done)

# 2. Extract the API key from the chunk
CC_API_KEY=$(curl -s "https://app.codecapsules.io/$CC_CHUNK" | grep -oE 'appstraxServicesApiKey:"[^"]*"' | cut -d'"' -f2)
```

If this fails (e.g. the frontend structure changes), ask the user to find the key manually: open browser DevTools Network tab, log in to app.codecapsules.io, and copy the `x-api-key` header from the login request.

### Log in

```bash
curl -s -X POST "https://appstrax-services.codecapsules.io/api/auth/login" \
  -H "Content-Type: application/json" \
  -H "x-api-key: $CC_API_KEY" \
  -d '{"email": "'"$CC_EMAIL"'", "password": "'"$CC_PASSWORD"'", "type": "[User] Login Initiated"}'
```

This returns `{"token": "<jwt>"}`. Use it as `Authorization: Bearer <token>` on all subsequent requests. The token is a JWT that expires periodically — if you get a 401, re-authenticate.

## API base URLs

| Base URL | Purpose |
|----------|---------|
| `https://api-v2.codecapsules.io/api` | Main API — spaces, teams, repos, plans, capsule creation |
| `<cluster-api-endpoint>` | Per-cluster ops — listing/deleting capsules, builds, domains, bindings, logs. Get this from the space object's `cluster.clusterApiEndpoint` field |

## Common workflows

### List teams

```bash
curl -s -H "Authorization: Bearer $TOKEN" \
  "https://api-v2.codecapsules.io/api/teams"
```

Returns `{"data": [...]}`. Each team has: `id`, `name`, `slug`, `isPersonal`, `members` (array of `{id, email, role}`), `owner`, and `spaces` (array of space objects).

### List available regions/clusters

```bash
curl -s -H "Authorization: Bearer $TOKEN" \
  "https://api-v2.codecapsules.io/api/clusters"
```

Returns `{"data": [...]}`. Each cluster has: `id`, `name`, `clusterApiEndpoint`. Known clusters:
- `za-1` — South Africa (Cape Town)
- `az-uk-1` — United Kingdom (South)
- `aws-eu-1` — Europe (North)
- `us-1` — USA

### List spaces

```bash
curl -s -H "Authorization: Bearer $TOKEN" \
  "https://api-v2.codecapsules.io/api/spaces"
```

Returns `{"data": [...]}`. Each space has: `id`, `slug`, `name`, `clusterId`, `teamId`, `namespaceKey`, and `cluster.clusterApiEndpoint`.

Space slugs look like `my-space-abcd` (name + 4-char suffix).

### Create a space

```bash
curl -s -X POST -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  "https://api-v2.codecapsules.io/api/spaces" \
  -d '{"name": "<space-name>", "teamId": "<team-id>", "clusterId": "<cluster-id>"}'
```

Returns the created space object. The `teamId` comes from `GET /api/teams` and `clusterId` from `GET /api/clusters`.

### List capsules in a space

First get the space by slug, then use its cluster endpoint:

```bash
# Get the space
SPACE=$(curl -s -H "Authorization: Bearer $TOKEN" \
  "https://api-v2.codecapsules.io/api/spaces/slug/<space-slug>")

# Extract cluster endpoint and namespace from the space
ENDPOINT=$(echo $SPACE | jq -r '.data.cluster.clusterApiEndpoint')
NS=$(echo $SPACE | jq -r '.data.namespaceKey')

# List capsules
curl -s -H "Authorization: Bearer $TOKEN" \
  "$ENDPOINT/namespaces/$NS/capsules"
```

Returns `{"capsules": [...]}`. Each capsule has: `id`, `name`, `type`, `status`, `products`, `repo`, `jsonManifest`, `shouldAutoBuild`, `currentDeploymentId`, `createdOn`. For data capsules, the `type` field is `"data"` — check `jsonManifest.dbType` for the specific database type.

### List connected repos

```bash
curl -s -H "Authorization: Bearer $TOKEN" \
  "https://api-v2.codecapsules.io/api/git/repo/connected"
```

Returns `{"data": [...]}` with `id`, `name`, `url` for each repo.

### Create a capsule

Follow this order — each step needs info from the previous one:

1. **List spaces** → get the space slug
2. **Get space by slug** → get the full space object (you need its IDs and cluster info)
3. **List repos** → get the repo `id` and `url` (for app capsules)
4. **Confirm with the user** — show them the name, type, and repo
5. **POST to create** — see `references/api.md` for the full payload structure

**Important:** The create endpoint sometimes returns 400 or 500 even when the capsule was actually created (likely a timeout or race condition). Always verify by listing capsules in the space after a create call, regardless of the HTTP response.

After creation, you can **trigger a build** immediately using `POST /capsules/<id>/builds` (see below), or remind the user that builds are also triggered automatically by `git push`.

### Delete a capsule

**IMPORTANT — deletion is irreversible.** You MUST follow these steps in order:

1. **List capsules** in the space to find the capsule name and ID
2. **Show the user** the capsule name, type, and ID you intend to delete
3. **Ask the user to explicitly confirm** (e.g. "Please confirm you want to delete capsule 'my-app'")
4. **Wait for confirmation** — do NOT proceed until the user says yes
5. Only then run the DELETE request

Never skip the confirmation step, even if the user provided the capsule ID directly. Even if the user says "delete X", treat it as a request and confirm before executing.

**⚠️ STOP: You must confirm with the user before running this command. See the safety section at the top of this file.**

```bash
curl -s -X DELETE -H "Authorization: Bearer $TOKEN" \
  "$ENDPOINT/namespaces/$NS/capsules/<capsule-id>"
```

### Bind a data capsule to an app capsule

This injects connection environment variables (`DATABASE_URL`, `TEXTFILE_URL`, etc.) into the app capsule:

```bash
curl -s -X PATCH -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  "$ENDPOINT/data-capsule/<data-capsule-id>/bind-to" \
  -d '{"capsuleId": "<backend-capsule-id>"}'
```

### Unbind a data capsule from an app capsule

```bash
curl -s -X PATCH -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  "$ENDPOINT/data-capsule/<data-capsule-id>/unbind-from" \
  -d '{"capsuleId": "<backend-capsule-id>"}'
```

### Trigger a build

You can trigger a build manually without a `git push`:

```bash
curl -s -X POST -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  "$ENDPOINT/capsules/<capsule-id>/builds" -d '{}'
```

Returns the build object with `id`, `status` (`Starting`, `Building`, `Succeeded`, `Failed`), `type` (`manual`), and `metadata` (branch, commit).

### Check build status

```bash
# List all builds for a capsule (most recent first)
curl -s -H "Authorization: Bearer $TOKEN" \
  "$ENDPOINT/capsules/<capsule-id>/builds"
```

Returns `{"builds": [...]}`. Each build has: `id`, `status`, `type`, `isDeployed`, `createdOn`, `finishedOn`, `metadata` (branch, commit info).

Build status values: `Starting`, `Building`, `Succeeded`, `Failed`.

### Get build logs

```bash
curl -s -H "Authorization: Bearer $TOKEN" \
  "$ENDPOINT/builds/<build-id>/logs?limit=500&start=<start-ns>&end=<end-ns>&direction=forward"
```

**Timestamp range is critical.** The `start` and `end` parameters are nanosecond Unix timestamps. You must pad generously — build logs often start well before the build's `createdOn` and extend after `finishedOn`. Pad with **at least 120 seconds before** `createdOn` and **5 minutes after** `finishedOn`. Do NOT use `0` or arbitrarily large values — the API returns 500 for out-of-range timestamps.

```bash
# Convert build createdOn/finishedOn to nanosecond timestamps with padding
START_NS=$(python3 -c "from datetime import datetime, timedelta; t=datetime.fromisoformat('BUILD_CREATED_ON'.replace('Z','+00:00')) - timedelta(seconds=120); print(int(t.timestamp()*1e9))")
END_NS=$(python3 -c "from datetime import datetime, timedelta; t=datetime.fromisoformat('BUILD_FINISHED_ON'.replace('Z','+00:00')) + timedelta(minutes=5); print(int(t.timestamp()*1e9))")
```

**Note:** Build log responses often contain ANSI escape codes and other control characters that break `jq`. Strip them before parsing, e.g.: `curl ... | sed 's/\x1b\[[0-9;]*m//g' | jq .` or use `python3 -c "import json, sys; print(json.dumps(json.loads(sys.stdin.read()), indent=2))"`.

Each log entry has: `id`, `capsuleId`, `buildId`, `time` (nanosecond string), `log` (plain text, may contain ANSI codes), `logType` (`build`), `replicaName`. The response wraps logs in `data.logs` (not top-level `logs`).

### Get system logs (runtime)

Same endpoint format as build logs but on the capsule:

```bash
curl -s -H "Authorization: Bearer $TOKEN" \
  "$ENDPOINT/capsules/<capsule-id>/logs?limit=100&start=<start-ns>&end=<end-ns>&direction=forward"
```

Same timestamp rules apply — pad generously, do not use `0` or very large values.

Each log entry has: `id`, `capsuleId`, `time` (nanosecond string), `log` (JSON string with level, message, etc.), `logType` (`app`), `replicaName`, `stream`.

### Manage environment variables

```bash
# Get all env vars for a capsule
curl -s -H "Authorization: Bearer $TOKEN" \
  "$ENDPOINT/capsules/<capsule-id>/configs"

# Set env vars (PUT replaces all user-set vars — include all you want to keep)
curl -s -X PUT -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  "$ENDPOINT/capsules/<capsule-id>/configs" \
  -d '{"configs":[{"key":"MY_VAR","value":"my-value","setBy":"User"}]}'
```

GET returns `{"configs": [...]}` with `id`, `key`, `value`, `setBy` for each var. PUT replaces the full list — always include all vars you want to keep (read first, append, then write back). The `setBy` field should be `"User"` for custom vars. System-injected vars (from bindings) have `setBy: "System"`.

### Scale/resize a capsule

Update CPU, RAM, storage, or replica count on an existing capsule:

```bash
curl -s -X PATCH -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  "$ENDPOINT/capsules/<capsule-id>/products" \
  -d '{"cpu":{"qty":50,"unit":"m"},"memory":{"qty":0.2,"unit":"G"},"storage":{"qty":0,"unit":"G"},"replicaScale":{"qty":1}}'
```

All four fields (`cpu`, `memory`, `storage`, `replicaScale`) are required. Returns the full updated capsule object. Use `GET /api/plans` to find valid resource combinations.

**Stop a capsule**: set `"replicaScale": {"qty": 0}` to scale to zero replicas.
**Start/restart a capsule**: set `"replicaScale": {"qty": 1}` (or higher) to bring it back up.

### Get capsule metrics

```bash
curl -s -H "Authorization: Bearer $TOKEN" \
  "$ENDPOINT/capsules/<capsule-id>/metrics?start=<ISO-timestamp>&end=<ISO-timestamp>&intervalSeconds=30"
```

Uses ISO 8601 timestamps (e.g. `2026-03-11T09:00:00.000Z`) — NOT nanoseconds. `intervalSeconds` controls the granularity (e.g. `2` for fine-grained, `30` or `60` for overview).

Returns `{"data": {"cpuUsage": {...}, "memUsage": {...}, "netTxUsage": {...}, "netRxUsage": {...}}}`. Each metric has:
- `instances[]` — one per replica, with `name` (pod name) and `values` (array of `[unix-seconds, value]` pairs)
- `unit` — `"cpu"` (cores), `"b"` (bytes for memory and network)
- `name` — human-readable name

### Get access logs (HTTP requests)

```bash
curl -s -H "Authorization: Bearer $TOKEN" \
  "$ENDPOINT/capsules/<capsule-id>/access-logs?limit=100&start=<start-ns>&end=<end-ns>&direction=forward"
```

Same time format as system logs (nanosecond Unix timestamps). Each log entry has `logType: "access"` and a JSON `log` field containing: `RequestMethod`, `RequestPath`, `RequestHost`, `OriginStatus`, `Duration`, `DownstreamContentSize`, `time`, and user-agent info.

### Manage custom domains

```bash
# List domains for a capsule
curl -s -H "Authorization: Bearer $TOKEN" \
  "$ENDPOINT/capsules/<capsule-id>/domains"

# Add a custom domain
curl -s -X POST -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  "$ENDPOINT/capsules/<capsule-id>/domains" \
  -d '{"hostname": "myapp.example.com"}'

# Remove a custom domain (use the domain ID from the list response)
curl -s -X DELETE -H "Authorization: Bearer $TOKEN" \
  "$ENDPOINT/capsules/<capsule-id>/domains/<domain-id>"
```

The list response includes: `domains` (array with `id`, `hostname`, `isMain`, `status` with SSL cert info), `ingressAddresses` (A record IPs for DNS), and `capsuleHostName` (default hostname).

To add a custom domain, the user must configure DNS:
- **Root domain** → A record pointing to the IP(s) in `ingressAddresses`
- **Subdomain** → CNAME record pointing to the `capsuleHostName`

Maximum 5 subdomains per capsule. SSL certificates are provisioned automatically.

## Capsule types

| Type | Category | Needs repo? | Notes |
|------|----------|-------------|-------|
| `backend` | App | Yes | Node.js, Python, Go, Java, Ruby servers. Supports Procfile and custom run commands |
| `frontend` | App | Yes | Static sites (React, Angular, Vue, Svelte, Next.js). Set `userBuildCommand` and `userBuildOutputDirectory` |
| `docker` | App | Yes | Custom Dockerfile. Set `servePort` (as string). Useful for PHP, Laravel, Caddy, or any custom stack |
| `agent` | App | Yes | AI agent capsules. Set env vars `PROVIDER_NAME`, `PROVIDER_MODEL`, `PROVIDER_API_KEY` via the UI. Has built-in chat interface and RAG support |
| `wordpress` | App | No | Managed WordPress. Requires binding to both a MySQL data capsule and a storage capsule |
| `mysql` | Data | No | MySQL database |
| `mongodb` | Data | No | MongoDB database. Supports public access toggle (via UI) |
| `postgresql` | Data | No | PostgreSQL database |
| `documentdb` | Data | No | DocumentDB |
| `redis` | Data | No | Redis in-memory store |
| `storage` | Data | No | Persistent file storage. Accessible via WebDAV protocol. Can be bound to multiple app capsules |

### WordPress setup

WordPress capsules require two data capsules bound to them:
1. A **MySQL** data capsule (for the WordPress database)
2. A **Storage** capsule (for persistent file storage — themes, uploads, plugins)

Create all three capsules, then bind both data capsules to the WordPress capsule. The platform auto-injects `DATABASE_URL` and `PERSISTENT_STORAGE_DIR` environment variables.

### Agent capsule notes

Agent capsules support LLM providers (Anthropic, Google Gemini, custom). Key environment variables (set via UI):
- `PROVIDER_NAME` — e.g. `anthropic`, `google`
- `PROVIDER_MODEL` — e.g. `claude-sonnet-4-20250514`
- `PROVIDER_API_KEY` — the provider's API key

Template repos are available on the CodeCapsules GitHub org (base agent, Telegram bot agent, calendar agent).

### Storage capsule notes

Storage capsules provide a virtual cloud hard drive accessible via WebDAV. After binding to an app capsule, files are available at the path in the `PERSISTENT_STORAGE_DIR` environment variable. The storage can also be browsed via WebDAV clients (macOS Finder, Windows Explorer) when public access is enabled via the UI.

## Plans and pricing

CodeCapsules has three plan tiers: **Basic**, **Standard**, and **Premium**. Pricing varies by capsule type and region. There is no free tier — Basic is the cheapest (e.g. frontend from $3/month, backend from $10.50/month).

Use `GET /api/plans` to list plans, then filter by `capsuleType` and `clusterId`. When the user doesn't specify a plan, use Basic (lowest `sortKey`). To apply a plan, compute the `products` fields from it:
- `cpu.qty` = `plan.cpu × 1000` (convert cores to millicores)
- `memory.qty` = `plan.ram`
- `storage.qty` = `plan.storage`

If resource fields are omitted from the create payload, the API uses minimal defaults (below the Basic plan) — these are very low and may not be suitable for production.

## Capsule URLs

- **Frontend** capsules: `https://<capsule-name>.codecapsules.co.za`
- **Backend/Docker** capsules: hostname returned in the capsule's domain listing (`GET /capsules/<id>/domains` → `capsuleHostName`)
- **Custom domains**: can be added to any app capsule (see "Manage custom domains" above)

## When Things Go Wrong

If an API call returns an unexpected result (wrong status code, empty response, missing fields):

1. **Verify the actual state** — list the space's capsules to check whether the resource was actually created, deleted, or modified despite the error
2. **Check your payload** — compare field names and structure against this skill's documentation. Do not guess alternative field names (e.g. `manifest` vs `jsonManifest`, `repoId` vs `repoKey`)
3. **Retry once** — transient errors (502, 503, timeouts) may resolve on a second attempt
4. **Stop and ask the user** — if the issue persists after one retry, do not brute-force different endpoint paths or payload structures. Follow the self-update protocol below instead

## Known API gaps

The following platform features are available in the CodeCapsules web UI but **do not have known API endpoints**. Do NOT guess endpoints for these — if the user needs them, ask them to perform the action in the UI or see the "Self-update" section below.

- **Space deletion** — deleting a space
- **Backups** — creating, listing, or restoring backups
- **HTTP Basic Auth** — password-protecting a capsule URL
- **Team member management** — inviting or removing team members (listing works via `GET /api/teams`)
- **Repository connection** — connecting new GitHub repos (OAuth flow, must be done in UI)

## Self-update protocol

If you encounter an API endpoint or workflow that is **not documented in this skill**, do NOT guess the endpoint path, parameter names, or payload structure. Instead:

1. Tell the user what you're trying to do and that the endpoint isn't documented yet
2. Ask the user to perform the action in the CodeCapsules web UI with browser DevTools Network tab open
3. Ask them to paste the `curl` equivalent of the request they see (most browsers let you right-click a request → Copy as cURL)
4. Once you have the verified curl command, use it to complete the task
5. **Update this skill**: add the new endpoint to `SKILL.md` and/or `references/api.md` so future sessions don't hit the same gap. Remove the entry from the "Known API gaps" section if it's now covered

This keeps the skill accurate and growing without ever guessing wrong.

For full API payload structures and manifest details, read `references/api.md`.
