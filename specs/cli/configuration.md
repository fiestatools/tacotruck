<!-- spec-meta last-updated-sha: HEAD last-updated-pr: null last-updated-date: 2026-02-20 spec-version: 1.0 -->

# Configuration

tacotruck is configured through two orthogonal mechanisms: integration config files (describing API shape and mapping) and credentials (providing authentication secrets). These may be combined in a single `creds.json` file or split into separate files.

---

## Integration config files

Integration configs describe how to talk to an API: its base path, auth scheme, paging strategy, and the set of source/target endpoints for each entity type.

The bundled configs live in the `configs/` directory of the installed package:

| File | Integration |
|---|---|
| `configs/testfiesta.json` | TestFiesta |
| `configs/testrail.json` | TestRail |
| `configs/junit.json` | JUnit XML file source |
| `configs/jira.json` | JIRA |
| `configs/sample_config.json` | Annotated reference |

### Config file format (Zod schema)

The config is validated with Zod. The discriminated union key is `type`.

#### Common fields (all types)

| Field | Type | Required | Description |
|---|---|---|---|
| `name` | string | Yes | Logical name for the integration (e.g., `"testrail"`). |
| `type` | `"api"` \| `"junit"` \| `"json"` | Yes | Integration type. |
| `base_path` | string | No | Prefix appended to `base_url` before all API paths (e.g., `"index.php?"`). |
| `auth` | AuthConfig | No | Authentication scheme. |
| `requests_per_second` | number | No | Rate limit applied across all requests. |
| `sourceThrottle` | number | No | Throttle limit for source requests. |
| `sourceThrottleTime` | number | No | Throttle interval in ms for source requests. |
| `source` | Record\<string, EntityConfig\> | No | Map of entity type name to source endpoint definition. |
| `target` | Record\<string, EntityConfig\> | No | Map of entity type name to target endpoint definition. |
| `typeConfig` | TypeConfig | No | Source type mapping and denormalized key definitions. |

#### `type: "api"` additional fields

| Field | Type | Required | Description |
|---|---|---|---|
| `multi_target` | MultiTargetConfig | No | Configuration for the TestFiesta bulk submission endpoint. |

#### `type: "json"` and `type: "junit"` additional fields

| Field | Type | Required | Description |
|---|---|---|---|
| `file_path` | string | Yes | Path to the input file. |

---

### `AuthConfig`

| Field | Type | Required | Description |
|---|---|---|---|
| `type` | string | Yes | Auth type identifier (e.g., `"basic"`, `"bearer"`). |
| `location` | `"header"` \| `"query"` \| `"body"` | Yes | Where the auth credential is placed in the request. |
| `key` | string | No | Header/query parameter name (e.g., `"Authorization"`). |
| `payload` | string | No | Value template. May contain `{placeholder}` tokens that are substituted from credentials at runtime (e.g., `"Bearer {token}"`, `"Basic {base64Credentials}"`). |

**TestRail example:**
```json
{
  "type": "basic",
  "location": "header",
  "key": "Authorization",
  "payload": "Basic {base64Credentials}"
}
```

**TestFiesta example:**
```json
{
  "type": "bearer",
  "location": "header",
  "key": "Authorization",
  "payload": "Bearer {token}"
}
```

---

### `EntityConfig`

Each key under `source` or `target` maps to an entity config:

```json
{
  "endpoints": {
    "<operation>": EndpointDefinition
  }
}
```

#### `EndpointDefinition`

At least one of `path`, `bulk_path`, or `single_path` must be present (enforced by Zod).

| Field | Type | Required | Description |
|---|---|---|---|
| `path` | string | No | URL path template for GET or general use. May contain `{variable}` placeholders resolved from credentials or previously-fetched data. |
| `bulk_path` | string | No | URL path for bulk POST operations. |
| `single_path` | string | No | URL path for single-record POST operations. |
| `data_key` | string | No | Key under which data is wrapped in the POST body (e.g., `"entries"`). |
| `include_source` | boolean | No | If `true`, includes the source config name in the POST payload. |
| `update_key` | string | No | Field name used to match records for updates (default: `"id"`). |
| `payload_key` | string | No | Alternative key used to wrap the payload. |
| `throttle` | number | No | Per-endpoint throttle count. |
| `throttleTime` | number | No | Per-endpoint throttle interval in ms. |

