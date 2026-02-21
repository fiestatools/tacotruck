<!-- spec-meta last-updated-sha: HEAD last-updated-pr: null last-updated-date: 2026-02-20 spec-version: 1.0 -->

# ETL Pipeline Architecture

The ETL pipeline is implemented in `src/controllers/etl-base-v2.ts` (`EtlBaseV2`) with integration-specific subclasses in `testfiesta-etl.ts` and `testrail-etl.ts`. The pipeline has three stages: **Extract**, **Transform**, and **Load**.

---

## Overview

```
┌────────────────────────────────────────────────────────────────────────────────────┐
│                                EtlBaseV2                                           │
│                                                                                    │
│  ┌──────────────────┐    ┌──────────────────┐    ┌──────────────────────────────┐  │
│  │     EXTRACT      │    │    TRANSFORM     │    │           LOAD               │  │
│  │                  │──▶ │                  │──▶ │                              │  │
│  │  extractFromSource│   │ transform(data)  │   │  loadToTarget(type, data)    │  │
│  │  paginateAll()   │    │                  │    │  batch() + retry()           │  │
│  │  buildDepChain() │    │  field mapping   │    │  API submission              │  │
│  └──────────────────┘    │  ignoreFilters   │    └──────────────────────────────┘  │
│                          │  overrides       │                                      │
│                          └──────────────────┘                                      │
└────────────────────────────────────────────────────────────────────────────────────┘
```

---

## ETL options

Options are passed to the ETL constructor and control cross-cutting behavior. The Zod schema `EtlOptionsSchema` validates them at instantiation:

| Option | Type | Default | Description |
|---|---|---|---|
| `retryAttempts` | number | `3` | Number of retry attempts for failed API requests. |
| `retryDelay` | number | `1000` | Delay in ms between retries (used as a base for exponential backoff if `retryWithExponentialBackoff` is set). |
| `timeout` | number | `30000` | HTTP request timeout in ms. |
| `enablePerformanceMonitoring` | boolean | `false` | Track request timing and emit performance metrics. |
| `strictMode` | boolean | `false` | If `true`, any single entity failure aborts the entire pipeline. If `false`, failures are collected and the pipeline continues. |

### ETL options by caller

| Caller context | `retryAttempts` | `timeout` | Notes |
|---|---|---|---|
| `testfiesta run:submit` | `1` | `2000` ms | Fast-fail for CI submissions |
| `testfiesta project:delete` | `3` | `30000` ms | Standard |
| `testrail run:submit` | `3` | `30000` ms | Standard |
| `testrail project:delete` | `3` | `30000` ms | Standard |
| `migrate` command | From credentials file | From credentials file | Configurable |

---

## Stage 1: Extract

### `extractFromSource(config, entityTypes)`

Dispatches to individual entity extractors based on `entityTypes`. If `entityTypes` is `undefined`, all entity types defined in `config.source` are extracted.

For each entity type:

1. **Dependency resolution** — `buildDependencyChain(entityType, config)` determines which entity types must be fetched first (because the endpoint URL contains `{otherType.id}` substitution tokens). For example, TestRail `cases` requires `suites` because the cases endpoint is per-suite.

2. **Pagination** — `paginateAll(endpoint, config, entityType, previousData?)` fetches all pages of an entity type. The paging algorithm is:

   a. Construct the initial URL by substituting path variables from the credentials and (for denormalized endpoints) from previously-fetched parent data.

   b. Make a GET request.

   c. Check the response for the `link_key` path (e.g., `_links.next`). If a next-page URL is present, request it and append results.

   d. Continue until no next-page URL is found.

   e. Apply `limit` config:
      - `type: "count"` — stop after `value` records total. If `cutoff: "hard"`, discard any records received after the count is reached. If `cutoff: "soft"`, keep them.
      - `type: "match"` — stop when a record is found where `record[fieldName] == matchValue`. The matching record is not included.

