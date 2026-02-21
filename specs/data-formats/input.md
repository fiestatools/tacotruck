<!-- spec-meta last-updated-sha: HEAD last-updated-pr: null last-updated-date: 2026-02-20 spec-version: 1.0 -->

# Input Data Formats

tacotruck accepts test result data from three sources: JUnit XML files, raw JSON files, and live API sources (TestRail, TestFiesta, JIRA). This document covers the file-based formats parsed by `loadRunData` and the two XML parsing paths.

---

## File format detection

`loadRunData(filePath)` dispatches based on file extension (case-insensitive):

| Extension | Parser used |
|---|---|
| `.xml` | `JunitXmlParser` |
| `.json` | `JSON.parse` (raw) |
| Anything else | Error: `"Unsupported file format: <ext>. Only JSON and XML files are supported."` |

The path is resolved relative to `process.cwd()` using `path.resolve`. If the file does not exist at the resolved path, the error is: `"Data file not found: <resolvedPath>"`.

---

## JUnit XML format

### Parser

Class: `JunitXmlParser` (`src/utils/junit-xml-parser.ts`)

Uses the `fast-xml-parser` library with these options:

| Option | Value |
|---|---|
| `ignoreAttributes` | `false` |
| `attributeNamePrefix` | `""` (none) |
| `parseAttributeValue` | `true` |
| `textNodeName` | `"_text"` |

### Supported XML structures

#### Structure 1: `<testsuites>` root (multi-suite)

```xml
<?xml version="1.0" encoding="UTF-8"?>
<testsuites name="My Test Suite" tests="3" failures="1" errors="0" skipped="1" time="1.234">
  <testsuite name="com.example.MyTests" tests="2" failures="1" errors="0" skipped="0" time="0.8" timestamp="2026-02-20T12:00:00">
    <testcase name="testLogin" classname="com.example.MyTests" time="0.3">
      <!-- passed — no child elements -->
    </testcase>
    <testcase name="testCheckout" classname="com.example.MyTests" time="0.5">
      <failure message="Expected 200 but got 500" type="AssertionError">
        Full stack trace here
      </failure>
    </testcase>
  </testsuite>
  <testsuite name="com.example.OtherTests" tests="1" failures="0" errors="0" skipped="1" time="0.434">
    <testcase name="testReport" classname="com.example.OtherTests" time="0.1">
      <skipped message="Not implemented yet"/>
    </testcase>
  </testsuite>
</testsuites>
```

#### Structure 2: Single `<testsuite>` root

```xml
<?xml version="1.0" encoding="UTF-8"?>
<testsuite name="com.example.MyTests" tests="2" failures="0" errors="0" skipped="0" time="0.5">
  <testcase name="testA" classname="com.example.MyTests" time="0.2"/>
  <testcase name="testB" classname="com.example.MyTests" time="0.3"/>
</testsuite>
```

### XML element reference

#### `<testsuites>` (optional root)

| Attribute | Type | Description |
|---|---|---|
| `name` | string | Optional suite name. Used as the run name in output. |
| `tests` | integer | Total test count (informational; recomputed from children). |
| `failures` | integer | Total failure count (informational). |
| `errors` | integer | Total error count (informational). |
| `skipped` | integer | Total skipped count (informational). |
| `time` | float | Total wall-clock time in seconds. |

#### `<testsuite>`

| Attribute | Type | Description |
|---|---|---|
| `name` | string | Suite name. Mapped to `section.name`. Used for section-to-case matching. |
| `tests` | integer | Test count for this suite. |
| `failures` | integer | Failure count. |
| `errors` | integer | Error count. |
| `skipped` | integer | Skipped count. |
| `time` | float | Wall-clock time in seconds. |
| `timestamp` | string | ISO 8601 datetime. Stored as `section.timestamp` / `created_at`. |
| `file` | string | File path. Stored as `section.file`. |

#### `<testcase>`

| Attribute | Type | Description |
|---|---|---|
| `name` | string | Test name. Mapped to `testcase.name` and used as the case title. |
| `classname` | string | Class/module name. Used for section matching (see below). |
| `time` | float | Test duration in seconds. Stored as `testcase.time`. |

#### `<failure>` (child of `<testcase>`)

Sets `testcase.status = "failed"`.