**Standard operations for source endpoints:**

| Operation | Description |
|---|---|
| `index` | List all records (paginated) |
| `get` | Fetch a single record by ID |
| `delete` | Delete a record |

**Standard operations for target endpoints:**

| Operation | Description |
|---|---|
| `create` | Create a new record (uses `single_path` or `bulk_path`) |
| `update` | Update an existing record |
| `delete` | Delete a record |

---

### `MultiTargetConfig` (TestFiesta only)

Used when `type: "api"` and the integration is TestFiesta.

| Field | Type | Description |
|---|---|---|
| `path` | string | API path for bulk submission (e.g., `"/projects/{projectKey}/data"`). |
| `data_key` | string | Key wrapping the data array in the POST body (e.g., `"entries"`). |
| `include_source` | boolean | Whether to include source metadata in the payload. |

---

### URL path substitution

Path templates use `{variable}` syntax. At runtime, substitution variables are resolved from:

1. **Credentials** — any key in the credential object (e.g., `{token}`, `{base64Credentials}`, `{base_url}`, `{project_id}`, `{run_id}`, `{section_id}`).
2. **Previously-fetched data** — for source paths that depend on a parent entity, the path `{projects.id}` is resolved by expanding over all fetched project IDs (building one URL per project).

The `findSubstitutionKeys` utility extracts `{...}` tokens from a path string. `bracketSubstitution` performs the replacement.

Dependency chains are resolved by `buildDependencyChain`: if a path contains `{suites.id}`, then `suites` must be fetched before the current entity can be fetched.

---

### Paging

The TestRail config uses response-based paging:

```json
"paging": {
  "location": "response",
  "link_key": "_links.next"
}
```

- `location: "response"` means the next page URL is read from the response JSON at the key path specified by `link_key`.
- `_links.next` supports dot-notation for nested keys.

---

### Source entity options (legacy config fields)

The following fields appear in `sample_config.json` and control data extraction behavior in the classic ETL path:

| Field | Type | Description |
|---|---|---|
| `data_key` | string | Key in the API response where the array of records lives. |
| `target` | string | Internal type name to use instead of the endpoint name. |
| `mapping` | Record\<string, string\> | Field rename map applied during extraction. Maps source field names to internal field names. The special destination name `source_id` marks the primary identifier. All unmapped fields are stored under `custom_fields`. |
| `limit.type` | `"count"` \| `"match"` | Stop after N records (`count`) or when a record matching an ID is found (`match`). |
| `limit.value` | string \| number | For `count`: the number. For `match`: `"fieldName:value"` (e.g., `"id:1234"`). The matching record is excluded. |
| `limit.cutoff` | `"hard"` \| `"soft"` | Whether to discard records received after the limit (`hard`) or keep them (`soft`). |

---

## Credentials

Credentials provide authentication secrets to integration configs. They are never stored inside the integration config files.

### Credentials JSON format

```json
{
  "<integration_name>": {
    "<direction>": {
      "<credential_key_1>": "<value_1>",
      "<credential_key_2>": "<value_2>"
    }
  }
}
```

- `<integration_name>` must match the `name` field in the integration config (case-sensitive in file, uppercased for env vars).
- `<direction>` is `"source"` or `"target"`.

The `base_url` field is required in all credential blocks (enforced by Zod `credentialsSchema`).

**Full example for TestRail source + TestFiesta target:**

```json
{
  "testrail": {
    "source": {
      "base64Credentials": "dXNlcm5hbWU6cGFzc3dvcmQ=",
      "base_url": "https://example.testrail.io/"
    }
  },
  "testfiesta": {
    "target": {
      "token": "TF_99866737272422404.7f879e26fd76d09ce9ca263f9231f2cf",
      "base_url": "https://api.testfiesta.ai/"
    }
  }
}
```

### Per-integration credential keys

#### TestRail

