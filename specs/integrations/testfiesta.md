<!-- spec-meta last-updated-sha: HEAD last-updated-pr: null last-updated-date: 2026-02-20 spec-version: 1.0 -->

# TestFiesta Integration

tacotruck submits test runs and migration data to TestFiesta and can also pull data from TestFiesta as a migration source.

---

## Connection configuration

**Config file:** `configs/testfiesta.json`

| Field | Value |
|---|---|
| `name` | `"testfiesta"` |
| `type` | `"api"` |
| `base_path` | `"v1/{handle}"` |
| `auth.type` | `"bearer"` |
| `auth.location` | `"header"` |
| `auth.key` | `"Authorization"` |
| `auth.payload` | `"Bearer {token}"` |

**Full URL structure:**

```
{base_url}v1/{handle}/{endpoint_path}
```

Example: `https://api.testfiesta.ai/v1/acme-corp/projects/MY-PROJECT/data`

The `{handle}` and `{token}` placeholders are resolved from the credentials object at request time.

---

## Credentials required

| Field | Description |
|---|---|
| `base_url` | TestFiesta API base URL (e.g., `https://api.testfiesta.ai/`). Must include trailing slash. |
| `token` | Bearer token in the format `TF_<numerics>.<32-hex-chars>` (e.g., `TF_99866737272422404.7f879e26fd76d09ce9ca263f9231f2cf`). |
| `handle` | Organization handle/slug (e.g., `acme-corp`). Used in the `base_path`. |
| `projectKey` | Project key (e.g., `MY-PROJECT`). Used in the `multi_target.path` template. |

---

## Bulk submission endpoint (multi_target)

The TestFiesta integration uses a single "multi_target" endpoint for submitting all entity types. This is the primary path used by `run:submit` and `migrate`.

```json
"multi_target": {
  "path": "/projects/{projectKey}/data",
  "data_key": "entries",
  "include_source": true
}
```

**Resolved URL:** `{base_url}v1/{handle}/projects/{projectKey}/data`

**Request method:** `POST`

**Request body:**

```json
{
  "entries": [
    { "type": "<entity_type>", ...entity_fields... },
    ...
  ]
}
```

All entity records from a single load operation are bundled into one `entries` array. The `type` field is added by the loader based on the entity type name.

If `include_source: true`, each entry includes a `source` field containing the source integration name (e.g., `"testrail"`).

---

## Target endpoints (non-bulk paths)

These paths are used when the multi_target approach is not appropriate, for example when creating a project before any data is submitted.

### `projects.create`

```
POST /projects
body: { "entries": [ {...}, ... ] }
include_source: true
```

Creates one or more projects. `data_key: "entries"`.

### `suites.create`

```
POST api/v1/data
body: { "entries": [ {...}, ... ] }
include_source: true
```

### `cases.create`

```
POST api/v1/data
body: { "entries": [ {...}, ... ] }
include_source: true
```

### `plans.create`

```
POST api/v1/data
body: { "entries": [ {...}, ... ] }
include_source: true
```

### `runs.create`

```
POST api/v1/data
body: { "entries": [ {...}, ... ] }
include_source: true
```

### `executions.create`

```
POST api/v1/data
body: { "entries": [ {...}, ... ] }
include_source: true
```

---

## `run:submit` flow (TestFiesta)

This is the primary use case: submit CI test results to a TestFiesta project.

### Input parsing

`loadRunData(filePath)` handles both formats:

- **`.xml` files** — parsed by `JunitXmlParser` (see `specs/data-formats/input.md`).
- **`.json` files** — parsed as raw JSON.

The result is a `RunData` object (see `specs/data-formats/output.md`).

### ETL load

`TestFiestaETL.load(runData)` transforms the `RunData` into TestFiesta API calls:

1. The `RunData.run` object is mapped to a run entry.
2. Each `RunData.execution` is mapped to an execution entry.
3. All entries are submitted via the `multi_target` endpoint.

### Delete project

`TestFiestaETL.deleteProject()` calls:

```
DELETE /v1/{handle}/projects/{project_id}
```

Mapped via `loadToTarget('projects', {}, 'delete')`.

---

## Source endpoints (pulling from TestFiesta)

When TestFiesta is the migration source, the following read operations are available:

| Entity type | Endpoint |
|---|---|
| `projects` | `GET /v1/{handle}/projects` |
| `suites` | `GET /v1/{handle}/projects/{projectKey}/suites` |
| `cases` | `GET /v1/{handle}/projects/{projectKey}/cases` |
| `runs` | `GET /v1/{handle}/projects/{projectKey}/runs` |
| `executions` | `GET /v1/{handle}/projects/{projectKey}/executions` |

Pagination on TestFiesta source endpoints follows the same `_links.next` pattern as other integrations.

---

## Entity schemas submitted to TestFiesta

The following describes what tacotruck submits to the TestFiesta API for each entity type during a typical migration from TestRail.

### Project

```json
{
  "type": "project",
  "name": "<project name>",
  "key": "<generated handle>",
  "externalId": "<testrail_project_id>",
  "source": "testrail"
}
```

### Run

```json
{
  "type": "run",
  "name": "<run name>",
  "externalId": "<testrail_run_id>",
  "source": "testrail",
  "projectKey": "<project_key>"
}
```

### Execution (test result)

```json
{
  "type": "execution",
  "name": "<test name>",
  "status": "<status string>",
  "duration": <milliseconds>,
  "externalId": "<source_id>",
  "source": "testrail"
}
```

Status values are normalized to:
- `"passed"`
- `"failed"`
- `"skipped"`
- `"untested"`

Duration is parsed from elapsed strings:
- `"30s"` → `30000`
- `"1m 30s"` → `90000`
- `"2m"` → `120000`

---

## Authentication errors

| HTTP Status | Meaning | tacotruck behavior |
|---|---|---|
| `401 Unauthorized` | Invalid or expired token | Logged as error, pipeline aborts |
| `403 Forbidden` | Token valid but insufficient permissions | Logged as error, pipeline aborts |
| `404 Not Found` | Project or handle does not exist | Logged as error per-record in non-strict mode |
| `429 Too Many Requests` | Rate limit exceeded | Retried after `Retry-After` delay |

---

## Project management

### `testfiesta project:create`

Submits a project creation request. The project `key` must be unique within the organization. If the key already exists, the API returns a 409 Conflict error.

### `testfiesta project:delete`

Calls the delete endpoint for the specified project ID. This is an irreversible operation. The project and all associated data are deleted.

---

## Notes on the base path

The `base_path` is `"v1/{handle}"`. This means every endpoint path appended to the base URL includes the organization handle, making all operations inherently scoped to a single organization. It is not possible to submit data to multiple organizations in a single `run:submit` invocation.

To submit to multiple organizations, run `tacotruck` multiple times with different tokens and handles.
