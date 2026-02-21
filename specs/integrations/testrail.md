<!-- spec-meta last-updated-sha: HEAD last-updated-pr: null last-updated-date: 2026-02-20 spec-version: 1.0 -->

# TestRail Integration

tacotruck can both pull data from TestRail (source) and push data to TestRail (target). Both directions use TestRail's REST API v2.

---

## Connection configuration

**Config file:** `configs/testrail.json`

| Field | Value |
|---|---|
| `name` | `"testrail"` |
| `type` | `"api"` |
| `base_path` | `"index.php?"` |
| `auth.type` | `"basic"` |
| `requests_per_second` | `2` |

**Auth scheme:** HTTP Basic Authentication.

```
Authorization: Basic <base64Credentials>
```

Where `base64Credentials` is the base64-encoding of `username:password` (or `username:api_key`).

**Full URL structure:**

```
{base_url}{base_path}{api_path}
```

Example: `https://example.testrail.io/index.php?api/v2/get_projects`

**Rate limit:** `requests_per_second: 2` is enforced by the API client across all requests to this integration.

---

## Pagination

TestRail uses offset-based pagination via query string parameters.

| Paging config field | Value |
|---|---|
| `location` | `"response"` |
| `link_key` | `"_links.next"` |
| `options.location` | `"querystring"` |
| `options.limit.key` | `"limit"` |
| `options.limit.value` | `"250"` |
| `options.offset.key` | `"offset"` |

The extractor reads `response._links.next` to determine whether a next page exists. That value is used verbatim as the URL for the next request.

Each request fetches up to 250 records.

---

## Entity types

### As a source (pulling from TestRail)

| Entity type | API path | Response key | Notes |
|---|---|---|---|
| `projects` | `api/v2/get_projects` | `projects` | All active projects |
| `suites` | `api/v2/get_suites/{projects.id}` | (array) | One request per project; empty for single-suite projects |
| `sections` | `api/v2/get_sections/{projects.id}&suite_id={suites.id}` | (array) | Hierarchical; parent section has `parent_id` |
| `cases` | `api/v2/get_cases/{projects.id}&suite_id={suites.id}` | `cases` | Denormalized: one request per suite per project |
| `runs` | `api/v2/get_runs/{projects.id}` | `runs` | All runs in a project |
| `results` | `api/v2/get_results_for_run/{runs.id}` | `results` | All result records for a run |
| `plans` | `api/v2/get_plans/{projects.id}` | `plans` | All plans |
| `milestones` | `api/v2/get_milestones/{projects.id}` | `milestones` | Supports nested milestones |

#### Dependency chains

The following dependency chains are enforced during extraction:

| Entity | Depends on |
|---|---|
| `suites` | `projects` |
| `sections` | `projects`, `suites` |
| `cases` | `projects`, `suites` |
| `runs` | `projects` |
| `results` | `runs` |
| `plans` | `projects` |
| `milestones` | `projects` |

#### Denormalized keys

The `cases` entity type requires a double expansion. The config defines:

```json
"denormalized_keys": {
  "cases": {
    "suites": {
      "projects.id": "project_id"
    }
  }
}
```

This means: for each suite, use the suite's associated project ID in the cases endpoint URL. The extractor builds URLs such as `api/v2/get_cases/{project_id}&suite_id={suite_id}`.

---

### As a target (pushing to TestRail)

When TestRail is the migration target (not the typical case, but supported), the following endpoints are available.

| Entity type | Operation | API path | Notes |
|---|---|---|---|
| `projects` | `create` | `api/v2/add_project` | Single record only; `data_key` is empty string (raw body) |
| `projects` | `update` | `api/v2/update_project/{id}` | `update_key: "id"` |
| `suites` | `create` | (TBD) | Bulk submission endpoint not defined in bundled config |
| `cases` | `create` | (TBD) | Bulk submission endpoint not defined in bundled config |

The bundled `testrail.json` source config provides the project-creation and -deletion endpoints explicitly used by the `testrail project:create` and `testrail project:delete` CLI commands.

---

## run:submit flow (detailed)

The `testrail run:submit` command uses a specialized submission path (`TestRailETL.submitTestRun`) that differs from the generic ETL pipeline.

### Input transformation

Before any API calls, the parsed XML/JSON data is passed through `transformXmlData`, which restructures it into:

```typescript
{
  sections: Array<{
    name: string,
    cases: Array<{
      title: string,
      type_id: number,      // 3 = "Automated"
      template_id: number,  // 1 = "Test Case (Text)"
      priority_id: number,  // 2 = "Medium"
      custom_preconds?: string,
      custom_steps?: string,
      custom_expected?: string,
    }>
  }>,
  results: Array<{
    title: string,
    status_id: number,
    comment: string,
    elapsed: string,      // "3s", "1m 30s", etc.
    created_by: number,   // default: 1
    attachments?: []
  }>
}
```