| Key | Description |
|---|---|
| `base_url` | Full URL including trailing slash (e.g., `https://example.testrail.io/`). |
| `base64Credentials` | Base64-encoded `username:password` or `username:apikey`. Generate with `echo -n "user:pass" \| base64 -w0`. |
| `project_id` | (Optional) Pre-set project ID. Used by delete operations. |
| `run_id` | (Optional) Set dynamically during `submitTestRun` after the run is created. |
| `section_id` | (Optional) Set dynamically during case creation. |

#### TestFiesta

| Key | Description |
|---|---|
| `base_url` | TestFiesta API base URL (e.g., `https://api.testfiesta.ai/`). |
| `token` | Bearer token. Format: `TF_<numeric_part>.<hex_part>`. |
| `handle` | Organization handle/slug. Used in path substitution for `v1/{handle}` base path. |
| `projectKey` | Project key. Used in path substitution for `/projects/{projectKey}/data`. |
| `project_id` | Numeric project ID. Used by delete operations. |

#### JIRA

| Key | Description |
|---|---|
| `base_url` | JIRA instance URL (e.g., `https://example.atlassian.net/`). |
| `base64Credentials` | Base64-encoded `username:password_or_api_token`. |

### Environment variables

Instead of a credentials file, export JSON for each integration/direction pair:

```bash
export TESTRAIL_SOURCE_CREDENTIALS='{"base64Credentials":"dXNlcjpwYXNz","base_url":"https://example.testrail.io/"}'
export TESTFIESTA_TARGET_CREDENTIALS='{"token":"TF_xxx","base_url":"https://api.testfiesta.ai/"}'
```

Pattern: `<INTEGRATION_NAME_UPPERCASE>_<DIRECTION_UPPERCASE>_CREDENTIALS`

The value must be a JSON string. The JSON is parsed and the `.<direction>` key is extracted. For example, `TESTRAIL_SOURCE_CREDENTIALS` is parsed and `parsedEnv["source"]` is extracted.

Note: The env var JSON only needs to contain the direction-specific credential object, not a full nested structure.

### Credential storage security

- Credentials are never written to disk by tacotruck itself.
- The `creds.json` file (referenced in examples) is user-managed.
- The installed package ships with a `creds.json` file at the repo root only for development use. Do not commit credentials to version control.

---

## Ignore patterns

Source data can be filtered before transformation by providing an ignore config JSON. Pass it with `-I <path>` on the `migrate` command (legacy interface).

### Ignore config format

```json
{
  "<entity_type>": {
    "<field_name>": ["<regex_pattern_1>", "<regex_pattern_2>"]
  }
}
```

Each regex in the array is tested against the field value. If any regex matches, the record is dropped from the pipeline. Patterns are standard JavaScript regular expressions (without the `/` delimiters).

**Example — ignore projects with "Example" in the name:**

```json
{
  "projects": {
    "name": ["[Ee]xample"]
  }
}
```

---

## Data overrides

Overrides merge additional fields into each record of a given type after all field mappings are applied. Provided as a JSON string via `-o` on the `migrate` command (legacy interface).

### Override format

```json
{
  "<entity_type>": {
    "<field_to_add_or_overwrite>": "<value>"
  }
}
```

The override object is merged into each record in the specified entity type array. Existing keys are overwritten; new keys are added.

**Example — tag all runs with a runner name:**

```json
{
  "runs": {
    "runner": "jenkins-runner-1"
  }
}
```

---

## Git info embedding

Unless `--no-git` is passed (or `noGit: true` in the API), `loadConfig` attempts to extract git context from `.git/config`, `.git/HEAD`, and `.git/logs/HEAD`. If found, it attaches a `gitInfo` object to the config:

```typescript
{
  repo: string,    // URL from git remote
  branch: string,  // Current branch name
  sha: string      // Last commit SHA from the HEAD log
}
```

This information is available downstream to integrations that track source control context.

---

## Custom integrations

Create a JSON file conforming to the integration config schema, then pass its path as the `--source` or `--target` argument:

```bash
tacotruck migrate -s ./my-source.json -t testfiesta -c ./creds.json
```

The config file is loaded via the same `loadConfig` path as built-in integrations. All features (paging, auth, mapping, limits) are available in custom configs.