| Attribute | Type | Description |
|---|---|---|
| `message` | string | Short failure message. Stored as `failure.message` and used as `defects` in TestRail output. |
| `type` | string | Exception class name. Stored as `failure.type`. |
| Text content | string | Full stack trace or failure details. Stored as `failure._text`. |

#### `<error>` (child of `<testcase>`)

Sets `testcase.status = "error"`.

Same attributes as `<failure>`. Stored in `testcase.error` instead of `testcase.failure`. Also causes `status_id = 5` (Failed) in TestRail transform.

#### `<skipped>` (child of `<testcase>`)

Sets `testcase.status = "skipped"`.

| Attribute | Type | Description |
|---|---|---|
| `message` | string | Skip reason. Stored as `skipped.message`. |

### Status determination (JunitXmlParser)

Status is determined purely by the presence of child elements on `<testcase>`:

| Child element | Status assigned |
|---|---|
| `<failure>` present | `"failed"` |
| `<error>` present | `"error"` |
| `<skipped>` present | `"skipped"` |
| None of the above | `"passed"` |

Note: The presence check is evaluated in order (`failure` → `error` → `skipped`). If a test case somehow has both `<failure>` and `<error>`, `failure` wins.

### Parser output structure (`JunitParserResult`)

`JunitXmlParser.build()` returns:

```typescript
{
  root: RootSuite | null,   // Aggregated suite-level metadata
  section: TestSuite[],     // One entry per <testsuite>
  testcase: TestCase[],     // Flat array of all test cases across all suites
}
```

The `root` key is also set under the `xmlToJsMap.suites` key (default: `"root"`), `section` under `xmlToJsMap.suite` (default: `"section"`), and `testcase` under `xmlToJsMap.testcase` (default: `"testcase"`). With default settings, these are redundant (same keys).

#### `RootSuite`

```typescript
{
  name?: string,      // Always "root" when synthesized
  tests?: number,     // Sum of all suite tests counts
  errors?: number,    // Sum of all suite error counts
  failures?: number,  // Sum of all suite failure counts
  skipped?: number,   // Sum of all suite skipped counts
  time?: number,      // Sum of all suite times
  timestamp?: string, // Not set on root (undefined)
}
```

If neither `<testsuites>` nor `<testsuite>` is present at the root, `root` is `{}` (empty object).

#### `TestSuite`

```typescript
{
  name: string,       // From <testsuite name="...">. Empty string if absent.
  tests: number,
  errors: number,
  failures: number,
  skipped: number,
  time: number,
  timestamp?: string,
  file?: string,
  testcases: TestCase[]
}
```

#### `TestCase`

```typescript
{
  name: string,       // From <testcase name="...">. Empty string if absent.
  classname: string,  // From <testcase classname="...">. Empty string if absent.
  time: number,       // Seconds. 0 if absent.
  status: string,     // "passed" | "failed" | "error" | "skipped"
  failure?: {
    message?: string,
    type?: string,
    _text?: string    // Raw text content of the <failure> element
  },
  error?: {
    message?: string,
    type?: string,
    _text?: string
  },
  skipped?: {
    message?: string
  }
}
```

---

## xUnit XML format (legacy parser)

Class: `XUnitParser` (v1: `src/utils/xunit-parser.ts`, v2: `src/utils/xunit-parser-v2.ts`)

This is an older parsing path used during migrations (the `testrail run:submit` command uses `JunitXmlParser` then `transformXmlData`; `XUnitParser` is available for programmatic use).

`XUnitParser` uses the same `fast-xml-parser` options plus a `collapse()` normalization pass that flattens attribute objects (`$` keys are merged into the parent object).

### Mappings applied by `XUnitParser`

**Test run (from `<testsuites>` root):**

| Source attribute | Internal field |
|---|---|
| `name` | `name` |

**Test suite (from `<testsuite>`):**

| Source attribute | Internal field |
|---|---|
| `name` | `name` |
| `timestamp` | `created_at` |

**Test case (from `<testcase>`):**

| Source attribute | Internal field |
|---|---|
| `name` | `name` |
| `error` | `error` |
| `failure` | `failure` |
| `system-out` | `system_out` |
| `skipped` | `skipped` |

Unmapped attributes are dropped.

### XUnitParser output

`parseContent()` returns:

