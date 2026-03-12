# CodeCapsules API Reference

Detailed payload structures and endpoint specifics. Read this when you need to create capsules or work with builds.

## Create capsule payload

`POST https://api-v2.codecapsules.io/api/capsules/capsules`

The `space` field must be the **complete space object** as returned by `GET /api/spaces/slug/<slug>` — do not trim fields. The API (especially for data capsules) returns 500 if fields like `createdAt`, `team`, `cluster.region`, `cluster.cloud`, etc. are missing.

```json
{
  "space": {
    "id": "<space.id>",
    "clusterId": "<space.clusterId>",
    "teamId": "<space.teamId>",
    "namespaceKey": "<space.namespaceKey>",
    "cluster": {
      "id": "<space.cluster.id>",
      "clusterApiEndpoint": "<space.cluster.clusterApiEndpoint>",
      "clusterApiId": "<space.cluster.clusterApiId>"
    }
  },
  "capsuleRequest": {
    "name": "my-app",
    "description": "My application",
    "manifest": {
      "manifestType": "backend",
      "repo": {
        "repoId": "<repo-id>",
        "branch": "main",
        "url": "https://github.com/org/repo"
      },
      "entrypointCommand": "npm start",
      "userBuildCommand": "npm run build"
    },
    "products": {
      "cpu": { "qty": 50, "unit": "m" },
      "memory": { "qty": 0.2, "unit": "G" },
      "storage": { "qty": 0, "unit": "G" },
      "replicaScale": { "qty": 1 },
      "gpu": { "qty": 0 }
    }
  }
}
```

**Response (200):** `{"data": { <full capsule object> }, "meta": {}}` — the capsule object inside `data` includes `id`, `name`, `type`, `repo`, `jsonManifest`, `products`, `status`, `createdOn`, etc. Always read the capsule ID from `response.data.id`, not `response.id`.

## Manifest fields by capsule type

| User type | `manifestType` | Required manifest fields |
|-----------|---------------|------------------------|
| `backend` | `"backend"` | `repo`, optionally `entrypointCommand` |
| `frontend` | `"frontend"` | `repo`, optionally `userBuildCommand`, `userBuildOutputDirectory` |
| `docker` | `"docker"` | `repo`, optionally `servePort` (as string, e.g. `"3000"`) |
| `agent` | `"agent"` | `repo` |
| `wordpress` | `"wordpress"` | — |
| `mysql` | `"data"` | `"dataType": "mysql"` |
| `mongodb` | `"data"` | `"dataType": "MongoDb"` |
| `documentdb` | `"data"` | `"dataType": "DocumentDb"` |
| `postgresql` | `"data"` | `"dataType": "PostgreSQL"` |
| `redis` | `"data"` | `"dataType": "Redis"` |
| `storage` | `"data"` | `"dataType": "PersistentStorageGanesha"` |

Note: use `dataType` (not `dbType`) in the **request** manifest. The response `jsonManifest` uses `dbType` instead — these are different. Casing on values must be exact.

## Default products (API minimums)

These are the minimal resources applied when no plan/products are specified. They are below the Basic plan and may not be suitable for production:

| Type | cpu (m) | memory (G) | storage (G) |
|------|---------|-----------|-------------|
| `backend` | 50 | 0.2 | 0 |
| `frontend` | 30 | 0.12 | 0 |
| `docker` | 50 | 0.2 | 0 |
| `agent` | 50 | 0.2 | 0 |
| Data capsules | 0 | 0 | 10 |

All types: `replicaScale.qty: 1`, `gpu.qty: 0`.

## Plans endpoint

`GET https://api-v2.codecapsules.io/api/plans`

Returns `{"data": [...]}`. Filter client-side by `clusterId` and `capsuleType`. Three tiers: Basic, Standard, Premium. Each plan has:
- `name`, `cpu` (cores), `ram` (GB), `storage` (GB), `pricePerMonth`
- `sortKey` — sort ascending for cheapest-first (Basic=1, Standard=2, Premium=3)
- `capsuleType` — e.g. `backend`, `frontend`, `docker`, `mongo`, `postgresql`, etc.
- `clusterId` — region the plan applies to

To apply a plan, convert to `products` fields:
- `cpu.qty` = `plan.cpu × 1000` (cores → millicores), `unit: "m"`
- `memory.qty` = `plan.ram`, `unit: "G"`
- `storage.qty` = `plan.storage`, `unit: "G"`

## Repos endpoints

| Endpoint | Returns |
|----------|---------|
| `GET /api/git/repo/connected` | All connected repos across teams |
| `GET /api/git/team-repo/<team-id>` | Repos for a specific team |

Each repo: `id`, `name`, `url`. Use `id` in the manifest's `repo.repoId` field.

## Spaces endpoints

| Endpoint | Method | Purpose |
|----------|--------|---------|
| `GET /api/spaces` | GET | List all spaces |
| `GET /api/spaces/slug/<slug>` | GET | Get space by slug (full object with cluster info) |
| `POST /api/spaces` | POST | Create a space: `{"name": "...", "teamId": "...", "clusterId": "..."}` |

Space deletion is not available via the API — use the web UI.

## Teams endpoints

| Endpoint | Method | Purpose |
|----------|--------|---------|
| `GET /api/teams` | GET | List all teams (includes `members`, `owner`, `spaces`) |
| `GET /api/teams/<team-id>` | GET | Get a specific team |

