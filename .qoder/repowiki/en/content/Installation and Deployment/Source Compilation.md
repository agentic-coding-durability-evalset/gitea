# Source Compilation

<cite>
**Referenced Files in This Document**   
- [go.mod](file://go.mod)
- [go.sum](file://go.sum)
- [Makefile](file://Makefile)
- [README.md](file://README.md)
- [CONTRIBUTING.md](file://CONTRIBUTING.md)
</cite>

## Table of Contents
1. [Introduction](#introduction)
2. [Go Module System](#go-module-system)
3. [Build Process with Makefile](#build-process-with-makefile)
4. [Setting Up the Development Environment](#setting-up-the-development-environment)
5. [Compiling the Binary](#compiling-the-binary)
6. [Custom Builds](#custom-builds)
7. [Common Issues](#common-issues)
8. [Build-Time Configuration](#build-time-configuration)
9. [Conclusion](#conclusion)

## Introduction
This document provides comprehensive guidance on compiling Gitea from source. It covers the Go module system, the build process using the Makefile, setting up the development environment, and compiling the binary. Additionally, it addresses common issues, custom builds, and build-time configuration options.

**Section sources**
- [README.md](file://README.md#L1-L214)

## Go Module System
The Go module system is used to manage dependencies in Gitea. The `go.mod` file specifies the module path and the required dependencies, while the `go.sum` file contains the expected cryptographic checksums of the content of specific module versions.

### Dependency Management
Dependencies are managed using Go Modules. The `go.mod` file lists the required modules and their versions. The `go.sum` file ensures the integrity of the downloaded modules by storing their checksums.

### Versioning
Versioning in Go Modules is semantic. The `go.mod` file specifies the minimum version required for each dependency. The `go.sum` file records the exact version and checksum of each module used in the project.

**Section sources**
- [go.mod](file://go.mod#L1-L310)
- [go.sum](file://go.sum#L1-L1155)
- [CONTRIBUTING.md](file://CONTRIBUTING.md#L1-L607)

## Build Process with Makefile
The build process is orchestrated by the `Makefile`, which defines various targets for building, testing, and releasing Gitea.

### Build Targets
The `Makefile` includes several targets:
- `build`: Builds the backend and frontend.
- `backend`: Builds the backend files.
- `frontend`: Builds the frontend files.
- `release`: Prepares the release binaries.

### Cross-Compilation
Cross-compilation is supported through the `release` target, which uses the `xgo` tool to build binaries for different platforms and architectures.

### Custom Build Tags
Custom build tags can be specified using the `TAGS` variable. For example, to include SQLite support, use `TAGS="bindata sqlite sqlite_unlock_notify"`.

**Section sources**
- [Makefile](file://Makefile#L1-L959)

## Setting Up the Development Environment
To set up the development environment, ensure you have the following tools installed:
- Go (version specified in `go.mod`)
- Node.js (version specified in `package.json`)
- pnpm

### Installing Dependencies
Dependencies can be installed using the following commands:
```sh
make tidy
make node-check
```

**Section sources**
- [README.md](file://README.md#L1-L214)
- [Makefile](file://Makefile#L1-L959)

## Compiling the Binary
To compile the binary, run the following command:
```sh
TAGS="bindata" make build
```
This command builds the backend and frontend, generating the `gitea` binary.

**Section sources**
- [README.md](file://README.md#L1-L214)
- [Makefile](file://Makefile#L1-L959)

## Custom Builds
Custom builds can be created by specifying different build tags. For example, to enable SQLite support:
```sh
TAGS="bindata sqlite sqlite_unlock_notify" make build
```

### Disabling Features
To disable certain features, omit the corresponding build tags. For example, to build without SQLite support:
```sh
TAGS="bindata" make build
```

**Section sources**
- [README.md](file://README.md#L1-L214)
- [Makefile](file://Makefile#L1-L959)

## Common Issues
### Dependency Resolution Failures
Ensure that the `go.mod` and `go.sum` files are up to date by running `make tidy`.

### Compilation Errors
Check the Go version and ensure it meets the minimum requirement specified in `go.mod`.

### Platform-Specific Build Problems
For cross-compilation issues, ensure that the `xgo` tool is correctly installed and configured.

**Section sources**
- [CONTRIBUTING.md](file://CONTRIBUTING.md#L1-L607)
- [Makefile](file://Makefile#L1-L959)

## Build-Time Configuration
Build-time configuration options are specified using build tags and environment variables. These options affect the runtime behavior of the compiled binary.

### Configuration Options
- `TAGS`: Specifies build tags to include or exclude features.
- `LDFLAGS`: Specifies linker flags for customizing the binary.

**Section sources**
- [Makefile](file://Makefile#L1-L959)

## Conclusion
Compiling Gitea from source involves understanding the Go module system, using the Makefile for building, and setting up the development environment. By following the steps outlined in this document, you can successfully compile and customize Gitea for your needs.

**Section sources**
- [README.md](file://README.md#L1-L214)
- [Makefile](file://Makefile#L1-L959)
- [CONTRIBUTING.md](file://CONTRIBUTING.md#L1-L607)