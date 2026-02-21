<!-- spec-meta last-updated-sha: HEAD last-updated-pr: null last-updated-date: 2026-02-20 spec-version: 1.0 -->

# CLI Commands

tacotruck is invoked as `tacotruck <command> [subcommand] [flags]`. On startup the program renders an ASCII art banner and then dispatches to the registered command.

Every command accepts `-v, --verbose` to enable `DEBUG`-level structured logging. Without it the log level is `INFO`.

On fatal error the process exits with code `1`. On success it exits with code `0`.

---

## Top-level commands

| Command | Description |
|---|---|
| `migrate` | Pull from a source integration and push to a target integration |
| `testfiesta` | TestFiesta-specific sub-commands |
| `testrail` | TestRail-specific sub-commands |
| `--version` | Print the package version and exit |
| `--help` | Print usage and exit |

---

## `migrate`

Pull data from one or more sources and push it to one or more targets using the ETL pipeline.

### Syntax

```
tacotruck migrate -s <source> -t <target> [options]
```

### Flags

| Flag | Short | Required | Type | Description |
|---|---|---|---|---|
| `--source` | `-s` | Yes | string | Source integration name (`testrail`, `testfiesta`, `jira`, `junit`) or path to a custom JSON config file. For file-type sources use `junit:<path>` syntax. Multiple sources are comma-separated. |
| `--target` | `-t` | Yes | string | Target integration name or path to custom JSON config. Multiple targets are comma-separated. |
| `--credentials` | `-c` | No | path | Path to a JSON credentials file. If omitted, environment variables `<SOURCE_NAME>_SOURCE_CREDENTIALS` and `<TARGET_NAME>_TARGET_CREDENTIALS` are used. |
| `--verbose` | `-v` | No | boolean | Enable debug logging (default: `false`) |

### Flags present in the README but wired at the legacy layer

The following flags are documented in `README.md` and the original interface but are handled by the legacy ETL path rather than the current `migrate` command in `src/cli/commands/migrate.ts`:

| Flag | Description |
|---|---|
| `--ignore` / `-I` | Path to a JSON file specifying records to ignore by data type and regex |
| `--overrides` / `-o` | JSON string to merge into each output record |
| `--data-types` / `-d` | Comma-separated list of entity types to include (limits the default "all") |
| `--offset` | Paging offset |
| `--limit` | Paging limit |
| `--count` | Maximum record count |
| `--no-git` | Do not embed git repo metadata in output |

### Credential resolution

When `--credentials` is not provided, the command logs a warning and looks for environment variables in the format:

```
<INTEGRATION_NAME_UPPERCASE>_SOURCE_CREDENTIALS
<INTEGRATION_NAME_UPPERCASE>_TARGET_CREDENTIALS
```

Example for `testrail` → `testfiesta`:

```bash
export TESTRAIL_SOURCE_CREDENTIALS='{"base64Credentials":"dXNlcjpwYXNz","base_url":"https://example.testrail.io/"}'
export TESTFIESTA_TARGET_CREDENTIALS='{"token":"TF_xxx","base_url":"https://api.testfiesta.ai/"}'
```

When the credentials file is provided via `-c`, the file must exist and be valid JSON; otherwise the process exits with code `1`.

### Config loading order

For each source/target name, `loadConfig` searches for the config JSON in this order:

1. `$PACKAGE_ROOT/configs/<name>.json` (using the `packageRoot` stored in async-local-storage on startup)
2. `./configs/<name>.json` (relative to the current working directory)
3. `./<name>.json`

If none is found, the process logs an error and exits with code `1`.

### Exit codes

| Code | Condition |
|---|---|
| `0` | Pipeline completed without fatal errors |
| `1` | Config not found, credentials file missing, JSON parse failure, or pipeline threw an unhandled exception |

### Examples

```bash
# Migrate all entities from TestRail to TestFiesta using env var credentials
tacotruck migrate -s testrail -t testfiesta

# Use a credentials file and enable verbose output
tacotruck migrate -s testrail -t testfiesta -c ./creds.json -v

# Migrate from two sources simultaneously
tacotruck migrate -s testrail,jira -t testfiesta -c ./creds.json
```

---

## `testfiesta` sub-commands

These commands interact directly with the TestFiesta API.

### `testfiesta run:submit`

Parse a local test result file (JSON or JUnit XML) and submit it as a test run to TestFiesta.

#### Syntax

```
tacotruck testfiesta run:submit -d <path> -t <token> -h <org> -p <project> [options]
```

#### Flags