Each team member has: `id`, `email`, `role` (`"owner"` or `"user"`). Inviting/removing members is not available via the API.

## Clusters endpoint

`GET /api/clusters` — returns `{"data": [...]}` with available regions:

| Cluster ID | Name | Endpoint |
|------------|------|----------|
| `za-1` | South Africa | `https://za-1.codecapsules.io` |
| `az-uk-1` | United Kingdom | `https://capsule-api.az-uk-1.ccdns.co` |
| `aws-eu-1` | Europe | `https://capsule-api.aws-eu-1.ccdns.co` |
| `us-1` | USA | `https://capsule-api.us-1.ccdns.co` |

## Environment variables (configs)

| Endpoint | Method | Purpose |
|----------|--------|---------|
| `GET /capsules/<id>/configs` | GET | List all env vars |
| `PUT /capsules/<id>/configs` | PUT | Replace all user-set env vars |

GET response:
```json
{"configs": [{"id": "...", "capsuleId": "...", "key": "MY_VAR", "value": "my-value", "setBy": "User", "createdOn": "..."}]}
```

PUT payload (replaces full list — include all vars you want to keep):
```json
{"configs": [{"key": "MY_VAR", "value": "my-value", "setBy": "User"}]}
```

`setBy` values: `"User"` (custom vars), `"System"` (auto-injected from bindings — do not overwrite these).

## Scale/resize capsule

`PATCH <cluster-endpoint>/capsules/<capsule-id>/products`

All four fields are required:

```json
{
  "cpu": { "qty": 50, "unit": "m" },
  "memory": { "qty": 0.2, "unit": "G" },
  "storage": { "qty": 0, "unit": "G" },
  "replicaScale": { "qty": 1 }
}
```

Returns the full updated capsule object (200).

## Domains endpoints

All on the cluster endpoint:

| Endpoint | Method | Purpose |
|----------|--------|---------|
| `GET /capsules/<id>/domains` | GET | List domains (includes SSL status, ingress IPs) |
| `POST /capsules/<id>/domains` | POST | Add domain: `{"hostname": "example.com"}` |
| `DELETE /capsules/<id>/domains/<domain-id>` | DELETE | Remove domain (204) |

Response fields: `domains[]` (`id`, `hostname`, `isMain`, `status`), `ingressAddresses` (IPs for A records), `capsuleHostName`.

## Builds endpoints

| Endpoint | Method | Purpose |
|----------|--------|---------|
| `GET /capsules/<id>/builds` | GET | List builds for a capsule |
| `POST /capsules/<id>/builds` | POST | Trigger a manual build (body: `{}`) |
| `GET /builds/<build-id>/logs` | GET | Get build logs (see time conversion below) |

Build object fields: `id`, `status` (`Starting`/`Building`/`Succeeded`/`Failed`), `type` (`manual`/`auto`), `isDeployed`, `createdOn`, `finishedOn`, `metadata` (`branch`, `commit.latest`, `repoUrl`).

## Logs endpoints

| Endpoint | Method | Purpose |
|----------|--------|---------|
| `GET /capsules/<id>/logs` | GET | System/runtime logs |
| `GET /capsules/<id>/access-logs` | GET | HTTP access logs |

Both use the same query params: `?limit=100&start=<ns>&end=<ns>&direction=forward`

System log entry: `logType: "app"`, JSON `log` field with `level`, `msg`, logger info.
Access log entry: `logType: "access"`, JSON `log` field with `RequestMethod`, `RequestPath`, `RequestHost`, `OriginStatus`, `Duration`, `DownstreamContentSize`, `request_User-Agent`.

Response includes: `data.logs[]`, `data.totalLogCount`, `data.start`, `data.end`.

## Metrics endpoint

`GET <cluster-endpoint>/capsules/<id>/metrics?start=<ISO>&end=<ISO>&intervalSeconds=<n>`

- `start` / `end` — ISO 8601 timestamps (e.g. `2026-03-11T09:00:00.000Z`). NOT nanoseconds.
- `intervalSeconds` — data point granularity (e.g. `2`, `30`, `60`)

Response:
```json
{
  "data": {
    "cpuUsage": { "instances": [{"name": "pod-name", "values": [[unix_s, value], ...]}], "unit": "cpu", "name": "cpu usage" },
    "memUsage": { "instances": [...], "unit": "b", "name": "memory usage" },
    "netTxUsage": { "instances": [...], "unit": "b", "name": "network transmit usage" },
    "netRxUsage": { "instances": [...], "unit": "b", "name": "network receive usage" }
  }
}
```

Each `values` entry is `[unix-seconds, metric-value]`. CPU is in cores (e.g. `0.003` = 3 millicores). Memory and network are in bytes. Instances are per-replica.

## Build log time conversion

Build objects have `createdOn` and `finishedOn` as ISO strings (e.g. `2026-03-06T11:45:27.000Z`). These are always UTC — parse them as UTC regardless of the local timezone (e.g. use `TZ=UTC date` or `date -u`). The logs endpoint needs nanosecond Unix timestamps:

1. Parse the ISO string to a Unix timestamp in seconds (as UTC)
2. Multiply by 1,000,000,000 to get nanoseconds
3. Subtract 60 billion from `createdOn` (60s buffer before)
4. Add 300 billion to `finishedOn` (5min buffer after)
