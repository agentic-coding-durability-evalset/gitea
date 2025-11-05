# Repository Management

<cite>
**Referenced Files in This Document**   
- [repo.go](file://models/repo/repo.go)
- [fork.go](file://models/repo/fork.go)
- [transfer.go](file://models/repo/transfer.go)
- [create.go](file://services/repository/create.go)
- [fork.go](file://services/repository/fork.go)
- [transfer.go](file://services/repository/transfer.go)
- [delete.go](file://services/repository/delete.go)
- [archiver.go](file://services/repository/archiver/archiver.go)
- [protected_branch.go](file://models/git/protected_branch.go)
- [branch.go](file://services/repository/branch.go)
</cite>

## Table of Contents
1. [Introduction](#introduction)
2. [Repository Models](#repository-models)
3. [Repository Services](#repository-services)
4. [Repository Operations](#repository-operations)
5. [Branch Protection Rules](#branch-protection-rules)
6. [Repository Mirroring](#repository-mirroring)
7. [Access Control Configuration](#access-control-configuration)
8. [Relationships Between Repositories, Users, and Organizations](#relationships-between-repositories-users-and-organizations)
9. [Common Issues](#common-issues)
10. [Performance Considerations](#performance-considerations)
11. [Best Practices for Repository Organization](#best-practices-for-repository-organization)

## Introduction
Gitea is a self-hosted Git service that provides a web-based interface for managing Git repositories. It offers a comprehensive set of features for repository management, including creation, forking, cloning, transferring, and archiving. This document provides a detailed overview of Gitea's repository management capabilities, focusing on the implementation details of repository models and services, repository operations, branch protection rules, repository mirroring, access control configuration, and the relationships between repositories, users, and organizations. It also addresses common issues such as repository corruption and push/pull failures, and provides performance considerations for large repositories and best practices for repository organization.

## Repository Models
The repository model in Gitea is defined in the `models/repo/repo.go` file. The `Repository` struct represents a Git repository and contains various fields such as `ID`, `OwnerID`, `OwnerName`, `LowerName`, `Name`, `Description`, `Website`, `OriginalServiceType`, `OriginalURL`, `DefaultBranch`, `DefaultWikiBranch`, `NumWatches`, `NumStars`, `NumForks`, `NumIssues`, `NumClosedIssues`, `NumOpenIssues`, `NumPulls`, `NumClosedPulls`, `NumOpenPulls`, `NumMilestones`, `NumClosedMilestones`, `NumOpenMilestones`, `NumProjects`, `NumClosedProjects`, `NumOpenProjects`, `NumActionRuns`, `NumClosedActionRuns`, `NumOpenActionRuns`, `IsPrivate`, `IsEmpty`, `IsArchived`, `IsMirror`, `Status`, `Units`, `PrimaryLanguage`, `IsFork`, `ForkID`, `BaseRepo`, `IsTemplate`, `TemplateID`, `Size`, `GitSize`, `LFSSize`, `CodeIndexerStatus`, `StatsIndexerStatus`, `IsFsckEnabled`, `CloseIssuesViaCommitInAnyBranch`, `Topics`, `ObjectFormatName`, `TrustModel`, `Avatar`, `CreatedUnix`, `UpdatedUnix`, and `ArchivedUnix`. The `Repository` struct also includes methods for loading attributes, checking if a repository is being migrated, checking if a repository is broken, marking a repository as broken and empty, and loading the owner of the repository.

**Section sources**
- [repo.go](file://models/repo/repo.go#L1-L994)

## Repository Services
The repository services in Gitea are defined in the `services/repository` directory. These services include functions for creating, forking, transferring, and deleting repositories, as well as for managing repository branches, tags, and other repository-related operations. The services are implemented in various files such as `create.go`, `fork.go`, `transfer.go`, `delete.go`, and `archiver.go`. These services interact with the repository models and provide a higher-level interface for managing repositories.

**Section sources**
- [create.go](file://services/repository/create.go#L1-L484)
- [fork.go](file://services/repository/fork.go#L1-L256)
- [transfer.go](file://services/repository/transfer.go#L1-L549)
- [delete.go](file://services/repository/delete.go#L1-L404)
- [archiver.go](file://services/repository/archiver/archiver.go#L1-L391)

## Repository Operations
Gitea supports a wide range of repository operations, including creation, forking, cloning, transferring, and archiving. These operations are implemented in the repository services and provide a comprehensive set of tools for managing repositories.

### Creation
The `CreateRepository` function in `services/repository/create.go` is used to create a new repository. It takes a `CreateRepoOptions` struct as input, which contains various options such as the repository name, description, original URL, Git service type, Gitignores, issue labels, license, readme, default branch, privacy, mirroring, templating, auto-initialization, status, trust model, mirror interval, and object format name. The function first checks if the user can create a repository in the specified owner, then creates the repository in the database, initializes the Git repository, and updates the repository status to ready.

**Section sources**
- [create.go](file://services/repository/create.go#L1-L484)

### Forking
The `ForkRepository` function in `services/repository/fork.go` is used to fork a repository. It takes a `ForkRepoOptions` struct as input, which contains the base repository, name, description, and single branch. The function first checks if the user can fork the repository, then creates the forked repository in the database, clones the repository, and updates the repository status to ready.

**Section sources**
- [fork.go](file://services/repository/fork.go#L1-L256)

### Transferring
The `StartRepositoryTransfer` function in `services/repository/transfer.go` is used to transfer a repository from one owner to another. It takes a `doer`, `newOwner`, `repo`, and `teams` as input. The function first checks if the repository is ready for transfer, then creates a pending repository transfer, and notifies the users who can accept or reject the transfer.

**Section sources**
- [transfer.go](file://services/repository/transfer.go#L1-L549)

### Archiving
The `StartArchive` function in `services/repository/archiver/archiver.go` is used to archive a repository. It takes an `ArchiveRequest` struct as input, which contains the repository, type, commit ID, and archive reference short name. The function first checks if the archive already exists, then creates the archive and updates the repository status to ready.

**Section sources**
- [archiver.go](file://services/repository/archiver/archiver.go#L1-L391)

## Branch Protection Rules
Gitea provides branch protection rules to ensure that certain branches are protected from unauthorized changes. These rules are defined in the `models/git/protected_branch.go` file and include options such as `CanPush`, `EnableWhitelist`, `WhitelistUserIDs`, `WhitelistTeamIDs`, `EnableMergeWhitelist`, `WhitelistDeployKeys`, `MergeWhitelistUserIDs`, `MergeWhitelistTeamIDs`, `CanForcePush`, `EnableForcePushAllowlist`, `ForcePushAllowlistUserIDs`, `ForcePushAllowlistTeamIDs`, `ForcePushAllowlistDeployKeys`, `EnableStatusCheck`, `StatusCheckContexts`, `EnableApprovalsWhitelist`, `ApprovalsWhitelistUserIDs`, `ApprovalsWhitelistTeamIDs`, `RequiredApprovals`, `BlockOnRejectedReviews`, `BlockOnOfficialReviewRequests`, `BlockOnOutdatedBranch`, `DismissStaleApprovals`, `IgnoreStaleApprovals`, `RequireSignedCommits`, `ProtectedFilePatterns`, `UnprotectedFilePatterns`, and `BlockAdminMergeOverride`. These rules can be configured to protect specific branches or all branches in a repository.

**Section sources**
- [protected_branch.go](file://models/git/protected_branch.go#L1-L609)

## Repository Mirroring
Gitea supports repository mirroring, which allows a repository to be automatically synchronized with a remote repository. This feature is useful for keeping a local copy of a remote repository up to date. The mirroring functionality is implemented in the `services/repository/mirror.go` file and includes options such as the mirror interval, which determines how often the repository is synchronized with the remote repository.

**Section sources**
- [mirror.go](file://services/repository/mirror.go#L1-L100)

## Access Control Configuration
Gitea provides access control configuration to manage who can access and modify repositories. This is achieved through the use of teams, organizations, and user permissions. The access control configuration is defined in the `models/perm/access` directory and includes options such as `AccessMode`, `Permission`, and `HasAccessUnit`. These options can be used to grant or revoke access to repositories based on user roles and team memberships.

**Section sources**
- [access.go](file://models/perm/access/access.go#L1-L100)

## Relationships Between Repositories, Users, and Organizations
In Gitea, repositories are owned by users or organizations. Users can create repositories, fork repositories, and transfer repositories to other users or organizations. Organizations can have multiple teams, and each team can have different levels of access to repositories. The relationships between repositories, users, and organizations are managed through the `models/repo/repo.go` and `models/user/user.go` files, which define the `Repository` and `User` structs, respectively. These structs include fields such as `OwnerID`, `OwnerName`, `Owner`, `OwnerID`, `OwnerName`, `Owner`, `OwnerID`, `OwnerName`, `Owner`, and `OwnerID`, `OwnerName`, `Owner`.

**Section sources**
- [repo.go](file://models/repo/repo.go#L1-L994)
- [user.go](file://models/user/user.go#L1-L100)

## Common Issues
Gitea may encounter common issues such as repository corruption and push/pull failures. Repository corruption can occur due to various reasons such as disk errors, network issues, or software bugs. Push/pull failures can occur due to network issues, authentication problems, or repository configuration issues. To address these issues, Gitea provides tools for repository maintenance, such as the `git fsck` command, which can be used to check the integrity of a repository, and the `git gc` command, which can be used to clean up and optimize a repository.

**Section sources**
- [repo.go](file://models/repo/repo.go#L1-L994)
- [git.go](file://modules/git/git.go#L1-L100)

## Performance Considerations
For large repositories, performance considerations are important to ensure that operations such as cloning, pushing, and pulling are efficient. Gitea provides several performance optimizations, such as the use of shallow clones, which can reduce the amount of data transferred during cloning, and the use of incremental updates, which can reduce the amount of data transferred during pushing and pulling. Additionally, Gitea supports the use of Git LFS (Large File Storage) to manage large files, which can improve performance by reducing the size of the repository.

**Section sources**
- [repo.go](file://models/repo/repo.go#L1-L994)
- [git.go](file://modules/git/git.go#L1-L100)

## Best Practices for Repository Organization
To ensure that repositories are well-organized and easy to manage, it is recommended to follow best practices such as using descriptive repository names, organizing repositories into logical groups, and using branches and tags to manage different versions of the code. Additionally, it is recommended to use branch protection rules to protect important branches, and to use access control configuration to manage who can access and modify repositories. Finally, it is recommended to use repository mirroring to keep local copies of remote repositories up to date, and to use performance optimizations to ensure that operations are efficient.

**Section sources**
- [repo.go](file://models/repo/repo.go#L1-L994)
- [branch.go](file://services/repository/branch.go#L1-L799)
- [access.go](file://models/perm/access/access.go#L1-L100)
- [mirror.go](file://services/repository/mirror.go#L1-L100)