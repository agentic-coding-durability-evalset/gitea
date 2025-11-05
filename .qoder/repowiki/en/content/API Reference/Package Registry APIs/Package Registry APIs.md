# Package Registry APIs

<cite>
**Referenced Files in This Document**   
- [api.go](file://routers/api/packages/api.go)
- [npm.go](file://routers/api/packages/npm/npm.go)
- [npm/api.go](file://routers/api/packages/npm/api.go)
- [nuget/nuget.go](file://routers/api/packages/nuget/nuget.go)
- [nuget/api_v3.go](file://routers/api/packages/nuget/api_v3.go)
- [nuget/links.go](file://routers/api/packages/nuget/links.go)
- [container/manifest.go](file://routers/api/packages/container/manifest.go)
- [container/auth.go](file://routers/api/packages/container/auth.go)
- [maven/maven.go](file://routers/api/packages/maven/maven.go)
- [maven/api.go](file://routers/api/packages/maven/api.go)
- [generic/generic.go](file://routers/api/packages/generic/generic.go)
- [package.go](file://models/packages/package.go)
- [package_version.go](file://models/packages/package_version.go)
- [package_file.go](file://models/packages/package_file.go)
- [descriptor.go](file://models/packages/descriptor.go)
- [packages.go](file://services/packages/packages.go)
- [auth.go](file://services/packages/auth.go)
- [README.md](file://routers/api/packages/README.md)
</cite>

## Table of Contents
1. [Introduction](#introduction)
2. [Authentication Methods](#authentication-methods)
3. [Package Types and Endpoints](#package-types-and-endpoints)
   - [npm](#npm)
   - [NuGet](#nuget)
   - [Container/OCI](#containeroci)
   - [Maven](#maven)
   - [Generic](#generic)
4. [Package Metadata Structure](#package-metadata-structure)
5. [Versioning Schemes](#versioning-schemes)
6. [Dependency Resolution](#dependency-resolution)
7. [Common Issues](#common-issues)
8. [Performance Considerations](#performance-considerations)
9. [Client Examples](#client-examples)

## Introduction
Gitea provides a comprehensive Package Registry system that supports multiple package managers through protocol-specific endpoints. This documentation details the API endpoints, authentication methods, metadata formats, and usage patterns for each supported package type including npm, NuGet, Container/OCI, Maven, and Generic packages. The registry enables users to publish and consume packages within their Gitea instance, with support for API tokens and deploy tokens for authentication.

**Section sources**
- [README.md](file://routers/api/packages/README.md)

## Authentication Methods
Gitea's Package Registry supports multiple authentication methods depending on the package type. The primary authentication methods are API tokens and basic authentication using username and password.

For npm, NuGet, Maven, and Generic packages, users can authenticate using an API token via the Authorization header with the "Bearer" scheme. The token must have appropriate permissions for the package operations being performed.

NuGet clients can authenticate using either API tokens or basic authentication with username and password. When using basic authentication, the username is the Gitea username and the password is an API token.

Container/OCI registry uses a token-based authentication scheme where clients first request a token from the `/v2/token` endpoint, providing their credentials. This token is then used to authenticate subsequent requests to the registry endpoints.

Deploy tokens can also be used for authentication, particularly useful for CI/CD pipelines and automated deployments. These tokens can be scoped to specific repositories or organizations with defined permissions.

**Section sources**
- [api.go](file://routers/api/packages/api.go)
- [auth.go](file://routers/api/packages/container/auth.go)
- [nuget/nuget.go](file://routers/api/packages/nuget/nuget.go)

## Package Types and Endpoints

### npm
The npm package registry endpoints follow the npm Registry API specification. Packages are organized by scope and package name.

**Endpoints:**
- `GET /api/packages/{owner}/npm/{package}` - Retrieve package metadata
- `GET /api/packages/{owner}/npm/{package}/-/{version}/{filename}` - Download package file
- `PUT /api/packages/{owner}/npm/-/package` - Publish a package
- `PUT /api/packages/{owner}/npm/{package}/-disttag/{tag}` - Add or update a distribution tag

The package metadata endpoint returns a JSON object containing all versions of the package, distribution tags, and version-specific metadata including dependencies, license information, and download URLs. When publishing packages, the npm client sends a JSON payload containing the package metadata and file attachments.

**Section sources**
- [npm.go](file://routers/api/packages/npm/npm.go)
- [npm/api.go](file://routers/api/packages/npm/api.go)

### NuGet
The NuGet package registry implements both v2 and v3 protocols, providing compatibility with various NuGet clients.

**Endpoints:**
- `GET /api/packages/{owner}/nuget/index.json` - Service index (v3)
- `GET /api/packages/{owner}/nuget/registration/{id}/index.json` - Registration index
- `GET /api/packages/{owner}/nuget/registration/{id}/{version}.json` - Registration leaf
- `GET /api/packages/{owner}/nuget/package/{id}/{version}/{filename}` - Download package
- `PUT /api/packages/{owner}/nuget/` - Upload package
- `DELETE /api/packages/{owner}/nuget/{id}/{version}` - Delete package

The service index provides discovery of available resources. The registration index contains information about all versions of a package, while the registration leaf provides detailed metadata for a specific version including dependencies, authors, and publication date. Package uploads use the standard NuGet package format (.nupkg).

**Section sources**
- [nuget/nuget.go](file://routers/api/packages/nuget/nuget.go)
- [nuget/api_v3.go](file://routers/api/packages/nuget/api_v3.go)
- [nuget/links.go](file://routers/api/packages/nuget/links.go)

### Container/OCI
The Container/OCI registry implements the Open Container Initiative Distribution Specification, allowing compatibility with Docker, containerd, and other OCI-compliant clients.

**Endpoints:**
- `GET /v2/` - Check registry support
- `GET /v2/token` - Authenticate and retrieve token
- `GET /v2/_catalog` - List repositories
- `GET /v2/{repository}/tags/list` - List tags for a repository
- `POST /v2/{repository}/blobs/uploads/` - Initiate blob upload
- `PUT /v2/{repository}/blobs/uploads/{uuid}` - Complete blob upload
- `GET /v2/{repository}/blobs/{digest}` - Retrieve blob
- `HEAD /v2/{repository}/manifests/{reference}` - Check manifest existence
- `GET /v2/{repository}/manifests/{reference}` - Retrieve manifest
- `PUT /v2/{repository}/manifests/{reference}` - Upload manifest

The registry supports both Docker and OCI image manifests, as well as image indexes (manifest lists). Clients authenticate by first requesting a token from the `/v2/token` endpoint, then using that token to access the registry endpoints. The registry handles both tagged references and digests for image identification.

**Section sources**
- [api.go](file://routers/api/packages/api.go)
- [container/manifest.go](file://routers/api/packages/container/manifest.go)
- [container/auth.go](file://routers/api/packages/container/auth.go)

### Maven
The Maven package registry follows the Maven repository layout and protocol, supporting both Maven and Gradle clients.

**Endpoints:**
- `GET /api/packages/{owner}/maven/{group}/{artifact}/{version}/{filename}` - Download package file
- `PUT /api/packages/{owner}/maven/{group}/{artifact}/{version}/{filename}` - Upload package file
- `HEAD /api/packages/{owner}/maven/{group}/{artifact}/{version}/{filename}` - Check file existence
- `GET /api/packages/{owner}/maven/{group}/{artifact}/maven-metadata.xml` - Retrieve version metadata

Packages are organized in a directory structure based on group ID, artifact ID, and version. The registry supports standard Maven artifacts including JAR files, POM files, and associated checksums (.md5, .sha1, etc.). When uploading artifacts, clients send the file content directly to the appropriate endpoint. The registry automatically generates and serves maven-metadata.xml files containing version information.

**Section sources**
- [maven/maven.go](file://routers/api/packages/maven/maven.go)
- [maven/api.go](file://routers/api/packages/maven/api.go)

### Generic
The Generic package registry provides a simple key-value store for arbitrary files, allowing users to store any type of file as a package.

**Endpoints:**
- `GET /api/packages/{owner}/generic/{package}/{version}/{filename}` - Download package file
- `PUT /api/packages/{owner}/generic/{package}/{version}/{filename}` - Upload package file
- `DELETE /api/packages/{owner}/generic/{package}/{version}` - Delete package
- `DELETE /api/packages/{owner}/generic/{package}/{version}/{filename}` - Delete package file

Generic packages are identified by name, version, and filename. This registry type is useful for storing build artifacts, configuration files, or any other binary files that don't fit into the other package types. Each package version can contain multiple files.

**Section sources**
- [generic/generic.go](file://routers/api/packages/generic/generic.go)

## Package Metadata Structure
The structure of package metadata varies by package type, reflecting the specific requirements and conventions of each package manager.

For npm packages, metadata includes the package name, version, description, author, license, dependencies, and repository information. The metadata is stored in the package.json format and includes additional fields like homepage, keywords, and bin scripts.

NuGet package metadata contains comprehensive information including title, authors, description, summary, project URL, license URL, icon URL, release notes, and dependency groups organized by target framework. The metadata also includes publication details like creation date and version-specific properties.

Container/OCI images store metadata in the image manifest and configuration. The manifest contains information about layers, platform (OS, architecture), and media types. The configuration includes environment variables, command to run, exposed ports, and labels containing additional metadata like maintainer, version, and description.

Maven package metadata is defined in the POM (Project Object Model) file, which includes group ID, artifact ID, version, packaging type, dependencies, properties, and build configuration. The registry extracts relevant metadata from the POM file for internal use and display.

Generic packages have minimal metadata, consisting primarily of the package name, version, and filename. Additional metadata can be stored as properties associated with the package version.

**Section sources**
- [descriptor.go](file://models/packages/descriptor.go)
- [package.go](file://models/packages/package.go)
- [package_version.go](file://models/packages/package_version.go)

## Versioning Schemes
Gitea's Package Registry supports different versioning schemes depending on the package type.

npm packages follow Semantic Versioning (SemVer) 2.0.0, which uses a three-part version number (MAJOR.MINOR.PATCH) with optional pre-release and build metadata. The registry supports version ranges and tags (like "latest" or "beta") for flexible package resolution.

NuGet packages use a four-part version number (Major.Minor.Build.Revision) with optional pre-release suffixes. The registry supports both stable and pre-release versions, with pre-release versions identified by suffixes like "-alpha", "-beta", or "-rc".

Container/OCI images use tags for versioning, with the special "latest" tag commonly used for the most recent version. Images can also be referenced by their content digest (SHA256 hash), providing immutable references. The registry supports multiple tags pointing to the same image digest.

Maven packages use a flexible versioning scheme that can include numbers, letters, and hyphens. Common patterns include numeric versions (1.0.0), snapshot versions (1.0.0-SNAPSHOT), and release candidates (1.0.0-RC1). The registry handles snapshot versions specially, allowing overwriting of the same version.

Generic packages use a simple string for versioning, allowing users to define their own versioning scheme. This could be semantic versions, dates, commit hashes, or any other identifier that makes sense for the specific use case.

**Section sources**
- [package_version.go](file://models/packages/package_version.go)
- [package.go](file://models/packages/package.go)

## Dependency Resolution
Dependency resolution mechanisms vary by package manager, with each implementing its own algorithm for resolving package dependencies.

npm uses a flat dependency tree by default, installing dependencies at the top level when possible to avoid duplication. It supports both direct dependencies and various types of dependencies (devDependencies, peerDependencies, optionalDependencies). The registry provides the full dependency tree in the package metadata, allowing clients to resolve transitive dependencies.

NuGet implements a nearest-wins version resolution strategy, where the version of a dependency is determined by its position in the dependency graph. Dependencies are organized by target framework, allowing different dependencies for different frameworks. The registry provides dependency groups in the package metadata, with each group containing dependencies for a specific target framework.

Maven uses a depth-first search algorithm to resolve dependencies, with the first declaration of a dependency taking precedence. It supports dependency scopes (compile, provided, runtime, test, system) that affect when dependencies are available. The registry serves maven-metadata.xml files that contain information about available versions, helping clients resolve version ranges.

Container/OCI images have a layered dependency model, where each layer depends on the previous layer. The registry stores each layer as a separate blob, allowing efficient sharing of common layers between images. Image indexes can reference multiple manifests, enabling multi-platform images.

Generic packages do not have a built-in dependency system, as they are designed for arbitrary files without dependency relationships.

**Section sources**
- [package_file.go](file://models/packages/package_file.go)
- [package_version.go](file://models/packages/package_version.go)

## Common Issues
Several common issues may arise when working with the Package Registry, typically related to authentication, metadata, or storage.

Authentication failures are a frequent issue, often caused by using an invalid or expired token, insufficient permissions, or incorrect authentication method. For Container/OCI registry, ensure the client is properly requesting and using tokens from the `/v2/token` endpoint.

Incorrect package metadata can prevent package consumption or display. For npm packages, ensure the package.json file contains valid JSON with required fields. For NuGet packages, validate that the .nuspec file contains all required metadata. For Maven packages, ensure the POM file is well-formed XML.

Storage quota limitations can prevent package uploads. Users and organizations may have limits on the total number of packages, total storage size, or storage size per package type. Monitor usage and request quota increases if needed.

For Container/OCI images, manifest validation errors can occur if the manifest JSON is malformed or contains unsupported media types. Ensure manifests comply with the OCI Image Format Specification.

Package name conflicts can occur when trying to publish a package with a name that already exists. The registry enforces uniqueness of package names within an owner's namespace.

**Section sources**
- [auth.go](file://services/packages/auth.go)
- [packages.go](file://services/packages/packages.go)
- [package.go](file://models/packages/package.go)

## Performance Considerations
Several performance considerations should be taken into account when using the Package Registry, particularly for large packages or high-volume operations.

For large package uploads, consider using the Container/OCI registry's chunked upload capability, which allows uploading large layers in smaller chunks. This improves reliability and allows resuming interrupted uploads.

Implement client-side caching to reduce repeated downloads of the same packages. Most package managers have built-in caching mechanisms that should be configured appropriately.

For high-volume operations, consider rate limiting to prevent overwhelming the server. The registry may implement rate limiting to protect against abuse.

CDN integration can significantly improve download performance for geographically distributed users. Configure a CDN in front of the Gitea instance to cache package downloads and reduce server load.

Monitor storage usage and implement package cleanup policies to remove old or unused packages. This helps maintain performance and stay within storage quotas.

For metadata-heavy operations like package searches, ensure the database is properly indexed and consider the impact on server resources.

**Section sources**
- [packages.go](file://services/packages/packages.go)
- [package_blob_upload.go](file://models/packages/package_blob_upload.go)

## Client Examples

### npm Client
```bash
# Publish a package
npm publish --registry http://gitea.example.com/api/packages/username/npm

# Install a package
npm install package-name --registry http://gitea.example.com/api/packages/username/npm

# Set registry in .npmrc
echo "@scope:http://gitea.example.com/api/packages/username/npm" > .npmrc
```

### NuGet Client
```bash
# Add package source
nuget sources Add -Name "Gitea" -Source "http://gitea.example.com/api/packages/username/nuget/index.json"

# Push a package
nuget push package.nupkg -Source "Gitea" -ApiKey "your-api-token"

# Install a package
nuget install package-name -Source "Gitea"
```

### Docker Client
```bash
# Login to registry
docker login gitea.example.com

# Tag and push an image
docker tag image-name gitea.example.com/username/repository:tag
docker push gitea.example.com/username/repository:tag

# Pull an image
docker pull gitea.example.com/username/repository:tag
```

### Maven Client
```xml
<!-- In pom.xml -->
<repositories>
  <repository>
    <id>gitea</id>
    <url>http://gitea.example.com/api/packages/username/maven</url>
  </repository>
</repositories>

<distributionManagement>
  <repository>
    <id>gitea</id>
    <url>http://gitea.example.com/api/packages/username/maven</url>
  </repository>
</distributionManagement>
```

```bash
# Deploy artifact
mvn deploy
```

### Generic Package Client
```bash
# Upload a file
curl -X PUT -H "Authorization: token your-api-token" \
  -T file.bin \
  http://gitea.example.com/api/packages/username/generic/package-name/1.0.0/file.bin

# Download a file
curl -H "Authorization: token your-api-token" \
  http://gitea.example.com/api/packages/username/generic/package-name/1.0.0/file.bin \
  -o file.bin
```

**Section sources**
- [api.go](file://routers/api/packages/api.go)
- [npm.go](file://routers/api/packages/npm/npm.go)
- [nuget/nuget.go](file://routers/api/packages/nuget/nuget.go)
- [container/manifest.go](file://routers/api/packages/container/manifest.go)
- [maven/maven.go](file://routers/api/packages/maven/maven.go)
- [generic/generic.go](file://routers/api/packages/generic/generic.go)