# TacoTruck

[![ci](https://github.com/fiestatools/tacotruck/actions/workflows/ci.yml/badge.svg)](https://github.com/fiestatools/tacotruck/actions/workflows/ci.yml)
![NPM Version](https://img.shields.io/npm/v/%40fiestatools%2Ftacotruck)

Tacotruck exists to make it easy to move quality data to wherever you need it. Whether you are looking to report test results to your various quality systems or trying to migrate historical data between test case management tools, tacotruck provides a simple, easily extendable interface for doing so.

### Table of Contents

- 🚀 [Getting Started](#getting-started)
- 📖 [Documentation](#documentation)
- 💙 [Contribute](#contribute)
- 🏠 [Local Development](#local-development)
- 🔗 [Follow Us](#follow-us)
- ⚖️ [License](#license)

---

## <a name="getting-started">🚀 Getting Started</a>

### Usage as a CLI

#### Standalone Installation

##### MacOS & Linux

```bash
curl -fsSL https://testfiesta.com/install-tacotruck-cli.sh | bash
```

##### Homebrew

```bash
brew tap fiestatools/tacotruck
brew install tacotruck
```

#### Run instantly using npx

```bash
npx @fiestatools/tacotruck
```

#### Install globally using npm

```bash
npm install -g @fiestatools/tacotruck
```

### Usage as a library

```bash
npm install @fiestatools/tacotruck
```

```typescript
import { TestFiestaClient, TestRailClient } from '@fiestatools/tacotruck'

const tfClient = new TestFiestaClient({
  apiKey: '<YOUR_TF_API_KEY>',
  organizationHandle: '<YOUR_TF_ORGANIZATION_HANDLE>',
  // baseUrl is optional, defaults to 'https://api.testfiesta.com'
})

const trClient = new TestRailClient({
  apiKey: '<YOUR_TR_USERNAME>:<YOUR_TR_PASSWORD>',
  baseUrl: 'http://<username>.testrails.com',
})
```

### Example usage with various testing frameworks

Check out [tacotruck-examples](https://github.com/fiestatools/tacotruck-examples) for examples of using `tacotruck` with your favorite language and framework.

## <a name="documentation">📖 Documentation</a>

We highly recommend you take a look at the [Testfiesta docs](https://docs.testfiesta.com) for more information.

For CLI-specific documentation, check out the [TacoTruck CLI Getting Started Guide](https://docs.testfiesta.com/automation/tacotruck-cli/get-start).

## <a name="contribute">💙 Contribute</a>

We invite you to contribute and help improve Tacotruck 💙

Here are a few ways you can get involved:

- **Reporting Bugs:** If you come across any bugs or issues, please check out the [reporting bugs guide](https://docs.testfiesta.com/community/reporting-bugs) to learn how to submit a bug report.
- **Suggestions:** Have ideas to enhance Tacotruck? We'd love to hear them! Check out the [contribution guide](.github/CONTRIBUTING.md) to share your suggestions.
- **Questions:** If you have questions or need assistance, the [getting help guide](https://docs.testfiesta.com/tacotruck/community/getting-help) provides resources to help you out.

## <a name="local-development">🏠 Local Development</a>

Follow our [Contributing Guide](.github/CONTRIBUTING.md) to set up your local development environment and start contributing to the framework and documentation.

## <a name="follow-us">🔗 Follow Us</a>

## <a name="license">⚖️ License</a>