```typescript
{
  suites: Array<{
    name: string,
    created_at?: string,
    generated_source_id?: string  // UUID, set if no source_id
  }>,
  executions: Array<{
    name: string,
    error?: { message?: string, $t?: string },
    failure?: { message?: string },
    system_out?: string,
    skipped?: string | object,
    test_suite_id: string,   // UUID of the parent suite
    test_run_id: string      // UUID of the run
  }>,
  runs: Array<{
    name: string,
    generated_source_id: string  // UUID
  }>,
  testrail?: {
    suite: { name: string, description: string, created_at: string },
    sections: Array<{ id: string, name: string, parentId: null, created_at: string }>,
    cases: Array<{ id: string, section_id: string, title: string, custom_test_case_id: string }>,
    results: Array<{ case_id: string, status_id: number, comment: string, defects: string }>
  }
}
```

---

## `transformXmlData` (XML → TestRail format)

This function (`src/utils/xml-transform.ts`) converts `JunitXmlParser` output into the format needed for `testrail run:submit`. It is separate from `XUnitParser`.

### Section-to-case matching

For each `testcase`, the function finds the matching `section` (testsuite) by comparing `testcase.classname` against `section.name`:

1. Exact match: `sectionMap.get(testCase.classname)`
2. Partial match: `classname.includes(sectionName) || sectionName.includes(classname)` — first match wins
3. No match: use `sections[0]` (first section)
4. No sections at all: create a synthetic `"Default Section"` with a new UUID

### Status ID mapping (`transformXmlData`)

| Test case status | TestRail `status_id` |
|---|---|
| `"failed"` or `failure` present | `5` (Failed) |
| `"skipped"` or `skipped` present | `4` (Retest) |
| `"blocked"` | `2` (Blocked) |
| `"untested"` | `3` (Untested) |
| `"passed"` or no status element | `1` (Passed) |

The `defects` field is set to `failure.message` on failures. The `comment` field is set to `skipped.message` on skipped tests.

---

## Raw JSON format

When `loadRunData` is called on a `.json` file, the file is read as UTF-8 and parsed with `JSON.parse`. No schema is enforced by the loader itself.

The calling code (e.g., `testfiesta run:submit`) passes the result directly to `TestFiestaETL.load()`, which applies its own internal schema expectations.

If `JSON.parse` throws (malformed JSON), the error propagates as an `Error` result from `loadRunData`.

### Expected JSON structure for TestFiesta

When passing raw JSON to `testfiesta run:submit`, the file must already be in a format that `TestFiestaETL.load()` can consume. The expected structure mirrors the `JunitParserResult` shape:

```json
{
  "root": {
    "name": "My Test Run",
    "tests": 10,
    "failures": 1,
    "errors": 0,
    "skipped": 0,
    "time": 12.5
  },
  "section": [
    {
      "name": "Login Tests",
      "tests": 5,
      "failures": 0,
      "errors": 0,
      "skipped": 0,
      "time": 6.2,
      "testcases": [...]
    }
  ],
  "testcase": [
    {
      "name": "should login with valid credentials",
      "classname": "Login Tests",
      "time": 1.2,
      "status": "passed"
    },
    {
      "name": "should reject invalid password",
      "classname": "Login Tests",
      "time": 0.5,
      "status": "failed",
      "failure": {
        "message": "Expected redirect to /dashboard",
        "type": "AssertionError",
        "_text": "AssertionError: Expected redirect...\n  at ..."
      }
    }
  ]
}
```

---

## Mocha / Jest JSON reporters

There is no dedicated Mocha or Jest JSON parser in the current codebase. To use tacotruck with Mocha or Jest:

1. Use a JUnit reporter to produce `.xml` output and pass the XML file with `-d`.
2. Or write a custom transformer that produces a `JunitParserResult`-shaped JSON file and pass the `.json` file with `-d`.

**Recommended Jest configuration:**

```json
{
  "reporters": ["default", "jest-junit"]
}
```

**Recommended Mocha configuration:**

```bash
mocha --reporter mocha-junit-reporter
```

---

## Ignore config for file sources

When using `XUnitParser` (the `migrate` path), an `ignoreConfig` can suppress specific fields:

```json
{
  "runs": {
    "name": true
  },
  "suites": {
    "timestamp": true
  },
  "executions": {
    "system-out": true
  }
}
```

Setting a field to `true` deletes it from each record before mapping. Setting it to `false` or omitting it has no effect.

Note: This uses field names from the raw XML (pre-mapping), not the mapped names.