| Flag | Short | Required | Type | Description |
|---|---|---|---|---|
| `--data` | `-d` | Yes | path | Path to the test result file. Accepts `.json` (raw JSON) or `.xml` (JUnit XML). |
| `--token` | `-t` | Yes | string | TestFiesta API bearer token. Format: `TF_XXXXXXXX.YYYYYYYY` |
| `--organization` | `-h` | Yes | string | Organization handle (slug) in TestFiesta. |
| `--project` | `-p` | Yes | string | Project key in TestFiesta. |
| `--verbose` | `-v` | No | boolean | Enable debug logging. |

#### Behavior

1. Initializes the logger.
2. Calls `loadRunData(args.data)` to parse the file. For `.xml` files, uses `JunitXmlParser`. For `.json` files, parses raw JSON.
3. Instantiates `TestFiestaETL.fromConfig` with the provided credentials and hard-coded ETL options:
   - `baseUrl`: `http://localhost:5000` (note: this is the default and may differ in production)
   - `retryAttempts`: `1`
   - `retryDelay`: `500` ms
   - `timeout`: `2000` ms
   - `enablePerformanceMonitoring`: `true`
   - `strictMode`: `false`
4. Calls `testFiestaETL.load(runData)`, which runs the full ETL pipeline.
5. If `loadResult.metadata.errors.length === 0`, logs success. Otherwise logs the errors.

#### Exit codes

| Code | Condition |
|---|---|
| `0` | Run submitted without errors |
| `1` | File not found, parse error, API error, or uncaught exception |

#### Example

```bash
tacotruck testfiesta run:submit \
  -d ./results/junit.xml \
  -t TF_99866737272422404.7f879e26fd76d09ce9ca263f9231f2cf \
  -h acme-corp \
  -p WEB-APP
```

---

### `testfiesta project:create`

Create a new project in TestFiesta.

#### Syntax

```
tacotruck testfiesta project:create -n <name> -k <key> -t <token> -h <org> [options]
```

#### Flags

| Flag | Short | Required | Type | Description |
|---|---|---|---|---|
| `--name` | `-n` | Yes | string | Project display name. |
| `--key` | `-k` | Yes | string | Project key (short identifier). |
| `--token` | `-t` | Yes | string | TestFiesta API bearer token. |
| `--organization` | `-h` | Yes | string | Organization handle. |
| `--verbose` | `-v` | No | boolean | Enable debug logging. |

#### Behavior

Logs the intent to create a project. The implementation currently logs the action at DEBUG level and calls `runCreateProject` — full API integration is a stub in the current codebase.

---

### `testfiesta project:delete`

Delete a project from TestFiesta by its project ID.

#### Syntax

```
tacotruck testfiesta project:delete -i <id> -t <token> -h <org> [options]
```

#### Flags

| Flag | Short | Required | Type | Description |
|---|---|---|---|---|
| `--project-id` | `-i` | Yes | string | TestFiesta project ID to delete. |
| `--token` | `-t` | Yes | string | TestFiesta API bearer token. |
| `--organization` | `-h` | Yes | string | Organization handle. |
| `--verbose` | `-v` | No | boolean | Enable debug logging. |

#### Behavior

1. Instantiates `TestFiestaETL.fromConfig` with credentials `{ token, handle: organization, project_id: projectId }` and ETL options:
   - `retryAttempts`: `3`
   - `timeout`: `30000` ms
   - `enablePerformanceMonitoring`: `false`
   - `strictMode`: `false`
2. Calls `testFiestaETL.deleteProject()`, which calls `loadToTarget('projects', {}, 'delete')` on the configured endpoint.
3. Logs success or error and exits.

#### Exit codes

| Code | Condition |
|---|---|
| `0` | Project deleted successfully |
| `1` | API error or unexpected exception |

---

## `testrail` sub-commands

These commands interact directly with the TestRail API.

### `testrail run:submit`

Parse a local test result file and submit it as a new test run in TestRail, creating all necessary sections, cases, and results.

#### Syntax

```
tacotruck testrail run:submit -d <path> -e <email> -p <password> -u <url> -i <project-id> -n <run-name> [options]
```

#### Flags

