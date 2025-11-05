# Development Guide

<cite>
**Referenced Files in This Document**   
- [CONTRIBUTING.md](file://CONTRIBUTING.md)
- [Makefile](file://Makefile)
- [go.mod](file://go.mod)
- [README.md](file://README.md)
</cite>

## Table of Contents
1. [Introduction](#introduction)
2. [Contribution Guidelines](#contribution-guidelines)
3. [Build System](#build-system)
4. [Dependency Management](#dependency-management)
5. [Development Environment Setup](#development-environment-setup)
6. [Testing Procedures](#testing-procedures)
7. [Pull Request Workflow](#pull-request-workflow)
8. [Code Quality and Best Practices](#code-quality-and-best-practices)
9. [Common Issues and Troubleshooting](#common-issues-and-troubleshooting)
10. [Performance Considerations](#performance-considerations)

## Introduction

This development guide provides comprehensive information for contributing to the Gitea project. It covers the essential aspects of development including contribution guidelines, build system configuration, dependency management, testing procedures, and code quality standards. The guide is designed to help developers set up their development environment, understand the project structure, and follow best practices when contributing code changes to the Gitea repository.

**Section sources**
- [README.md](file://README.md#L14-L213)

## Contribution Guidelines

The Gitea project welcomes contributions from the community and has established clear guidelines to ensure a smooth and efficient contribution process. Contributors are expected to follow the workflow of Fork -> Patch -> Push -> Pull Request. Before starting work on any changes, it is mandatory to read the [CONTRIBUTORS GUIDE](CONTRIBUTING.md) to understand the project's expectations and requirements.

The contribution process emphasizes early communication and design discussion. Contributors should file an issue or comment on an existing one before beginning implementation work. This practice helps prevent duplication of effort and ensures that proposed changes align with the project's goals. For significant changes such as new features, a formal change proposal process must be followed to validate the design and gather community feedback.

Security-related issues should be reported privately to security@gitea.io rather than being filed as public issues. The project also uses a backport bot to automate the process of applying fixes to release branches once they have been merged into the main branch. Pull requests require approval from at least two maintainers before they can be merged, with exceptions for refactoring and documentation PRs which require only one approval.

**Section sources**
- [CONTRIBUTING.md](file://CONTRIBUTING.md#L0-L606)

## Build System

The Gitea project uses a comprehensive Makefile-based build system that orchestrates the compilation and packaging process. The build system is designed to handle both backend (Go) and frontend (JavaScript/TypeScript) components separately, allowing for flexible development workflows.

The primary build targets include:
- `make build`: Builds the complete application
- `make backend`: Builds only the Go backend components
- `make frontend`: Builds only the frontend assets using Webpack
- `make release`: Creates release packages for multiple platforms

The build process supports various configuration options through environment variables and make flags. The `TAGS` variable is particularly important as it controls build-time features such as SQLite support (`TAGS="bindata sqlite sqlite_unlock_notify"`) or PAM authentication. The build system also includes targets for generating documentation, man pages, and source distributions.

For development, the build system provides watch targets that automatically rebuild components when source files change:
- `make watch`: Watches all files and continuously rebuilds
- `make watch-frontend`: Watches frontend files and rebuilds
- `make watch-backend`: Watches backend files and continuously rebuilds

The Makefile also integrates with various code generation tools that create necessary files during the build process, such as bindata files for embedded assets and generated code from templates.

**Section sources**
- [Makefile](file://Makefile#L0-L958)
- [README.md](file://README.md#L14-L213)

## Dependency Management

Gitea uses Go Modules for backend dependency management, as specified in the go.mod and go.sum files. The project follows strict guidelines for modifying dependencies, requiring that changes to go.mod and go.sum be justified in the PR description and verified by reviewers to reference existing upstream commits.

The go.mod file specifies the minimum Go version required (1.25.3) and lists all direct dependencies with their versions. The project uses several replace directives to point to forked or modified versions of dependencies, such as replacing github.com/nektos/act with gitea.com/gitea/act for better integration with Gitea Actions.

For frontend dependencies, Gitea uses pnpm (a fast, disk-space-efficient package manager) with a pnpm-lock.yaml file to ensure consistent dependency resolution. The frontend dependencies are managed separately from the backend and include tools for linting, testing, and building the user interface.

The project also excludes certain dependency versions that are known to cause issues, such as various versions of uuid packages that have compatibility problems. This exclusion mechanism helps prevent accidental upgrades to problematic versions.

**Section sources**
- [go.mod](file://go.mod#L0-L309)
- [Makefile](file://Makefile#L0-L958)

## Development Environment Setup

Setting up a development environment for Gitea requires several prerequisites and configuration steps. The backend is written in Go and requires Go 1.25.3 or later, while the frontend requires Node.js LTS or greater and pnpm for package management.

To set up the development environment:
1. Install Go (1.25.3+) from https://go.dev/dl/
2. Install Node.js LTS from https://nodejs.org/en/download/
3. Install pnpm from https://pnpm.io/installation
4. Clone the Gitea repository
5. Install development dependencies with `make deps`

The development environment can be customized through various environment variables that control database connections for testing:
- `TEST_MYSQL_HOST`: MySQL test database host
- `TEST_PGSQL_HOST`: PostgreSQL test database host
- `TEST_MSSQL_HOST`: MSSQL test database host

For convenience, the project provides a target to install all necessary development tools:
```bash
make deps-tools
```

This installs tools such as golangci-lint, misspell, and swagger generators that are used for code quality checks and documentation generation.

**Section sources**
- [Makefile](file://Makefile#L0-L958)
- [README.md](file://README.md#L14-L213)

## Testing Procedures

Gitea has a comprehensive testing framework that includes unit tests, integration tests, and end-to-end tests. The testing infrastructure is designed to validate functionality across different database backends and ensure code quality through automated checks.

The primary testing targets in the Makefile include:
- `make test`: Runs all tests
- `make test-backend`: Runs Go backend tests
- `make test-frontend`: Runs frontend tests with Vitest
- `make test-sqlite`: Runs integration tests with SQLite
- `make test-mysql`: Runs integration tests with MySQL
- `make test-pgsql`: Runs integration tests with PostgreSQL
- `make test-mssql`: Runs integration tests with MSSQL
- `make test-e2e`: Runs end-to-end tests with Playwright

Integration tests can be run against specific database configurations by setting environment variables:
```bash
TEST_MYSQL_HOST=localhost:3306 TEST_MYSQL_DBNAME=testgitea TEST_MYSQL_USERNAME=root make test-mysql
```

The project also supports running specific tests by name:
```bash
make test#TestName
make test-sqlite#TestName
make test-e2e-sqlite#TestName
```

Code coverage can be measured with:
```bash
make unit-test-coverage
make integration-test-coverage
```

The testing framework includes specialized test configurations for different scenarios, such as migration testing and performance benchmarking. End-to-end tests use Playwright to automate browser interactions and validate the complete user experience.

**Section sources**
- [Makefile](file://Makefile#L0-L958)
- [CONTRIBUTING.md](file://CONTRIBUTING.md#L0-L606)

## Pull Request Workflow

The pull request workflow in Gitea follows a structured process designed to maintain code quality and facilitate efficient code review. Contributors should ensure their PRs are easy to review by following best practices such as keeping PRs small, focusing on a single change, and allowing edits by maintainers.

PRs must follow specific formatting guidelines:
- The title should describe the problem being fixed, not the solution
- The PR summary should include a detailed description of the changes
- For UI changes, both before and after screenshots must be included
- For new features, the summary must contain clear usage and testing descriptions

PRs that fix issues should reference the issue number in the summary:
```
Fixes #1234
Closes #5678
```

Breaking changes require special handling with a BREAKING section in the PR summary that explains the impact on users and how to mitigate the changes. The PR title and summary are used to generate the final commit message, which is rewritten by maintainers to ensure clarity and consistency.

The project uses automated labeling to track PR status, with labels indicating the affected components, PR type, and approval status. A PR must be labeled correctly before it can be merged. The merge queue is processed in order, with PRs receiving the `reviewed/wait-merge` label being merged as soon as possible.

**Section sources**
- [CONTRIBUTING.md](file://CONTRIBUTING.md#L0-L606)

## Code Quality and Best Practices

Gitea enforces strict code quality standards through automated tools and manual review processes. Contributors are expected to run various checks before submitting PRs to ensure code consistency and quality.

The primary code quality targets include:
- `make fmt`: Formats Go and template code using gofumpt
- `make lint`: Runs comprehensive linting on all code
- `make lint-backend`: Lints backend Go code
- `make lint-frontend`: Lints frontend JavaScript/TypeScript code
- `make lint-spell`: Checks spelling in code and documentation
- `make checks`: Runs all consistency checks

The project uses golangci-lint with a comprehensive configuration to enforce Go coding standards. Frontend code is linted with ESLint and Stylelint according to project-specific rules. Template files are checked for proper formatting and syntax.

Code must conform to the project's style guide, which includes:
- Using the standard copyright header in new files
- Running `make fmt` before committing
- Following Go best practices and idioms
- Writing clear, concise comments and documentation

The project also performs automated security checks using govulncheck to identify known vulnerabilities in dependencies.

**Section sources**
- [Makefile](file://Makefile#L0-L958)
- [CONTRIBUTING.md](file://CONTRIBUTING.md#L0-L606)

## Common Issues and Troubleshooting

Developers may encounter various issues during the development process, particularly related to build failures, test failures, and environment configuration.

Common build issues include:
- Missing dependencies: Ensure all tools are installed with `make deps`
- Go version mismatch: Verify Go 1.25.3 or later is installed
- Node.js version issues: Ensure Node.js LTS or greater is used
- pnpm not found: Install pnpm globally

Test failures can occur due to:
- Database connectivity issues: Verify test database containers are running
- Environment variable configuration: Check TEST_* environment variables
- Race conditions: Use `RACE_ENABLED=true` to run tests with race detection
- Flaky tests: Some integration tests may be environment-dependent

When encountering build or test failures, developers should:
1. Run `make deps` to ensure all dependencies are installed
2. Verify environment variables are correctly set
3. Check the specific error message in the output
4. Run the failing test individually to isolate the issue
5. Consult the project documentation and existing issues for similar problems

The project provides various diagnostic targets:
- `make go-check`: Verifies Go installation and version
- `make node-check`: Verifies Node.js and pnpm installation
- `make git-check`: Verifies Git with LFS support
- `make tidy-check`: Ensures go.mod and go.sum are up to date

**Section sources**
- [Makefile](file://Makefile#L0-L958)
- [CONTRIBUTING.md](file://CONTRIBUTING.md#L0-L606)

## Performance Considerations

The Gitea development workflow includes several performance considerations to optimize both the development experience and the resulting application performance.

For development workflow performance:
- The build system is designed to minimize unnecessary work
- Watch targets only rebuild changed components
- Dependency downloads are cached
- Frontend and backend builds can be run independently

For application performance:
- The project uses various optimization techniques in the codebase
- Database queries are optimized and indexed
- Caching mechanisms are implemented for frequently accessed data
- Resource usage is monitored and optimized

Performance testing is supported through:
- Benchmark targets (`make bench-sqlite`, `make bench-mysql`, etc.)
- Code coverage analysis to identify untested performance-critical paths
- Memory and CPU profiling capabilities

Developers should consider performance implications when implementing new features or modifying existing code, particularly for operations that may be performed frequently or on large datasets. The project encourages the use of efficient algorithms and data structures, and the avoidance of unnecessary database queries or file operations.

**Section sources**
- [Makefile](file://Makefile#L0-L958)
- [CONTRIBUTING.md](file://CONTRIBUTING.md#L0-L606)