### Suite mode detection

1. `GET api/v2/get_project/{project_id}` → check `suite_mode`:
   - `1` or `2`: single suite; the suite ID is obtained from `GET api/v2/get_suites/{project_id}` and the first element is used.
   - `3`: multi-suite; a new suite is created via `POST api/v2/add_suite/{project_id}` with `{ name: runName }`, and the returned `id` is used.

### Section creation

Sections are created one-by-one using `POST api/v2/add_section/{project_id}` with `suite_id` in the body. The batch processor settings for sections:

| Parameter | Value |
|---|---|
| Concurrency | `5` |
| Throttle count | `5` per second |
| Throttle interval | `1000` ms |

Request body per section:
```json
{
  "suite_id": <suite_id>,
  "name": "<section_name>"
}
```

A `casesIdMap` is built mapping `sectionName → section_id`.

### Case creation

For each section, the cases within that section are created using `POST api/v2/add_case/{section_id}`.

Batch processor settings for cases:
| Parameter | Value |
|---|---|
| Concurrency | `5` |

Request body per case:
```json
{
  "title": "<case_title>",
  "type_id": 3,
  "template_id": 1,
  "priority_id": 2
}
```

The returned case `id` is collected into `caseIds[]`.

### Run creation

`POST api/v2/add_run/{project_id}` with:
```json
{
  "suite_id": <suite_id>,
  "name": "<run_name>",
  "include_all": false,
  "case_ids": [<id1>, <id2>, ...]
}
```

The returned run `id` is stored in credentials as `run_id`.

### Results submission

`POST api/v2/add_results_for_cases/{run_id}` with:
```json
{
  "results": [
    {
      "case_id": <case_id>,
      "status_id": <status_id>,
      "comment": "<comment>",
      "elapsed": "<elapsed>"
    },
    ...
  ]
}
```

`status_id` mapping from internal status strings to TestRail status integers:

| Internal status | TestRail status_id |
|---|---|
| `"passed"` | `1` |
| `"failed"` | `5` |
| `"skipped"` | `6` |
| `"untested"` | `3` |
| Any other | `3` (Untested) |

---

## Field mappings (source extraction)

When pulling from TestRail into the intermediate representation:

### projects

| TestRail field | IR field | Notes |
|---|---|---|
| `id` | `source_id` | Primary identifier |
| `name` | `name` | |
| `announcement` | `announcement` | |
| `is_completed` | `is_completed` | Boolean |
| `suite_mode` | `suite_mode` | Integer (1, 2, or 3) |
| `show_announcement` | `show_announcement` | |
| All others | preserved | |

### suites

| TestRail field | IR field |
|---|---|
| `id` | `source_id` |
| `name` | `name` |
| `description` | `description` |
| `project_id` | `project_id` |
| All others | preserved |

### cases

| TestRail field | IR field | Notes |
|---|---|---|
| `id` | `source_id` | |
| `title` | `name` | |
| `section_id` | `parent_id` | Parent section reference |
| `suite_id` | `suite_id` | |
| `priority_id` | `priority` | Integer |
| `type_id` | `type` | Integer |
| `template_id` | `template_id` | |
| `estimate` | `estimate` | String (e.g., `"30s"`) |
| `custom_*` | `custom_fields.*` | All custom field columns |

### runs

| TestRail field | IR field |
|---|---|
| `id` | `source_id` |
| `name` | `name` |
| `description` | `description` |
| `project_id` | `project_id` |
| `suite_id` | `suite_id` |
| `is_completed` | `is_completed` |
| `milestone_id` | `milestone_id` |

### results

| TestRail field | IR field |
|---|---|
| `id` | `source_id` |
| `test_id` | `test_id` |
| `status_id` | `status` |
| `comment` | `comment` |
| `created_on` | `created_on` |
| `created_by` | `created_by` |
| `elapsed` | `elapsed` |

### milestones

| TestRail field | IR field |
|---|---|
| `id` | `source_id` |
| `name` | `name` |
| `description` | `description` |
| `due_on` | `due_on` |
| `started_on` | `started_on` |
| `completed_on` | `completed_on` |
| `parent_id` | `parent_id` |
| `is_completed` | `is_completed` |

---

## Error handling specifics

TestRail API errors are returned as:

```json
{
  "error": "No permissions to delete or edit this test run"
}
```

The API client treats any non-2xx response as an error. The error message includes the HTTP status code, status text, and the parsed `error` field from the body.

TestRail enforces a 250 record/request limit. The paging configuration handles this automatically.

If a project is in "single suite" mode (`suite_mode: 1`), there is exactly one suite per project. The `run:submit` command calls `GET api/v2/get_suites/{project_id}` and uses `suites[0].id`.

If the project has no suite (unexpected), the command logs an error and exits with code `1`.