3. **Denormalized endpoints** — when `config.denormalized_keys` is set (TestRail `cases` by `suites` by `projects.id`), the extractor expands the endpoint for each ID from the required parent collection and merges results.

4. Returns an `IntermediateRepresentation` (IR) object:

```typescript
{
  [entityType: string]: Record<string, unknown>[]
}
```

---

## Stage 2: Transform

### `transform(ir, mapping, ignorePatterns?, overrides?)`

The transform stage operates on the IR produced by Extract.

#### Field mapping

For each entity type in the IR, the mapping config specifies field renames:

```json
{
  "id": "source_id",
  "name": "name",
  "suite_id": "parent_id"
}
```

- Keys not in the mapping are not dropped — they are kept as-is (or collected under `custom_fields` depending on the implementation path).
- The special destination name `source_id` designates the primary identifier field.

#### Ignore filters

Ignore patterns are applied after field mapping. For each entity type and field combination, all records where `record[field]` matches any pattern regex are removed from the IR.

Pattern application is case-insensitive.

#### Overrides

Override objects are merged into each record after filtering. Existing keys are overwritten; new keys are added. Overrides have the highest precedence in the transform stage.

---

## Stage 3: Load

### `loadToTarget(entityType, data, operation?, endpointHints?)`

Sends transformed data to the target API.

#### Operation types

| Operation | Description |
|---|---|
| `"create"` (default) | Submit new records |
| `"update"` | Update existing records; uses `update_key` to identify records |
| `"delete"` | Delete records |

#### Bulk vs. single submission

The loader checks whether the endpoint config has a `bulk_path`:

- **Bulk path present** — sends a single POST with all records wrapped under `data_key`:
  ```json
  { "entries": [ {...}, {...} ] }
  ```
- **Single path only** — sends one POST per record.

If `include_source` is `true` in the endpoint config, a `source` key with the integration config name is added to each record.

#### Batch processing

Bulk submissions are divided into batches of configurable size (see `BatchProcessor`). The `BatchProcessor` is constructed with:

- `size` — number of records per batch (default: `50`)
- `concurrency` — number of concurrent batch requests (default: `5`)
- `throttle` — number of requests allowed per interval
- `throttleTime` — interval in ms for the throttle window

The actual execution flow:

1. `processBatch(batch)` → `loadBatch(batch)` → HTTP POST → response handler
2. If the request fails and `retryAttempts > 0`, the batch is retried after `retryDelay` ms.
3. In `strictMode`, the first batch failure aborts all remaining batches.
4. Outside `strictMode`, failed batches are collected into `metadata.errors`.

After all batches complete, a load result is returned:

```typescript
{
  success: boolean,
  totalProcessed: number,
  totalFailed: number,
  metadata: {
    errors: Array<{ entityType, record, error, timestamp }>
  }
}
```

---

## Retry logic

All HTTP requests made through `ApiClient` are wrapped in a retry loop governed by these parameters:

| Parameter | Source |
|---|---|
| `retryAttempts` | ETL options (per command) |
| `retryDelay` | ETL options (per command) |

The retry algorithm:

1. Attempt the request.
2. On failure: if `attempt < retryAttempts`, wait `retryDelay` ms and retry.
3. If all attempts are exhausted, throw the last error.

HTTP 429 (Too Many Requests) responses are treated as rate-limit errors and are retried even if they are not classified as transient by the default error classifier. Rate limit detection:

- Status code `429`
- Response header `Retry-After` — if present, the value is used as the delay (in seconds, converted to ms) for the next retry instead of `retryDelay`.

---

## Error handling

### During Extract

- Network errors are retried up to `retryAttempts` times.
- After exhausting retries, the extractor logs the error and either throws (if no data was collected yet) or returns partial data.
- Individual page fetch failures do not abort paging in non-strict mode; they are logged as warnings.

### During Load

- Each record failure is caught individually and added to `metadata.errors`:
  ```typescript
  {
    entityType: string,
    record: Record<string, unknown>,
    error: string,  // error.message
    timestamp: string  // ISO 8601
  }
  ```
