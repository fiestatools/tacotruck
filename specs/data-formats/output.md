<!-- spec-meta last-updated-sha: HEAD last-updated-pr: null last-updated-date: 2026-02-20 spec-version: 1.0 -->

# Output Data Formats

This document describes what tacotruck creates in TestFiesta and TestRail as a result of `run:submit` and `migrate` operations, and the intermediate representation used between pipeline stages.

---

## TestFiesta output (run:submit)

When `testfiesta run:submit` is called, tacotruck submits data to the TestFiesta `multi_target` endpoint. The submitted payload creates the following entities.

### Run

A test run is created from the root-level metadata of the input file.

**Endpoint:** `POST {base_url}v1/{handle}/projects/{projectKey}/data`

**Payload entry (run):**

```json
{
  "type": "run",
  "source": "testfiesta",
  "name": "<root.name or 'Test Run'>",
  "description": "",
  "created_at": "<ISO 8601 timestamp>"
}
```

Fields sourced from `JunitParserResult.root`:

| Output field | Source |
|---|---|
| `name` | `root.name` (default: `"Test Run"`) |
| `description` | Always empty string (`""`) |
| `created_at` | Submission timestamp (not from input) |

### Executions

One execution entry is created per `testcase` in the input.

**Payload entry (execution):**

```json
{
  "type": "execution",
  "source": "testfiesta",
  "name": "<testcase.name>",
  "status": "<normalized_status>",
  "duration": <milliseconds>,
  "suiteName": "<section.name>",
  "comment": "<failure or error message, if any>"
}
```

Fields sourced from `JunitParserResult.testcase[i]`:

| Output field | Source | Notes |
|---|---|---|
| `name` | `testcase.name` | |
| `status` | `testcase.status` | See normalization table below |
| `duration` | `testcase.time * 1000` | Converted from seconds to milliseconds |
| `suiteName` | `testcase.classname` | Used for grouping (not a foreign key) |
| `comment` | `testcase.failure.message` or `testcase.error.message` | Empty string if no failure |

### Status normalization

| JUnit status | TestFiesta status |
|---|---|
| `"passed"` | `"passed"` |
| `"failed"` | `"failed"` |
| `"error"` | `"failed"` |
| `"skipped"` | `"skipped"` |
| `undefined` or unrecognized | `"untested"` |

### Bulk payload structure

All entries are submitted in a single POST:

```json
{
  "entries": [
    { "type": "run", "name": "My Test Run", ... },
    { "type": "execution", "name": "testA", "status": "passed", ... },
    { "type": "execution", "name": "testB", "status": "failed", ... }
  ]
}
```

The `entries` array contains one run entry followed by N execution entries (one per test case).

---

## TestFiesta output (migrate from TestRail)

When migrating from TestRail to TestFiesta, the ETL pipeline creates these entity types in TestFiesta.

### Projects

| TestFiesta field | Source |
|---|---|
| `name` | TestRail `project.name` |
| `key` | Auto-derived (not set by tacotruck; assigned by TestFiesta) |
| `externalId` | TestRail `project.id` (string) |
| `source` | `"testrail"` |

### Suites

Sent via `POST api/v1/data` as entries of `type: "suite"`:

| TestFiesta field | Source |
|---|---|
| `name` | TestRail `suite.name` |
| `description` | TestRail `suite.description` |
| `externalId` | TestRail `suite.id` |
| `source` | `"testrail"` |
| `projectExternalId` | TestRail `project.id` |

### Cases

| TestFiesta field | Source |
|---|---|
| `name` | TestRail `case.title` |
| `externalId` | TestRail `case.id` |
| `source` | `"testrail"` |
| `suiteExternalId` | TestRail `suite.id` |
| `priority` | TestRail `case.priority_id` (integer) |
| `type` | TestRail `case.type_id` (integer) |
| `estimate` | TestRail `case.estimate` (string, e.g., `"30s"`) |
| `customFields` | All `custom_*` fields from TestRail |

### Runs

| TestFiesta field | Source |
|---|---|
| `name` | TestRail `run.name` |
| `description` | TestRail `run.description` |
| `externalId` | TestRail `run.id` |
| `source` | `"testrail"` |
| `projectExternalId` | TestRail `project.id` |
| `milestoneExternalId` | TestRail `run.milestone_id` (if set) |

### Executions (from TestRail `results`)

| TestFiesta field | Source |
|---|---|
| `externalId` | TestRail `result.id` |
| `source` | `"testrail"` |
| `status` | Normalized from TestRail `result.status_id` |
| `caseExternalId` | Resolved from `result.test_id` → `test.case_id` |
| `runExternalId` | TestRail `run.id` |
| `comment` | TestRail `result.comment` |
| `duration` | Parsed from TestRail `result.elapsed` (ms) |

**TestRail `status_id` → TestFiesta status mapping:**

