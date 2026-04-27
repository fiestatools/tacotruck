# Contributing to TacoTruck

Thank you for your interest in contributing to TacoTruck! 💙 This guide will help you set up your local development environment and get started with contributing.

## Table of Contents

- [Prerequisites](#prerequisites)
- [Local Development Setup](#local-development-setup)
- [Development Workflow](#development-workflow)
- [Running Tests](#running-tests)
- [Code Style and Linting](#code-style-and-linting)
- [Building the Project](#building-the-project)
- [Submitting Changes](#submitting-changes)
- [Project Structure](#project-structure)
- [Available Scripts](#available-scripts)

## Prerequisites

Before you begin, ensure you have the following installed:

- **Node.js** (version 20 or higher) - [Download here](https://nodejs.org/)
- **npm** (comes with Node.js)
- **Git** - [Download here](https://git-scm.com/)

## Local Development Setup

### 1. Fork and Clone the Repository

```bash
# Fork the repository on GitHub, then clone your fork
git clone https://github.com/YOUR_USERNAME/tacotruck.git
cd tacotruck
```

### 2. Install Dependencies

```bash
npm install
```

This will install all required dependencies and set up git hooks for code quality.

### 3. Verify Your Setup

Run the following commands to ensure everything is working:

```bash
npm run build

npm run test

# Check code formatting and linting
npm run lint

# Verify TypeScript types
npm run typecheck
```

If all commands complete successfully, you're ready to start developing! 🎉

## Development Workflow

### 1. Create a Feature Branch

```bash
git checkout -b feature/your-feature-name
# or
git checkout -b fix/your-bug-fix
```

### 2. Make Your Changes

- Edit the relevant files in the `src/` directory
- Add tests for new functionality in the appropriate `__tests__` directories
- Update documentation if needed

### 3. Test Your Changes

```bash
npm run test:watch

npm run test
```

### 4. Lint and Format

The project uses ESLint with automatic formatting. Pre-commit hooks will automatically run linting, but you can also run them manually:

```bash
npm run lint

npm run lint:fix
```

## Running Tests

TacoTruck uses [Vitest](https://vitest.dev/) for testing with comprehensive test coverage:

```bash
# Run all tests once
npm run test

npm run test:watch

```

### Test Structure

- Unit tests are located in `__tests__` directories alongside the source code
- Integration tests use [MSW (Mock Service Worker)](https://mswjs.io/) for API mocking
- Test files follow the pattern `*.test.ts`

## Code Style and Linting

The project uses:

- **ESLint** with [@antfu/eslint-config](https://github.com/antfu/eslint-config) for code quality
- **TypeScript** for type safety
- **Automatic formatting** on commit via git hooks

### Pre-commit Hooks

The project automatically runs linting on all staged files before each commit. If there are any issues, the commit will be blocked until they're resolved.

## Building the Project

```bash
# Build the project for distribution
npm run build

# Build CLI binaries (for releases)
npm run build:binaries
```

The build process uses [tsdown](https://tsdown.vercel.app/) and generates:

- CommonJS and ESM modules in `dist/`
- Type definitions
- CLI executable in `bin/`

## Submitting Changes

### 1. Commit Your Changes

```bash
git add .
git commit -m "feat: add new feature description"
# or
git commit -m "fix: resolve issue with specific component"
```

Use [conventional commit messages](https://www.conventionalcommits.org/):

- `feat:` for new features
- `fix:` for bug fixes
- `docs:` for documentation changes
- `test:` for adding tests
- `refactor:` for code refactoring
- `chore:` for maintenance tasks

### 2. Push to Your Fork

```bash
git push origin feature/your-feature-name
```

### 3. Create a Pull Request

1. Go to the [TacoTruck repository](https://github.com/fiestatools/tacotruck)
2. Click "New Pull Request"
3. Select your fork and branch
4. Fill out the PR template with:
   - Clear description of changes
   - Link to any related issues
   - Screenshots (if applicable)
   - Testing instructions

## Project Structure

```
tacotruck/
├── .github/           # GitHub workflows and templates
├── bin/              # CLI executable
├── configs/          # Configuration files
├── dist/             # Built output (generated)
├── scripts/          # Build and utility scripts
├── src/              # Source code
│   ├── cli/          # CLI commands and utilities
│   ├── clients/      # API clients (TestFiesta, TestRail)
│   └── utils/        # Shared utilities
├── test/             # Test setup and mocks
└── package.json      # Project configuration
```

### Key Directories

- **`src/cli/`** - Command-line interface implementation
- **`src/clients/`** - API client libraries for TestFiesta and TestRail
- **`src/utils/`** - Shared utilities (parsers, network, etc.)
- **`test/`** - Test configuration and mock service workers

## Getting Help

If you need assistance:

1. Check the [existing issues](https://github.com/fiestatools/tacotruck/issues)
2. Review the [TestFiesta documentation](https://docs.testfiesta.com)
3. Create a new issue with the "question" label
4. Join our community discussions

## Code of Conduct

Please note that this project is released with a [Contributor Code of Conduct](https://docs.testfiesta.com/community/code-of-conduct). By participating in this project you agree to abide by its terms.

---

Thank you for contributing to TacoTruck! Your efforts help make quality assurance better for everyone. 🌮✨