- In `strictMode: true`, the first error immediately rejects the entire load promise.
- In `strictMode: false` (default), the pipeline logs each error and continues.

### Error messages (exact strings)

| Condition | Log message |
|---|---|
| Config file not found | `"Config file not found: <path>"` |
| Credentials env var missing | `"No credentials file provided and no environment variable <VAR_NAME> found"` |
| Invalid JSON in credentials file | Surfaces native `JSON.parse` error message |
| API 4xx (non-429) | `"HTTP error: <statusCode> <statusText> - <responseBody>"` |
| API 5xx | `"HTTP error: <statusCode> <statusText> - <responseBody>"` |
| Max retries exceeded | `"Max retries (<n>) exceeded for <url>"` |

---

## Performance monitoring

When `enablePerformanceMonitoring: true`, each API call's duration is measured using `performance.now()` and emitted to the logger:

```json
{
  "type": "performance",
  "method": "GET",
  "url": "https://...",
  "duration_ms": 243
}
```

This log line is emitted at `DEBUG` level. It does not affect pipeline behavior.

---

## Data flow diagram (TestRail → TestFiesta migrate)

```
TestRail API                          ETL Pipeline                        TestFiesta API
────────────                          ────────────                        ──────────────
GET /get_projects
                ──────────────────────▶ Extract projects []
GET /get_suites?project_id=N
                ──────────────────────▶ Extract suites []
GET /get_cases?suite_id=M
                ──────────────────────▶ Extract cases []
GET /get_runs?project_id=N
                ──────────────────────▶ Extract runs []
GET /get_results_for_run?run_id=R
                ──────────────────────▶ Extract results []
                                               │
                                        Transform (field mapping,
                                        ignores, overrides)
                                               │
                                        Build bulk payloads
                                               │
                              POST /v1/{handle}/projects/data ──────────────────────▶
                              POST /api/v1/data (suites)      ──────────────────────▶
                              POST /api/v1/data (cases)       ──────────────────────▶
                              POST /api/v1/data (runs)        ──────────────────────▶
                              POST /api/v1/data (results)     ──────────────────────▶
```

---

## `EtlBaseV2` class interface

```typescript
class EtlBaseV2 {
  constructor(
    sourceConfig: IntegrationConfig,
    targetConfig: IntegrationConfig,
    sourceCredentials: Credentials,
    targetCredentials: Credentials,
    options: EtlOptions
  )

  static fromConfig(
    sourceName: string,
    targetName: string,
    credentialsPath: string | undefined,
    options: EtlOptions
  ): Promise<EtlBaseV2>

  // Run the full pipeline
  async run(
    entityTypes?: string[],
    ignorePatterns?: IgnoreConfig,
    overrides?: OverrideConfig
  ): Promise<LoadResult>

  // Stage methods (called by run())
  protected async extractFromSource(config: IntegrationConfig, entityTypes?: string[]): Promise<IR>
  protected async transform(ir: IR, mapping?: MappingConfig, ignorePatterns?: IgnoreConfig, overrides?: OverrideConfig): Promise<IR>
  async loadToTarget(entityType: string, data: Record[], operation?: string, hints?: object): Promise<LoadResult>

  // Utility
  protected buildDependencyChain(entityType: string, config: IntegrationConfig): string[]
}
```

---

## `BatchProcessor`

Located in `src/utils/batch-processor.ts`.

```typescript
class BatchProcessor<T> {
  constructor(options: {
    size: number,
    concurrency: number,
    throttle?: number,
    throttleTime?: number
  })

  async process(
    items: T[],
    handler: (batch: T[]) => Promise<void>
  ): Promise<void>
}
```

Uses `p-limit` for concurrency control and a custom throttle token-bucket for rate limiting. Processing is strictly sequential within a batch (all items in a batch are included in the same API call), but multiple batches may run concurrently up to the `concurrency` limit.
