<!-- spec-meta last-updated-sha: HEAD last-updated-pr: null last-updated-date: 2026-02-20 spec-version: 1.0 -->

# tacotruck

tacotruck is a test/QA data pipeline CLI and Node.js library published as `@testfiesta/tacotruck` (version `0.3.0`). It moves quality data between test case management systems and CI result formats. It runs as a standalone binary, a Docker container, a GitHub Action, or a CircleCI Orb, and can be imported as an ESM/CJS module inside JavaScript projects.

## Purpose

tacotruck solves three categories of problems:

1. **Result reporting** — take JUnit/xUnit XML output from a test runner and push it into one or more TCMs (TestFiesta, TestRail, Zephyr, etc.).
2. **TCM migration** — pull all historical data from a source TCM and load it into a target TCM, keeping data in sync during a cutover period.
3. **Evidence upload** — programmatically attach screenshots, logs, and other artifacts to test management records.

## Supported integrations

| Integration  | Source (pull from) | Target (push to) |
|---|:---:|:---:|
| TestRail     | yes | yes |
| TestFiesta   | yes | yes |
| JIRA         | yes | yes |
| JUnit XML    | yes | — |

For each API integration, any combination of the entity types listed in the integration-specific specs is available.

## Installation

### npm (library)

```bash
npm install @testfiesta/tacotruck
```

The package exports two entry points:

- `.` (`dist/index.js` / `dist/index.cjs`) — programmatic API (`pushData`, `pullData`)
- `./cli/index` (`dist/cli/index.js` / `dist/cli/index.cjs`) — CLI entry point

### Prebuilt binary

Download the release archive for your OS/architecture from the [GitHub releases page](https://github.com/testfiesta/tacotruck/releases), unpack it, and place the binary on your `PATH`. Verify with:

```bash
tacotruck --help
```

### Docker

A Docker image is available. The binary entry point is `tacotruck`.

### GitHub Action / CircleCI Orb

See the [TestFiesta GitHub organization](https://github.com/testfiesta/) for action and orb definitions.

## Quick start

### Submit JUnit results to TestFiesta

```bash
tacotruck testfiesta run:submit \
  -d ./test-results.xml \
  -t TF_XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX \
  -h my-org \
  -p MY-PROJECT
```

### Submit JUnit results to TestRail

```bash
tacotruck testrail run:submit \
  -d ./test-results.xml \
  -e user@example.com \
  -p apikey \
  -u https://example.testrail.io \
  -i 42 \
  -n "My CI Run"
```

### Migrate data from TestRail to TestFiesta

```bash
# Using a credentials file
tacotruck migrate -s testrail -t testfiesta -c ./creds.json

# Using environment variables
export TESTRAIL_SOURCE_CREDENTIALS='{"base64Credentials":"...","base_url":"https://example.testrail.io/"}'
export TESTFIESTA_TARGET_CREDENTIALS='{"token":"TF_...","base_url":"https://api.testfiesta.ai/"}'
tacotruck migrate -s testrail -t testfiesta
```

### Use as a Node.js module

```javascript
import { pushData, pullData } from '@testfiesta/tacotruck'

const config = {
  credentials: './creds.json',
  source: 'testrail',
  target: 'testfiesta',
}

const data = await pullData(config, { cases: [{ id: 1 }, { id: 2 }] })
await pushData(config, data)
```

## Repository layout

```
bin/index.js              # CLI entry shim
configs/                  # Bundled integration config files
  testfiesta.json
  testrail.json
  junit.json
  jira.json
  sample_config.json      # Annotated reference config
src/
  cli/                    # Commander command definitions
    index.ts              # Program root; registers all commands
    commands/
      migrate.ts          # `migrate` command
      testfiesta/         # `testfiesta` sub-commands
      testrail/           # `testrail` sub-commands
    utils.ts              # ASCII title rendering, package root management
  controllers/            # ETL orchestration classes
    etl-base-v2.ts        # Base ETL class (extract/transform/load pipeline)
    testfiesta-etl.ts     # TestFiesta-specific ETL methods
    testrail-etl.ts       # TestRail-specific ETL methods
    managers/             # Decomposed manager classes
  services/
    api-client.ts         # HTTP client wrapper
  utils/                  # Pure utility modules
    config-schema.ts      # Zod schemas for all config types
    enhanced-config-loader.ts
    junit-xml-parser.ts
    xunit-parser.ts / xunit-parser-v2.ts
    xml-transform.ts
    batch-processor.ts
    network.ts
    run-data-loader.ts
    ...
```