| TestRail status_id | TestRail label | TestFiesta status |
|---|---|---|
| `1` | Passed | `"passed"` |
| `2` | Blocked | `"blocked"` |
| `3` | Untested | `"untested"` |
| `4` | Retest | `"untested"` |
| `5` | Failed | `"failed"` |
| Any other | — | `"untested"` |

**Elapsed time parsing:**

TestRail stores durations as strings. tacotruck converts them to milliseconds:

| TestRail elapsed | Milliseconds |
|---|---|
| `"30s"` | `30000` |
| `"1m"` | `60000` |
| `"1m 30s"` | `90000` |
| `"2m 15s"` | `135000` |
| `null` or absent | `0` |

---

## TestRail output (run:submit)

When `testrail run:submit` creates a run, the following TestRail entities are created.

### Suite (suite_mode 3 only)

`POST api/v2/add_suite/{project_id}`

```json
{
  "name": "<run_name>"
}
```

Not created for `suite_mode: 1` or `suite_mode: 2`.

### Sections

`POST api/v2/add_section/{project_id}` (one per `<testsuite>`)

```json
{
  "suite_id": <suite_id>,
  "name": "<section.name>"
}
```

One section is created per `<testsuite>` in the input XML. Sections are flat (no parent–child nesting).

### Cases

`POST api/v2/add_case/{section_id}` (one per `<testcase>`)

```json
{
  "title": "<testcase.name>",
  "type_id": 3,
  "template_id": 1,
  "priority_id": 2
}
```

| Field | Value | Meaning |
|---|---|---|
| `type_id` | `3` | Automated |
| `template_id` | `1` | Test Case (Text) |
| `priority_id` | `2` | Medium |

These values are hardcoded in `testrail-etl.ts`. They cannot be overridden via CLI flags.

### Run

`POST api/v2/add_run/{project_id}`

```json
{
  "suite_id": <suite_id>,
  "name": "<run_name>",
  "include_all": false,
  "case_ids": [<id1>, <id2>, ...]
}
```

The `case_ids` array contains all case IDs created in the step above.

### Results

`POST api/v2/add_results_for_cases/{run_id}`

```json
{
  "results": [
    {
      "case_id": <case_id>,
      "status_id": <1|2|3|4|5>,
      "comment": "<skipped message or empty>",
      "defects": "<failure message or empty>"
    }
  ]
}
```

One result object per test case. The `results` array is submitted in a single API call (not batched).

**Status mapping (`transformXmlData`):**

| JUnit test case condition | TestRail `status_id` |
|---|---|
| `failure` element present | `5` (Failed) |
| `error` element present | `5` (Failed) |
| `skipped` element present | `4` (Retest) |
| `status === "blocked"` | `2` (Blocked) |
| `status === "untested"` | `3` (Untested) |
| Otherwise (passed) | `1` (Passed) |

---

## Intermediate representation (IR)

Between the Extract and Load phases, data lives in an intermediate representation (IR). Understanding the IR is important for implementing custom transformations or debugging pipeline issues.

### IR structure

```typescript
{
  [entityType: string]: Record<string, unknown>[]
}
```

Example after extracting from TestRail:

```json
{
  "projects": [
    { "source_id": 1, "name": "Web App", "suite_mode": 1 },
    { "source_id": 2, "name": "Mobile App", "suite_mode": 3 }
  ],
  "suites": [
    { "source_id": 10, "name": "Regression", "project_id": 1 },
    { "source_id": 11, "name": "Smoke", "project_id": 1 }
  ],
  "cases": [
    { "source_id": 100, "name": "Login works", "parent_id": 10, "suite_id": 10, "priority": 2 }
  ]
}
```

### Special field names in the IR

| Field | Description |
|---|---|
| `source_id` | The primary identifier from the source system. Set by the `mapping` config (see `"id": "source_id"`). |
| `generated_source_id` | A UUID assigned by the parser when no `source_id` is present (JUnit/xUnit sources). Not used by API targets. |
| `custom_fields` | Object collecting all source fields that were not explicitly mapped. |

### IR after transform

After the transform stage, field names are mapped according to `mapping` config, ignore patterns are applied, and overrides are merged. The IR structure is otherwise unchanged.

### IR consumed by LoadToTarget

When `loadToTarget` is called, it receives a flat array of records for one entity type. Each record is wrapped in the `data_key` object and submitted to the target API. No further transformation occurs at this stage.

---

## Result object

`loadToTarget` returns a `LoadResult`:

```typescript
{
  success: boolean,        // true if totalFailed === 0
  totalProcessed: number,  // Count of records attempted
  totalFailed: number,     // Count of records that failed after all retries
  metadata: {
    errors: Array<{
      entityType: string,
      record: Record<string, unknown>,
      error: string,        // error.message
      timestamp: string     // ISO 8601
    }>
  }
}
```

The `testfiesta run:submit` command checks `metadata.errors.length === 0` and logs accordingly:

- No errors: `"[INFO] Run submitted successfully"`
- Errors: `"[ERROR] Errors occurred: [<error1>, <error2>]"` (formatted from the errors array)