| Flag | Short | Required | Type | Description |
|---|---|---|---|---|
| `--data` | `-d` | Yes | path | Path to the test result file (`.json` or `.xml`). |
| `--email` | `-e` | Yes | string | TestRail username / email address. |
| `--password` | `-p` | Yes | string | TestRail password or API key. |
| `--url` | `-u` | Yes | string | TestRail instance base URL (e.g., `https://example.testrail.io`). |
| `--project-id` | `-i` | Yes | string | TestRail project numeric ID. |
| `--run-name` | `-n` | Yes | string | Display name for the created test run. |
| `--x` | `-D` | No | string | Description text for the test run. |
| `--suite-id` | `-s` | No | string | Suite ID (required for projects with `suite_mode: 3`). |
| `--include-all` | `-a` | No | boolean | Include all test cases in the run (mutually exclusive with `--case-ids`). |
| `--case-ids` | `-c` | No | string | Comma-separated case IDs to include (only when `--include-all` is not set). |
| `--verbose` | `-v` | No | boolean | Enable debug logging. |

#### Behavior

1. Loads and parses the data file via `loadRunData`.
2. Calls `transformXmlData(runData)` to convert parsed JUnit structure to the TestRail-compatible format (sections, cases, results).
3. Constructs `base64Credentials` from `Buffer.from(`${email}:${password}`).toString('base64')`.
4. Instantiates `TestRailETL.fromConfig` with credentials and ETL options:
   - `retryAttempts`: `3`
   - `timeout`: `30000` ms
   - `enablePerformanceMonitoring`: `false`
   - `strictMode`: `false`
5. Calls `testRailETL.submitTestRun(transformedData, { project_id }, runName)`, which:
   a. Fetches the project to determine `suite_mode`.
   b. If `suite_mode === 3`, creates a new test suite.
   c. Batch-creates sections (up to 5 concurrent, throttled to 5/s with a 1000ms interval).
   d. Batch-creates test cases (up to 5 concurrent), building a `casesIdMap`.
   e. Creates the test run with the resulting `case_ids` array.
   f. Posts all test results to `add_results_for_cases/{run_id}`.

#### Exit codes

| Code | Condition |
|---|---|
| `0` | Run submitted successfully |
| `1` | Any step fails |

---

### `testrail project:create`

Create a new project in TestRail with an interactive or flag-driven suite mode selection.

#### Syntax

```
tacotruck testrail project:create -n <name> -e <email> -p <password> -u <url> [options]
```

#### Flags

| Flag | Short | Required | Type | Description |
|---|---|---|---|---|
| `--name` | `-n` | Yes | string | Project display name. |
| `--email` | `-e` | Yes | string | TestRail username / email. |
| `--password` | `-p` | Yes | string | TestRail password or API key. |
| `--url` | `-u` | Yes | string | TestRail instance base URL. |
| `--suite-mode` | `-s` | No | integer (1\|2\|3) | Repository structure. If omitted, an interactive prompt is shown. |
| `--verbose` | `-v` | No | boolean | Enable debug logging. |

#### Suite mode values

| Value | Meaning |
|---|---|
| `1` | Single repository for all cases (default recommendation) |
| `2` | Single repository with baseline support |
| `3` | Multiple test suites |

If `--suite-mode` is not provided on the command line, the CLI presents an interactive `@inquirer/prompts` select list. Choosing any value other than `1`, `2`, or `3` throws an error at parse time.

#### Behavior

Calls `testRailETL.submitProjects({ name, suite_mode })`, which posts to `POST api/v2/add_project`.

#### Exit codes

| Code | Condition |
|---|---|
| `0` | Project created successfully |
| `1` | API error |

---

### `testrail project:delete`

Delete a project from TestRail by its numeric project ID.

#### Syntax

```
tacotruck testrail project:delete -i <id> -e <email> -p <password> -u <url> [options]
```

#### Flags

| Flag | Short | Required | Type | Description |
|---|---|---|---|---|
| `--project-id` | `-i` | Yes | string | TestRail project ID to delete. |
| `--email` | `-e` | Yes | string | TestRail username / email. |
| `--password` | `-p` | Yes | string | TestRail password or API key. |
| `--url` | `-u` | Yes | string | TestRail instance base URL. |
| `--force` | `-f` | No | boolean | Skip the interactive confirmation prompt. |
| `--verbose` | `-v` | No | boolean | Enable debug logging. |

#### Behavior

1. Without `--force`, displays an `@clack/prompts` confirmation dialog with `initialValue: false`. If the user cancels or answers no, the command exits without doing anything.
2. With confirmation, instantiates `TestRailETL` with credentials `{ base64Credentials, base_url, project_id }`.
3. Calls `testRailETL.deleteProjects()`, which calls `DELETE api/v2/delete_project/{project_id}`.
4. Logs success/error.

#### Exit codes

| Code | Condition |
|---|---|
| `0` | Deleted or user-cancelled |
| `1` | API error |
